# P0 Bug Fixes + Phased Build Plan

---

## Part 1: P0 Bug Fixes

### Fix 1: Governed use_agent counter — request-scoped via invocation_state

**Bug**: Counter lives in closure, shared across all requests, never resets. After `max_sub_agents` total invocations globally, all future requests fail permanently.

**Root cause**: Closure-scoped mutable state in a long-lived tool function.

**Fix**: Use a `contextvars.ContextVar` to scope the counter per async task (which maps 1:1 to HTTP requests in FastAPI/Uvicorn). Each request gets its own counter that starts at 0 and dies with the request.

```python
# strands_platform/control_plane/compilation/governed_tools.py

from strands import Agent, tool
from typing import AsyncIterator
from contextvars import ContextVar
from .tool_wrapper import SubAgentResult


# Context variable: automatically scoped per-asyncio-task (per-request in FastAPI)
_sub_agent_counter: ContextVar[int] = ContextVar("_sub_agent_counter", default=0)


def create_governed_use_agent(
    allowed_tool_names: list[str],
    allowed_models: list[str],
    max_sub_agents: int,
    parent_hooks: list,
    default_model,
    tool_resolver,
) -> callable:
    """Governed use_agent: LLM creates sub-agents within boundaries.

    Counter is request-scoped via ContextVar. Each HTTP request (asyncio task)
    gets its own counter starting at 0. No cross-request contamination.
    """

    @tool
    async def use_agent(
        system_prompt: str,
        task: str,
        tools: list[str] | None = None,
        model_id: str | None = None,
    ) -> AsyncIterator:
        """Create a specialized sub-agent for a specific task.

        Args:
            system_prompt: What the sub-agent should do.
            task: The task to complete.
            tools: Tool names (must be in allowed list).
            model_id: Model to use (must be in allowed list).
        """
        # Increment request-scoped counter
        current = _sub_agent_counter.get()
        current += 1
        _sub_agent_counter.set(current)

        if current > max_sub_agents:
            yield f"Error: Max sub-agents ({max_sub_agents}) per request exceeded."
            return

        # Validate tools
        resolved = []
        for name in (tools or []):
            if name not in allowed_tool_names:
                yield f"Error: Tool '{name}' not allowed. Available: {allowed_tool_names}"
                return
            resolved.append(tool_resolver.resolve(name))

        # Validate model
        if model_id and model_id not in allowed_models:
            yield f"Error: Model '{model_id}' not allowed. Available: {allowed_models}"
            return

        agent = Agent(
            system_prompt=system_prompt,
            model=default_model,
            tools=resolved,
            hooks=list(parent_hooks),
            callback_handler=None,
        )
        result = None
        async for event in agent.stream_async(task):
            yield SubAgentResult(agent=agent, event=event)
            if "result" in event:
                result = event["result"]
        yield str(result)

    return use_agent


def create_governed_swarm(
    max_size: int,
    allowed_patterns: list[str],
) -> callable:
    @tool
    def governed_swarm(
        task: str,
        swarm_size: int = 3,
        coordination_pattern: str = "collaborative",
    ) -> str:
        """Create a collaborative swarm of agents for a task."""
        if swarm_size > max_size:
            return f"Error: Swarm size {swarm_size} exceeds max ({max_size})."
        if coordination_pattern not in allowed_patterns:
            return f"Error: Pattern '{coordination_pattern}' not allowed."
        from strands_tools import swarm
        return swarm(task=task, swarm_size=swarm_size,
                     coordination_pattern=coordination_pattern)
    return governed_swarm
```

**Test**: Invoke governed `use_agent` twice across two sequential requests. First request creates `max_sub_agents` sub-agents successfully. Second request also creates `max_sub_agents` sub-agents successfully (counter reset). If the counter were closure-scoped, the second request would fail.

---

### Fix 2: Session wiring — engine passes session_id to agent via session manager

**Bug**: `session_id` accepted by API but discarded in engine. No conversation history persists. Multi-turn is broken.

**Root cause**: The hydrator creates a bare Agent with no session manager attached, and the engine doesn't pass session context.

**Fix**: The hydrator attaches a session manager to the Agent. The engine passes `session_id` when invoking the agent so Strands loads/saves conversation history.

```python
# strands_platform/execution_plane/session.py

from typing import Any


def create_session_manager(config: dict) -> Any:
    """Factory for Strands session managers.

    Returns None for in-memory (no persistence needed — Strands
    maintains conversation state within the Agent instance).
    Returns a configured session manager for DynamoDB/S3/AgentCore.
    """
    manager_type = config.get("session_manager", "memory")

    if manager_type == "memory":
        return None  # Strands Agent handles in-memory by default

    elif manager_type == "dynamodb":
        from strands.session.dynamodb import DynamoDBSessionManager
        return DynamoDBSessionManager(
            table_name=config["table_name"],
            ttl_hours=config.get("ttl_hours", 24),
        )

    elif manager_type == "s3":
        from strands.session.s3 import S3SessionManager
        return S3SessionManager(
            bucket=config["bucket"],
            prefix=config.get("prefix", "sessions/"),
        )

    elif manager_type == "agentcore":
        from strands.session.agentcore import AgentCoreSessionManager
        return AgentCoreSessionManager()

    else:
        raise ValueError(f"Unknown session manager: {manager_type}")
```

```python
# strands_platform/execution_plane/hydrator.py  (FIXED)

from strands import Agent
from strands.models.bedrock import BedrockModel
from .session import create_session_manager
from typing import Any


class Hydrator:
    """Turns an agent config dict into a live Strands Agent.

    FIXED: Now attaches session manager for conversation persistence.
    Config dict contains serializable primitives only (model params,
    tool names, hook names). Objects are rebuilt here.
    """

    def __init__(self, tool_resolver, hook_resolver):
        self._tools = tool_resolver
        self._hooks = hook_resolver
        # Cache model providers — stateless, safe to reuse across requests
        self._model_cache: dict[str, Any] = {}

    def hydrate(self, config: dict, session_id: str | None = None) -> Agent:
        """Hydrate a config dict into a live Strands Agent.

        Args:
            config: Serializable agent config dict.
            session_id: Optional session ID for conversation persistence.
                       If provided and a session manager is configured,
                       Strands will load existing conversation history.
        """
        model = self._build_model(config.get("model", {}))
        tools = [self._tools.resolve(t) for t in config.get("tools", [])]
        hooks = [self._hooks.resolve(h) for h in config.get("hooks", [])]
        session_manager = create_session_manager(config.get("memory", {}))

        agent = Agent(
            name=config.get("name", "agent"),
            system_prompt=config.get("system_prompt", ""),
            model=model,
            tools=tools,
            hooks=hooks,
            session_manager=session_manager,
        )

        # If session_id is provided, Strands loads conversation history
        # from the session manager before the first model call.
        # The session_id is passed at invocation time, not construction time.

        return agent, session_id

    def _build_model(self, model_config: dict) -> Any:
        """Build model provider, cached by config hash."""
        cache_key = f"{model_config.get('provider')}:{model_config.get('model_id')}:{model_config.get('region')}"
        if cache_key in self._model_cache:
            return self._model_cache[cache_key]

        provider = model_config.get("provider", "bedrock")
        if provider == "bedrock":
            model = BedrockModel(
                model_id=model_config.get("model_id",
                    "us.anthropic.claude-sonnet-4-20250514-v1:0"),
                region_name=model_config.get("region", "us-west-2"),
            )
        elif provider == "anthropic":
            from strands.models.anthropic import AnthropicModel
            model = AnthropicModel(model=model_config.get("model_id"))
        elif provider == "ollama":
            from strands.models.ollama import OllamaModel
            model = OllamaModel(model=model_config.get("model_id"))
        else:
            from strands.models.litellm import LiteLLMModel
            model = LiteLLMModel(
                model=f"{provider}/{model_config.get('model_id', '')}")

        self._model_cache[cache_key] = model
        return model
```

```python
# strands_platform/execution_plane/engine.py  (FIXED)

from strands import Agent
from ..control_plane.registry.models import RegistryBackend
from .hydrator import Hydrator
from typing import Any
import time
import asyncio


# Request timeout — prevents runaway requests from holding resources
DEFAULT_REQUEST_TIMEOUT_SECONDS = 300


class AgentEngine:
    """The shared runtime engine.

    FIXED:
    - session_id flows through to Agent invocation
    - Request-level timeout on all invocations
    - Conversation history loaded via session manager
    """

    def __init__(self, registry: RegistryBackend, hydrator: Hydrator,
                 cache_ttl_seconds: int = 60,
                 request_timeout_seconds: int = DEFAULT_REQUEST_TIMEOUT_SECONDS):
        self._registry = registry
        self._hydrator = hydrator
        self._cache_ttl = cache_ttl_seconds
        self._request_timeout = request_timeout_seconds
        self._config_cache: dict[str, tuple[float, dict]] = {}

    def get_agent_config(self, agent_id: str, version_id: str | None = None) -> dict:
        """Get agent config from registry. Cached with TTL."""
        cache_key = f"{agent_id}:{version_id or 'LATEST'}"
        now = time.time()

        if cache_key in self._config_cache:
            cached_time, config = self._config_cache[cache_key]
            if now - cached_time < self._cache_ttl:
                return config

        config = self._registry.get_agent_config(agent_id, version_id)
        self._config_cache[cache_key] = (now, config)
        return config

    def invoke(self, agent_id: str, message: str,
               session_id: str | None = None) -> str:
        """Synchronous invoke with session support and timeout."""
        config = self.get_agent_config(agent_id)
        agent, sid = self._hydrator.hydrate(config, session_id)

        # Strands Agent invocation — session_id enables conversation history
        if sid:
            result = agent(message, session_id=sid)
        else:
            result = agent(message)
        return str(result)

    async def invoke_stream(self, agent_id: str, message: str,
                            session_id: str | None = None):
        """Async streaming invoke with session support and timeout.

        Yields Strands events. Request-level timeout prevents runaway execution.
        """
        config = self.get_agent_config(agent_id)
        agent, sid = self._hydrator.hydrate(config, session_id)

        try:
            kwargs = {"session_id": sid} if sid else {}
            async with asyncio.timeout(self._request_timeout):
                async for event in agent.stream_async(message, **kwargs):
                    yield event
        except asyncio.TimeoutError:
            yield {
                "force_stop": True,
                "force_stop_reason": f"Request timeout ({self._request_timeout}s) exceeded.",
            }
```

**Test**: Send two messages to the same agent with the same `session_id`. The second message's response should reference context from the first. Without this fix, the second message would be processed in isolation.

---

### Fix 3: Server uses wildcard routes — supports dynamic agents

**Bug**: Routes pre-mounted per-agent at startup. Store-based registry can't serve dynamically-added agents. Also prevents route explosion at scale.

**Fix**: Single set of wildcard routes that resolve agent_id from the path parameter at request time.

```python
# strands_platform/execution_plane/server.py  (FIXED)

from fastapi import FastAPI, Request, HTTPException, Path
from fastapi.responses import StreamingResponse, JSONResponse
from .engine import AgentEngine
from .streaming import format_sse_event
import json
import asyncio
import logging

logger = logging.getLogger("strands_platform.server")


def create_app(engine: AgentEngine, platform_ir=None) -> FastAPI:
    """Create FastAPI app with wildcard routes.

    Routes resolve agent_id from path at request time, not at startup.
    Works with both file-based and store-based registries.
    Agents added to the registry after startup are immediately available.
    """
    app = FastAPI(title="Strands Agent Platform")

    # ── Registry ──────────────────────────────────────────────

    @app.get("/registry/agents")
    def list_agents():
        return engine._registry.list_agents()

    # ── Agent endpoints (wildcard) ────────────────────────────

    @app.post("/agents/{agent_id}")
    async def invoke_agent(request: Request, agent_id: str = Path(...)):
        _check_agent_accessible(agent_id, engine, platform_ir)
        body = await request.json()
        result = engine.invoke(agent_id, body["message"], body.get("session_id"))
        return {"response": result, "agent": agent_id}

    @app.post("/agents/{agent_id}/stream")
    async def stream_agent(request: Request, agent_id: str = Path(...)):
        _check_agent_accessible(agent_id, engine, platform_ir)
        body = await request.json()

        async def event_generator():
            async for event in engine.invoke_stream(
                agent_id, body["message"], body.get("session_id")
            ):
                sse_data = format_sse_event(event)
                if sse_data:
                    yield sse_data

        return StreamingResponse(event_generator(), media_type="text/event-stream")

    @app.get("/agents/{agent_id}/ui-config")
    def agent_ui_config(agent_id: str = Path(...)):
        """UI shell metadata. Frontend reads this to self-configure."""
        config = engine.get_agent_config(agent_id)
        return {
            "agent_id": agent_id,
            "ui_mode": config.get("ui_template", {}).get("ui_mode", "chat"),
            "ui_features": config.get("ui_template", {}).get("ui_features", {}),
            "exposure_mode": config.get("exposure_mode", "api"),
        }

    @app.post("/agents/{agent_id}/v/{version_id}")
    async def invoke_agent_versioned(request: Request, agent_id: str = Path(...),
                                     version_id: str = Path(...)):
        body = await request.json()
        config = engine.get_agent_config(agent_id, version_id)
        agent, sid = engine._hydrator.hydrate(config, body.get("session_id"))
        if sid:
            result = agent(body["message"], session_id=sid)
        else:
            result = agent(body["message"])
        return {"response": str(result), "agent": agent_id, "version": version_id}

    @app.post("/agents/{agent_id}/trigger")
    async def trigger_agent(request: Request, agent_id: str = Path(...)):
        """Fire-and-forget for background agents."""
        _check_agent_accessible(agent_id, engine, platform_ir, require_background=True)
        body = await request.json()
        asyncio.create_task(_run_background(agent_id, body, engine))
        return {"status": "triggered", "agent": agent_id}

    # ── Workflow endpoints (wildcard) ─────────────────────────

    @app.post("/workflows/{workflow_id}")
    async def invoke_workflow(request: Request, workflow_id: str = Path(...)):
        body = await request.json()
        workflow = engine.get_workflow(workflow_id)
        result = workflow(body["message"])
        return {"response": str(result), "workflow": workflow_id}

    @app.post("/workflows/{workflow_id}/stream")
    async def stream_workflow(request: Request, workflow_id: str = Path(...)):
        body = await request.json()
        workflow = engine.get_workflow(workflow_id)

        async def event_generator():
            async for event in workflow.stream_async(body["message"]):
                sse_data = format_sse_event(event)
                if sse_data:
                    yield sse_data

        return StreamingResponse(event_generator(), media_type="text/event-stream")

    return app


def _check_agent_accessible(agent_id: str, engine: AgentEngine,
                             platform_ir=None, require_background: bool = False):
    """Validate agent exists and isn't subagent_only.

    For file-based registry: check exposure_mode from IR.
    For store-based registry: check from loaded config.
    """
    try:
        config = engine.get_agent_config(agent_id)
    except KeyError:
        raise HTTPException(404, f"Agent '{agent_id}' not found.")

    exposure = config.get("exposure_mode", "api")
    if exposure == "subagent_only":
        raise HTTPException(403, f"Agent '{agent_id}' is subagent_only — not directly accessible.")
    if require_background and exposure != "background":
        raise HTTPException(400, f"Agent '{agent_id}' is not a background agent. Use /agents/{agent_id} instead.")


async def _run_background(agent_id: str, body: dict, engine: AgentEngine):
    try:
        engine.invoke(agent_id, body.get("message", ""), body.get("session_id"))
    except Exception as exc:
        logger.error(f"Background agent '{agent_id}' failed: {exc}", exc_info=True)
```

**Test**: Start server with file-based registry containing 2 agents. Both accessible. Add a third agent to the registry (store-based) without restarting. Third agent accessible immediately via wildcard route.

---

## Part 2: Phased Build Plan

### Milestone Structure

Three milestones, not eight phases. Each milestone produces a usable system. Each task has a verification checkpoint — the specific test or behavior that proves it works.

---

### Milestone 1: USABLE ENGINE
*Goal: One person can define agents in JSON and talk to them via CLI and API with streaming.*
*Duration estimate: 3-4 weeks*

```
TASK 1.01  Project scaffold + CI
  Create package structure (control_plane/, execution_plane/, cli/)
  Set up pyproject.toml, pytest, ruff, mypy, GitHub Actions
  ✅ CHECK: `pip install -e .` works, `pytest` runs with 0 tests passing

TASK 1.02  IR dataclasses
  Implement all frozen dataclasses in ir.py
  AgentIR, ToolRef, ModelConfig, WorkflowIR, PlatformIR
  ✅ CHECK: Create an AgentIR instance, verify it's frozen (mutation raises)

TASK 1.03  JSON parser
  Parse agents.json → PlatformIR
  Handle defaults block, tool refs, ${VAR} env interpolation
  ✅ CHECK: Parse the example agents.json from user-interaction-flows doc,
            produce correct PlatformIR with 4 agents

TASK 1.04  Schema validator
  JSON Schema for v1.0
  Validate before parsing proceeds
  ✅ CHECK: Invalid config (missing agent name) → SchemaValidationError
            with field path and expected type

TASK 1.05  Tool registry + resolver
  Register built-in strands_tools by name
  Register custom @tool functions
  Resolve ToolRef → concrete tool object
  Detect conflicts (same name in builtin + custom)
  ✅ CHECK: Resolve "calculator" → strands_tools.calculator
            Resolve "nonexistent" → ResolutionError with alternatives list
            Register custom "my_tool" + builtin "my_tool" → ConflictError

TASK 1.06  Hook registry
  Register HookProvider classes by name
  Built-in "logging" hook auto-registered
  ✅ CHECK: Resolve "logging" → LoggingHook instance
            Resolve "unknown" → ResolutionError

TASK 1.07  Dependency graph + cycle detection + topo sort
  Build dep graph from PlatformIR
  Kahn's algorithm with alphabetical tie-breaking
  Cycle detection with path reporting
  ✅ CHECK: 4-agent hierarchy → correct compilation order (leaves first)
            Circular A→B→A → CircularDependencyError with "A → B → A"

TASK 1.08  Tool wrapper generation (async streaming + sync)
  create_streaming_agent_tool: @tool async def with stream_async + SubAgentResult
  create_sync_agent_tool: @tool def with synchronous invocation
  Per-invocation Agent() construction (NOT closure over pre-built Agent)
  Description derivation from agent description/system_prompt
  ✅ CHECK: Generated streaming wrapper yields SubAgentResult events
            Generated sync wrapper returns string
            Two invocations produce independent Agent instances (different id())

TASK 1.09  Agent compiler
  AgentIR → agent_config dict (serializable, model params not objects)
  Resolve tools, attach hooks, build system prompt
  Generate @tool wrappers for agent-as-tool refs
  ✅ CHECK: Compile 4-agent hierarchy → 4 config dicts
            support_lead config has 3 tool wrappers (classifier, billing, technical)
            Each wrapper's docstring matches sub-agent description

TASK 1.10  Compiler orchestrator
  Wire parsing → schema validation → resolution → dep graph → topo sort → agent compilation
  Produce Platform object with .agents dict
  Collect GovernanceWarnings
  ✅ CHECK: compile_config("config/") → Platform with 4 agents
            platform.agents["support_lead"] is a valid Strands Agent
            platform.warnings is a list (may be empty)

TASK 1.11  Hydrator
  Config dict → live Strands Agent with session manager
  Model provider cache (stateless, safe to reuse)
  Session manager factory (memory, dynamodb, s3, agentcore)
  ✅ CHECK: Hydrate config dict with session_manager="memory" → Agent with no session manager
            Hydrate with session_manager="dynamodb" → Agent with DynamoDBSessionManager
            Two hydrations with same model config → same model object (cached)

TASK 1.12  Engine
  Load config from registry, hydrate, invoke with session_id
  Config cache with TTL
  Request-level timeout (asyncio.timeout)
  Streaming invocation via stream_async
  ✅ CHECK: invoke("support_lead", "hello", session_id="s1") → string result
            invoke_stream yields events with "data" keys
            Timeout fires after configured seconds → force_stop event

TASK 1.13  Server (wildcard routes + SSE)
  FastAPI with wildcard /agents/{agent_id} routes
  POST /agents/{id} → request-response
  POST /agents/{id}/stream → SSE streaming
  GET /agents/{id}/ui-config → metadata
  GET /registry/agents → list
  Exposure mode enforcement (subagent_only → 403)
  ✅ CHECK: curl POST /agents/support_lead → JSON response
            curl POST /agents/support_lead/stream → SSE event stream
            curl POST /agents/classifier (if subagent_only) → 403
            curl GET /registry/agents → list of agent IDs

TASK 1.14  File-based registry
  Read agents.json from disk at startup
  List, get_agent_config operations
  ✅ CHECK: FileRegistryBackend("config/").list_agents() → 4 agents
            get_agent_config("support_lead") → config dict

TASK 1.15  CLI: compile, run, serve, list
  strands-platform compile config/ --validate  (dry-run, exit code)
  strands-platform run config/ --agent support_lead  (interactive chat)
  strands-platform serve config/ --port 8080  (FastAPI server)
  strands-platform list config/  (print agents, tools, deps)
  ✅ CHECK: compile --validate on valid config → exit 0
            compile --validate on circular deps → exit 1 with cycle path
            run --agent starts interactive loop
            serve starts HTTP server, responds to requests
            list prints agent names and dependency tree

TASK 1.16  Error handling in sub-agent tool wrappers
  Catch exceptions inside @tool body
  Yield error event with agent name + exception summary
  Return structured error string the parent agent can reason about
  ✅ CHECK: Sub-agent that throws → parent receives tool error with agent name
            Streaming client receives error SSE event, not a connection drop

TASK 1.17  Integration test: multi-turn conversation
  Send message 1 with session_id → response
  Send message 2 with same session_id → response references message 1 context
  ✅ CHECK: Message 2 response contains context from message 1
            (requires a model that can reference prior turns —
             use a mock model for CI, real model for manual verification)

TASK 1.18  Benchmark: Agent() construction latency
  Measure Agent() constructor time with realistic tool counts (5, 10, 20 tools).
  Measure for a 3-level hierarchy (orchestrator → specialist → sub-specialist):
    total overhead = sum of all Agent() constructions per request.
  Document results in benchmarks/agent_construction.md.
  Decision gate:
    - If per-Agent() < 50ms with 10 tools → acceptable, no change needed.
    - If per-Agent() 50-200ms → add agent instance pooling: cache Agent objects
      keyed by config hash, reset conversation state between requests instead
      of full reconstruction. Implement as AgentPool in execution_plane/pool.py.
    - If per-Agent() > 200ms → pool is mandatory, file SDK issue upstream.
  ✅ CHECK: benchmarks/agent_construction.md exists with measured numbers.
            3-level hierarchy total overhead documented.
            If >50ms/agent: AgentPool implemented and engine uses it.
            If ≤50ms/agent: Decision documented as "acceptable, no pool needed."
```

**Milestone 1 exit criteria**: `strands-platform serve config/` starts, accepts HTTP requests and SSE streams, routes to correct agent, streams sub-agent events to client, persists conversation history across turns, handles errors gracefully. Agent definitions are JSON only. No skills, SOPs, workflows, governance, or dynamic capabilities yet.

---

### Milestone 2: GOVERNED PLATFORM
*Goal: Multi-agent workflows, skills, SOPs, governance, registry with versioning.*
*Duration estimate: 4-6 weeks*

```
TASK 2.01  MCP resolver + connection pool
  Resolve MCP configs → MCPClient params
  Connection pool: open at startup, health-check, close on shutdown
  Support SSE and stdio transports
  ✅ CHECK: Connect to a test MCP server → tool list returned
            Health check detects disconnected server → warning logged
            Shutdown closes all connections (no leaked subprocesses)

TASK 2.02  SOP resolver
  Load .sop.md files
  Substitute parameters (required params validated)
  Validate structure (## Steps heading, RFC 2119 keywords in Constraints)
  ✅ CHECK: Load SOP with parameters → substituted content string
            Missing required param → SOPValidationError
            SOP without Steps heading → validation warning

TASK 2.03  Skill resolver
  discover_skills() on directory
  Validate AgentSkills.io spec (name, description, path traversal, file size)
  Three patterns: file (add file_read), tool (create_skill_tool), meta-tool (create_skill_agent_tool)
  generate_skills_prompt() for Phase 1 metadata injection
  Filter skills by skills.filter whitelist
  ✅ CHECK: Discover 3 skills → 3 SkillDefinition objects
            Skill with "../etc/passwd" reference → rejected
            Pattern "tool" → skill() tool in agent's tools
            Pattern "meta-tool" → use_skill() tool with additional_tools passed to sub-agents
            Filter ["web-research"] → only 1 skill available

TASK 2.04  Graph compiler
  WorkflowIR (type=graph) → GraphBuilder → build()
  Nodes: agents, swarms, other graphs, A2A remote agents
  Edges: unconditional + conditional (dotted path → callable)
  max_node_executions, execution_timeout
  Hook attachment via set_hook_providers
  ✅ CHECK: 4-node graph with conditional edges compiles
            Condition "conditions.is_billing" resolves to callable
            max_node_executions=10 → builder.set_max_node_executions(10) called
            Invalid condition path → ResolutionError

TASK 2.05  Swarm compiler
  WorkflowIR (type=swarm) → Swarm
  Agent resolution, coordination pattern
  ✅ CHECK: Swarm with 3 agents compiles
            coordination="competitive" passes through

TASK 2.06  Workflow compiler (DAG via strands_tools.workflow)
  IMPORTANT: Unlike Graph (strands.multiagent.GraphBuilder) and Swarm
  (strands.multiagent.Swarm), Workflow is NOT a class-level primitive.
  It is a tool (strands_tools.workflow) that an agent invokes via
  workflow(action="create", ...) then workflow(action="start", ...).
  See: strandsagents.com/docs/user-guide/concepts/multi-agent/workflow/

  Compilation approach:
  1. WorkflowIR (type=workflow) → generate a coordinator Agent whose sole
     job is to call the workflow tool.
  2. Coordinator agent config:
     - system_prompt: "Execute the workflow exactly as defined. Do not
       modify tasks. Call workflow create, then workflow start, then
       report the results."
     - tools: [strands_tools.workflow]
     - The task list, dependencies, priorities, and system_prompts from
       WorkflowIR are baked into the coordinator's system_prompt as a
       structured block the LLM passes verbatim to workflow(action="create").
  3. The coordinator agent IS the compilation target. It gets wrapped as
     a tool (same as agent-as-tool) so other agents can invoke it.
  4. Workflow tool features used: create, start, status, pause, resume.
     Parallel execution of independent tasks handled by the tool internally.

  ✅ CHECK: 5-task workflow with dependencies compiles → coordinator agent created
            Coordinator agent has strands_tools.workflow in its tools
            Coordinator's system_prompt contains all 5 tasks with correct deps
            Independent tasks identified for parallel execution by the workflow tool
            Invoking coordinator with a message → workflow created + started + result returned

TASK 2.07  Workflow-as-tool + swarm-as-tool wrappers
  Same async streaming pattern as agent-as-tool
  ✅ CHECK: Agent with {"workflow": "pipeline"} → tool wrapper generated
            Wrapper yields streaming events from workflow execution

TASK 2.08  Handoff
  {"handoff": "user"} → strands_tools.handoff_to_user
  Description override support
  ✅ CHECK: Agent with handoff tool → handoff_to_user in tools list

TASK 2.09  Multi-agent streaming events
  format_sse_event handles: multiagent_node_start, multiagent_node_stream,
    multiagent_node_stop, multiagent_handoff, multiagent_result
  Workflow SSE endpoints
  ✅ CHECK: Graph execution streams node_start/stop events via SSE
            Client receives handoff event when control transfers between nodes

TASK 2.10  Policy compiler
  tool_policy → hook attachment + tool filtering
  read_only: filter to read-safe tools
  safe_write: filter out destructive
  approval_required: attach BeforeToolCallEvent interrupt hook
  no_internet: filter external + block non-internal MCP
  tenant_scoped: attach tenant scope hook
  ✅ CHECK: Agent with no_internet policy + web_search tool → web_search filtered out
            Agent with approval_required + destructive tool → interrupt hook attached
            Warning logged for destructive tools under approval

TASK 2.11  Cross-validate policy vs dynamic capabilities
  If agent has no_internet policy + dynamic use_agent allowing web_search → compile error
  allowed_tools in dynamic_capabilities must survive policy filtering
  ✅ CHECK: Contradictory config → CompilationError with explanation

TASK 2.12  Bedrock Guardrails integration
  GuardrailsConfig → configure guardrail ID on model invocations
  Guardrail hit → safe fallback + log
  No guardrails in production → GovernanceWarning
  ✅ CHECK: Guardrail configured → model call includes guardrail params
            Blocked response → fallback returned, audit log entry created

TASK 2.13  Cost management
  Allocation tags → applied to Bedrock API calls
  Token budget hook: track tokens per request, terminate when exceeded
  Prompt caching config
  ✅ CHECK: Tags appear in Bedrock API call metadata
            50k token budget, 60k response → terminated after response, warning logged

TASK 2.14  Data governance
  PII scanning hook (prohibited_data_patterns)
  Data residency validation at compile time
  ✅ CHECK: Tool output containing "SSN: 123-45-6789" → blocked/redacted
            Agent with us-west-2 model + eu-west-1 MCP server + residency="us" → compile error

TASK 2.15  Audit logging hook
  Log all invocations, tool calls, guardrail hits, access decisions
  Structured JSON format
  CloudTrail integration for AgentCore target
  ✅ CHECK: Agent invocation → audit log entry with agent_id, session_id, tools_used, duration

TASK 2.16  DynamoDB registry backend + compile-on-publish
  put/get agent configs
  Publish: create immutable version, update LATEST pointer (atomic via condition expression)
  Rollback: point LATEST to previous version
  List: scan with projection

  COMPILE-ON-PUBLISH SEMANTICS (resolves critique UNDER 2):
  The store backend uses a different execution model than the file backend:
  - File backend: compile at startup, cache in memory, serve from cache.
  - Store backend: compile at PUBLISH TIME, store the output, hydrate per request.

  On publish:
  1. Full compilation pipeline runs (parse → validate → resolve → dep graph → compile).
  2. Two artifacts stored per version in DynamoDB:
     a. config_blob: the raw user config JSON (for rollback, audit, re-compilation).
     b. compiled_config: a SERIALIZABLE agent_config dict. Contains:
        - model params (provider, model_id, region) — NOT model objects
        - tool names (list[str]) — NOT tool function objects
        - hook names (list[str]) — NOT hook instances
        - system_prompt (str)
        - session manager config (dict)
        - all other serializable fields
  3. Compilation warnings/errors block the publish if severity >= ERROR.

  On request (engine.get_agent_config):
  1. Load compiled_config from DynamoDB (cached with TTL).
  2. Hydrator converts compiled_config → live Agent (model objects, tool objects,
     hook instances constructed at this step).
  3. No parsing, no validation, no dependency resolution at request time.
     Those happened once at publish.

  This means:
  - Adding an agent via API → must call publish endpoint → triggers compilation.
  - Editing config in DynamoDB directly → bypasses compilation → undefined behavior.
  - The compiled_config is the contract between control plane and execution plane.

  Store previous_version_id in LATEST record to avoid scan on deprecation.

  ✅ CHECK: publish("agent_a", config, "user1") → version created, LATEST updated
            publish stores both config_blob AND compiled_config
            compiled_config is JSON-serializable (json.dumps succeeds)
            get_agent_config returns compiled_config (not raw config)
            hydrator can hydrate compiled_config into live Agent
            publish again → previous version deprecated via previous_version_id (no scan)
            rollback to v1 → LATEST points to v1
            Two simultaneous publishes → only one succeeds (condition expression)

TASK 2.17  CLI: publish, rollback
  strands-platform publish config/ --agent support_lead
  strands-platform rollback support_lead --version v-abc123
  ✅ CHECK: publish creates version record, prints version_id
            rollback changes published version, prints confirmation

TASK 2.18  Access control middleware
  Auth methods: cognito (JWT validation), iam, oauth, api_key
  Per-endpoint rate limiting (in-memory token bucket, Redis for distributed)
  Exposure mode enforcement at middleware level
  ✅ CHECK: Request without valid JWT → 401
            Requests exceeding rate limit → 429
            Request to subagent_only agent → 403

TASK 2.19  Multi-tenancy context flow
  Auth middleware extracts tenant_id from JWT claims
  tenant_id injected into invocation_state
  Session manager uses tenant_id as partition key
  TenantScopeHook verifies tool access is within tenant boundary
  ✅ CHECK: Two requests with different tenant JWTs → different session partitions
            Tool call attempting cross-tenant data access → blocked by hook
```

**Milestone 2 exit criteria**: Platform supports all 5 multi-agent patterns (graph, swarm, workflow, handoff, agent-as-tool), AgentSkills.io skills with progressive disclosure, SOPs with parameter substitution, MCP server connections, Bedrock Guardrails, tool policies, cost tracking, data governance, audit logging, access control with auth, versioned registry with publish/rollback. Multi-team ready.

---

### Milestone 3: DYNAMIC + EVALUATION + DEPLOYMENT
*Goal: Runtime sub-agent creation, quality gates, production deployment targets.*
*Duration estimate: 3-4 weeks*

```
TASK 3.01  Governed use_agent tool
  Request-scoped counter (ContextVar)
  allowed_tools, allowed_models validation
  Parent hooks inherited by sub-agents
  Async streaming (SubAgentResult yielding)
  ✅ CHECK: LLM creates 3 sub-agents in one request → all succeed
            LLM tries to create sub-agent with disallowed tool → error message returned
            Next request → counter reset, creates sub-agents again
            Sub-agent inherits parent's logging hook → sub-agent actions appear in traces

TASK 3.02  Governed swarm + workflow tools
  max_swarm_size enforcement
  allowed_patterns validation
  max_tasks enforcement for workflow
  ✅ CHECK: Swarm size 10 with max 5 → error
            Workflow with 20 tasks and max 10 → error

TASK 3.03  load_tool + dynamic MCP
  Include load_tool from strands_tools when enabled
  Include Dynamic MCP Client when enabled with security warning
  Reject dynamic_mcp in production without acknowledge_risk
  ✅ CHECK: load_tool enabled → tool in agent's tools list
            dynamic_mcp in production without ack → compile error
            dynamic_mcp with ack → warning logged, tool included

TASK 3.04  Hot reload
  Agent(load_tools_from_directory=True) when enabled
  Disabled when deployment target not in allowed_in
  ✅ CHECK: hot_reload enabled + target "development" → enabled
            hot_reload enabled + target "fargate" (not in allowed_in) → disabled + warning

TASK 3.05  Evaluation pipeline
  Separate CLI command: strands-platform eval config/ --dataset evals/quality.jsonl
  Integration with Strands Evals SDK evaluators
  Run agent against test cases, score with configured metrics
  Pass/fail reporting against thresholds
  NOT part of compile pipeline (separate operation, may take minutes)
  ✅ CHECK: eval with 10 test cases → report with per-metric scores
            accuracy=0.80 with threshold 0.85 → FAIL exit code 1
            accuracy=0.90 with threshold 0.85 → PASS exit code 0

TASK 3.06  User simulation
  Strands Evals user_simulation integration
  Multi-turn eval: simulator sends N messages, scores coherence
  ✅ CHECK: Simulation produces multi-turn conversation, scores helpfulness

TASK 3.07  Lambda deployment target
  Generate handler.py that loads registry + compiles + routes
  Generate requirements.txt, SAM/CDK template
  Optimize: pre-compile to reduce cold start
  ✅ CHECK: Generated handler responds to Lambda event
            Cold start < 10s with 5 agents

TASK 3.08  Fargate deployment target
  Generate Dockerfile, ECS task definition
  FastAPI server with health check endpoint
  ✅ CHECK: docker build → image runs
            curl /health → 200
            curl /agents/support_lead → response

TASK 3.09  AgentCore deployment target
  BedrockAgentCoreApp wrapper
  AgentCore Memory (session manager)
  AgentCore Observability (auto-enabled)
  AgentCore Gateway (MCP tool exposure)
  AgentCore Identity (auth config)
  AgentCore Policy (action boundaries)
  ✅ CHECK: Deploy to AgentCore → agent accessible via AgentCore runtime
            Observability dashboard shows traces
            Memory persists across sessions

TASK 3.10  Docker + Kubernetes deployment targets
  Generic Dockerfile (non-AWS)
  Kubernetes deployment manifest + service
  ✅ CHECK: kubectl apply → pods running, service reachable

TASK 3.11  YAML config parser
  Same IR output as JSON parser
  ✅ CHECK: Parse agents.yaml → identical PlatformIR as equivalent agents.json

TASK 3.12  Markdown config parser
  # Agent: name headers → AgentIR
  ✅ CHECK: Parse markdown agent definition → correct AgentIR

TASK 3.13  CLI: init scaffolding
  strands-platform init my-project
  Generate directory structure, example configs, placeholder files
  ✅ CHECK: init creates project dir with all expected files
            strands-platform compile my-project/config/ --validate → exit 0

TASK 3.14  Defaults merge semantics (deep merge)
  Document: agent model.model_id overrides defaults model.model_id
            but inherits defaults model.provider and model.region
  Implement: recursive dict merge with agent-level fields taking precedence
  ✅ CHECK: defaults={model:{provider:"bedrock", region:"us-west-2"}}
            agent={model:{model_id:"claude-opus"}}
            result={model:{provider:"bedrock", region:"us-west-2", model_id:"claude-opus"}}

TASK 3.15  Background agent scheduling
  EventBridge rule generation for background exposure_mode
  Cron expression in config
  ✅ CHECK: Background agent with cron "0 9 * * *" → EventBridge rule created

TASK 3.16  End-to-end integration test suite
  Test: single agent → CLI chat → correct response
  Test: multi-agent hierarchy → sub-agent streaming → correct SSE events
  Test: graph workflow → node events → correct execution order
  Test: governed dynamic → sub-agent created within limits
  Test: session persistence → multi-turn conversation history
  Test: guardrails → blocked content → safe fallback
  Test: publish + rollback → version lifecycle
  Uses mock model provider (deterministic responses) for CI
  ✅ CHECK: All tests pass in CI with no real model calls
```

**Milestone 3 exit criteria**: Complete platform. Dynamic sub-agent creation with governance. Evaluation pipeline with Strands Evals SDK. All deployment targets functional. YAML/Markdown config. Init scaffolding. End-to-end test suite passing in CI.

---

### Post-Milestone Backlog (Prioritized)

```
BACKLOG: MCP reconnection + circuit breaker
  Reconnect on disconnect with exponential backoff
  Circuit breaker: fail open after N failures, retry after cooldown

BACKLOG: Model fallback chains
  model.fallback array → try each on failure
  Log failover events

BACKLOG: A2A remote agents as graph nodes
  A2AAgent integration for cross-framework interop

BACKLOG: VS Code extension
  JSON schema autocomplete for config files
  Inline validation

BACKLOG: Builder Studio UI
  Web UI for create/edit/test/publish workflow
  Not required for platform to be functional
```

---

## Summary: What Gets Built When

```
Week 1-2:   IR, parser, schema, tool registry, hook registry
Week 3-4:   Dep graph, tool wrappers, agent compiler, engine, server, CLI, benchmark
            ─── Milestone 1: usable CLI + API (18 tasks) ───

Week 5-6:   MCP, skills, SOPs, graph/swarm/workflow compilers
Week 7-8:   Policies, guardrails, cost, audit, access control
Week 9-10:  DynamoDB registry + compile-on-publish, versioning, multi-tenancy
            ─── Milestone 2: governed multi-agent platform (19 tasks) ───

Week 11-12: Dynamic capabilities, evaluation pipeline
Week 13-14: Deployment targets, YAML/MD parsers, init, integration tests
            ─── Milestone 3: production-ready (16 tasks) ───
```

53 tasks. Every task has a verification checkpoint. Every checkpoint is testable without subjective judgment. An engineer can pick up any task, implement it, run the check, and know if it's done.
