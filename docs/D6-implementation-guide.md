# D6 — Claude Code Implementation Guide

> **Purpose**: Exact instructions for feeding docs to Claude Code to build the platform.
> **Key files**: D1-requirements.md, D2-low-level-design.md, D3-build-plan.md

---

## Files to Give Claude Code

```
docs/
├── D1-requirements.md       # WHAT to build (119 EARS requirements, architecture, object model)
├── D2-low-level-design.md   # HOW to build it (package structure, code, IR, sequences)
├── D3-build-plan.md         # WHEN to build it (52 tasks, 3 milestones, verification checks)
└── CLAUDE.md                # Project-level instructions for Claude Code
```

Do NOT give D4 (critique) or D5 (cross-correlation) to Claude Code. They're reference docs for humans reviewing the design. Claude Code needs build instructions, not retrospectives.

---

## CLAUDE.md (Drop This in Project Root)

Create this file at the root of your repo. Claude Code reads it automatically:

```markdown
# Strands Agent Platform

## What This Is
A config-driven agent platform built on the Strands Agents SDK.
Agents are definitions (JSON/YAML), not separate codebases.
One shared runtime serves all agents.

## Architecture
- Control Plane: parsing, schema validation, resolution, compilation, registry
- Execution Plane: engine, hydrator, server (FastAPI + SSE), session management
- See docs/D1-requirements.md sections 1-3 for full architecture.

## Key Technical Decisions
- Agent-as-tool uses async streaming @tool with per-invocation Agent() construction
  (NOT closure over pre-built Agent). See docs/D2-low-level-design.md section 5.
- IR dataclasses are frozen/immutable. PlatformIR is the only mutable container.
- Server uses wildcard routes (/agents/{agent_id}), not per-agent route mounting.
- Governed dynamic tools use contextvars.ContextVar for request-scoped counters.
- Session ID flows through engine → hydrator → Agent invocation for conversation persistence.
- All Strands streaming events (including multi-agent node_start/stop/handoff) forward via SSE.

## Code Style
- Python 3.13+
- Type hints on all function signatures
- No single-letter variable names in logic (loop indices fine)
- No magic numbers — named constants with comments
- Frozen dataclasses for IR, mutable only where documented
- Every module has a docstring explaining its purpose
- Tests use pytest, fixtures in conftest.py

## Dependencies
- strands-agents >= 1.0
- strands-agents-tools >= 0.2
- agentskills
- strands-agents-sops
- fastapi + uvicorn
- boto3 (for DynamoDB registry, Bedrock)
- pyyaml
- click (CLI)
- jsonschema (config validation)
- pytest, ruff, mypy (dev)

## Project Structure
See docs/D2-low-level-design.md section 1 for complete package structure.

## Build Plan
See docs/D3-build-plan.md for 52 tasks across 3 milestones.
Every task has a ✅ CHECK verification.
Build in order: TASK 1.01 → 1.02 → ... → 1.17 (Milestone 1), then 2.01-2.19, then 3.01-3.16.
```

---

## Context Strategy

Claude Code has a context window. Don't flood it. Feed the minimum context needed per task.

| Working On | Feed These Files/Sections | Approx Tokens |
|---|---|---|
| **Project bootstrap** | D1 §1-3, D2 §1 | ~3k |
| **IR dataclasses** | D2 §2 | ~4k |
| **Parser** | D2 §2 (IR types), D3 TASK 1.03 | ~5k |
| **Tool wrappers** | D2 §5 (full code), D3 TASK 1.08 | ~3k |
| **Governed tools** | D2 §6 (full code), D3 TASK 3.01 | ~3k |
| **Engine + Hydrator** | D2 §8, D3 Fix 2 code | ~4k |
| **Server** | D2 §9, D3 Fix 3 code | ~4k |
| **Policy compiler** | D2 §7, D1 §7.2 | ~3k |
| **Registry** | D2 §4, D3 TASKs 2.16-2.17 | ~4k |
| **Full milestone planning** | D1 + D3 (milestone section) | ~15k |

---

## Bootstrap Prompt

Run this first. Creates the empty project skeleton.

```
Read docs/D1-requirements.md sections 1-3 and docs/D2-low-level-design.md section 1.

Create the project:
1. pyproject.toml with dependencies: strands-agents>=1.0, strands-agents-tools>=0.2,
   agentskills, strands-agents-sops, fastapi, uvicorn, boto3, pyyaml, click, jsonschema.
   Dev deps: pytest, ruff, mypy, httpx (for FastAPI TestClient).
   Package name: strands-platform. Entry point: strands-platform = strands_platform.cli.main:app

2. The complete package structure from D2 section 1. Every directory with __init__.py.
   Every __init__.py empty for now.

3. strands_platform/errors.py with the error hierarchy from D2 section 3.

4. tests/conftest.py with:
   - A fixture that creates a temporary config directory with a valid agents.json
   - A fixture for a mock model provider that returns fixed responses

5. .github/workflows/ci.yml: checkout, setup-python 3.13, pip install -e .[dev], ruff check, mypy, pytest

Do not implement any business logic. Just the skeleton + errors + test fixtures.
```

---

## Milestone 1 Prompts (Tasks 1.01 – 1.17)

Run these in sequence. Each builds on the previous.

### TASK 1.02 — IR Dataclasses
```
Read docs/D2-low-level-design.md section 2 (Intermediate Representation).

Implement strands_platform/control_plane/parsing/ir.py with ALL dataclasses exactly as specified:
- Enums: ToolRefKind, SkillPattern, WorkflowType, ExposureMode, ToolPolicy, RiskLevel
- Frozen dataclasses: ToolRef, EdgeIR, WorkflowTaskIR, ModelConfig, SOPConfig, SkillsConfig,
  DynamicCapabilitiesConfig, HotReloadConfig, UITemplate, MemoryPolicy, OutputSchemaConfig,
  AgentIR, EndpointBindingIR, WorkflowIR, MCPServerConfig, GuardrailsConfig, CostConfig,
  DataGovernanceConfig, AccessControlConfig, AuditConfig, EvaluationDataset, EvaluationConfig,
  ObservabilityConfig
- Mutable: PlatformIR only

Write tests/test_ir.py:
- AgentIR is frozen (mutation raises)
- PlatformIR is mutable
- All enums have expected values
- ToolRef with kind=AGENT has correct fields
```

### TASK 1.03 — JSON Parser
```
Read docs/D2-low-level-design.md section 2 for IR types.
Read docs/D3-build-plan.md TASK 1.03 for acceptance criteria.

Implement strands_platform/control_plane/parsing/json_parser.py:
- parse_config_directory(path: str) → PlatformIR
- Handles agents.json, workflows.json, bindings.json, platform.json
- Applies defaults block
- Parses tool refs: "name", {"agent":"x"}, {"workflow":"y"}, {"mcp":"url"},
  {"handoff":"user"}, {"a2a":"endpoint"}
- Resolves ${VAR} from os.environ at parse time
- Validates instructions vs sop mutual exclusivity

Tests in tests/test_parser.py:
- Parse 4-agent config → PlatformIR with 4 AgentIR entries
- ${VAR} with VAR set → value substituted
- ${VAR} with VAR unset → ParseError naming the variable
- Both instructions string and sop on one agent → ParseError
- Defaults model applied to agent without explicit model
```

### TASK 1.04 — Schema Validator
```
Read docs/D3-build-plan.md TASK 1.04.

Implement:
- strands_platform/control_plane/schema/v1.py (JSON Schema dict for config v1.0)
- strands_platform/control_plane/schema/validator.py (validate using jsonschema lib)

Tests in tests/test_schema.py:
- Valid config → no error
- Missing required field → error with field path
- schema_version mismatch → SchemaVersionError with supported versions
```

### TASK 1.05 — Tool Registry + Resolver
```
Read docs/D1-requirements.md FR-5.3.1 through FR-5.3.4.
Read docs/D3-build-plan.md TASK 1.05.

Implement:
- strands_platform/control_plane/resolution/tool_registry.py
  - BUILTIN_TOOLS set (calculator, file_read, file_write, shell, python_repl,
    web_search, memory, journal, http_request, use_aws, editor, environment)
  - register_tool(name, fn), register_module(module_path)
  - Conflict detection
- strands_platform/control_plane/resolution/tool_resolver.py
  - resolve(name, context) → tool object
  - resolve_mcp(ref, mcp_servers) → MCP config

Tests in tests/test_tool_registry.py:
- Resolve "calculator" → tool object
- Resolve "nonexistent" → ResolutionError listing alternatives
- Register custom with same name as builtin → ConflictError
- register_module on a module with @tool functions → all registered
```

### TASK 1.06 — Hook Registry
```
Read docs/D3-build-plan.md TASK 1.06.

Implement:
- strands_platform/control_plane/resolution/hook_registry.py
  - register(name, hook_class), resolve(name, context) → instance
- strands_platform/observability/logging_hook.py
  - LoggingHook(HookProvider): logs BeforeInvocationEvent, AfterInvocationEvent

Tests in tests/test_hooks.py:
- "logging" → LoggingHook instance
- "unknown" → ResolutionError
```

### TASK 1.07 — Dependency Graph + Cycle Detection
```
Read docs/D2-low-level-design.md section 11.
Read docs/D3-build-plan.md TASK 1.07.

Implement strands_platform/control_plane/compilation/dependency_graph.py:
- build_dependency_graph(ir: PlatformIR) → dict[str, set[str]]
- topological_sort(deps) → list[str] (Kahn's, alphabetical tie-breaking)
- CircularDependencyError with cycle path

Tests in tests/test_dep_graph.py:
- 4-agent hierarchy → correct order (leaves first, deterministic)
- A→B→A → CircularDependencyError containing "A → B → A"
- Reference to undefined agent → ResolutionError
- Independent agents → alphabetical order
```

### TASK 1.08 — Tool Wrapper Generation
```
Read docs/D2-low-level-design.md section 5 — implement EXACTLY as shown.
Read docs/D3-build-plan.md TASK 1.08.

Implement strands_platform/control_plane/compilation/tool_wrapper.py:
- SubAgentResult dataclass
- create_streaming_agent_tool(config, name, desc) → async @tool
- create_sync_agent_tool(config, name, desc) → sync @tool
- create_agent_tool(config, name, desc, streaming=True) → factory
- _derive_description(description, system_prompt) → str

CRITICAL: Agent() is constructed INSIDE the @tool function body, NOT in a closure.
The config dict is in the closure. The Agent constructor runs per invocation.

Tests in tests/test_tool_wrapper.py:
- Mock Agent + mock stream_async → streaming wrapper yields SubAgentResult events
- Sync wrapper returns string
- Two streaming invocations → different Agent instances (different id())
- Description from "You analyze data. More details." → "You analyze data."
- Description > 200 chars → truncated with "..."
- Description override replaces derived
```

### TASK 1.09 — Agent Compiler
```
Read docs/D2-low-level-design.md agent compiler references.
Read docs/D3-build-plan.md TASK 1.09.

Implement strands_platform/control_plane/compilation/agent_compiler.py:
- AgentCompiler class
- compile(ir: AgentIR) → (agent_config_dict, tool_wrappers)
- Resolves tools, hooks, SOP/instructions, skills metadata
- Generates @tool wrappers for {"agent": ...} refs
- agent_config dict contains ONLY serializable primitives (model params, tool names, hook names)

Tests in tests/test_agent_compiler.py:
- 4-agent hierarchy compiles
- support_lead has tool wrappers for classifier, billing, technical
- Each wrapper docstring matches sub-agent description
- Config dict is JSON-serializable (json.dumps doesn't raise)
```

### TASK 1.10 — Compiler Orchestrator
```
Read docs/D2-low-level-design.md section 10.
Read docs/D3-build-plan.md TASK 1.10.

Implement strands_platform/control_plane/compilation/compiler.py:
- Compiler class: compile(ir) → Platform
- Platform class: .agents, .workflows, .registry, .get(name), .serve(port), .warnings

Wire: parse → validate → resolve → dep graph → sort → agent compile → Platform.

Implement strands_platform/__init__.py:
- compile_config(path, tool_resolver, hook_registry, dry_run) → Platform

Tests in tests/test_compiler.py:
- compile_config("tests/fixtures/config/") → Platform with expected agents
- platform.get("support_lead") works
- platform.get("nonexistent") → KeyError with alternatives
- platform.warnings is a list
```

### TASK 1.11 — Hydrator + Session
```
Read docs/D2-low-level-design.md section 8 — use the FIXED version from D3.
Read docs/D3-build-plan.md TASK 1.11 + Fix 2 code.

Implement:
- strands_platform/execution_plane/session.py (create_session_manager factory)
- strands_platform/execution_plane/hydrator.py (config dict → Agent with session)
  - Model provider cache (same config → same object)
  - Returns (agent, session_id) tuple

Tests in tests/test_hydrator.py:
- memory session → Agent with no external session manager
- dynamodb session → Agent with session manager attached
- Same model config twice → same model object (cached)
- session_id passed through
```

### TASK 1.12 — Engine
```
Read docs/D2-low-level-design.md section 8 — use FIXED version from D3.
Read docs/D3-build-plan.md TASK 1.12.

Implement strands_platform/execution_plane/engine.py:
- AgentEngine: get_agent_config (cached), invoke (sync), invoke_stream (async)
- session_id flows through
- asyncio.timeout on streaming invocation
- force_stop event on timeout

Tests in tests/test_engine.py:
- invoke passes session_id to hydrator
- invoke_stream yields events
- Timeout → force_stop event
- Config cache: second call within TTL → no registry hit
```

### TASK 1.13 — Server (Wildcard Routes + SSE)
```
Read docs/D2-low-level-design.md section 9 — use FIXED version from D3.
Read docs/D3-build-plan.md TASK 1.13.

Implement:
- strands_platform/execution_plane/server.py (wildcard routes, exposure mode enforcement)
- strands_platform/execution_plane/streaming.py (format_sse_event)

Routes:
- POST /agents/{agent_id} → request-response
- POST /agents/{agent_id}/stream → SSE
- GET /agents/{agent_id}/ui-config → metadata
- POST /agents/{agent_id}/v/{version_id} → version-pinned
- POST /agents/{agent_id}/trigger → background fire-and-forget
- POST /workflows/{workflow_id} and /stream variants
- GET /registry/agents → list

Tests in tests/test_server.py (FastAPI TestClient):
- POST existing agent → 200
- POST subagent_only → 403
- GET ui-config → correct ui_mode
- GET registry/agents → list
```

### TASK 1.14 — File Registry Backend
```
Read docs/D2-low-level-design.md section 4.2.
Read docs/D3-build-plan.md TASK 1.14.

Implement strands_platform/control_plane/registry/file_backend.py.

Tests in tests/test_registry.py:
- list_agents → expected count
- get_agent_config → dict with expected keys
- Unknown agent → KeyError with available list
```

### TASK 1.15 — CLI
```
Read docs/D3-build-plan.md TASK 1.15.

Implement CLI with Click in strands_platform/cli/:
- main.py: app group
- compile_cmd.py: compile --validate (dry-run, exit codes)
- run_cmd.py: run --agent (interactive loop)
- serve_cmd.py: serve --port (uvicorn)
- list_cmd.py: list (print agents, deps)

Tests in tests/test_cli.py (Click CliRunner):
- compile valid config → exit 0
- compile circular deps → exit 1
- list → stdout contains agent names
```

### TASK 1.16 — Error Handling in Wrappers
```
Read docs/D3-build-plan.md TASK 1.16.

Update tool_wrapper.py: wrap Agent() and stream_async() in try/except.
On error: yield {"error": True, "agent": name, "message": str(exc)} as SubAgentResult.
Then yield f"Error in {name}: {exc}" as the final string result.

Tests in tests/test_tool_wrapper_errors.py:
- Agent that raises → wrapper yields error event, doesn't crash parent
- Error event contains agent name
```

### TASK 1.17 — Multi-Turn Integration Test
```
Read docs/D3-build-plan.md TASK 1.17.

tests/test_integration_multiturn.py:
- Create a mock model that echoes input + remembers previous turns
- Invoke agent with session_id="test-session", message="My name is Alice"
- Invoke again with same session_id, message="What's my name?"
- Assert second response references "Alice"

This is the M1 exit test.
```

### TASK 1.18 — Agent() Construction Benchmark
```
Read docs/D3-build-plan.md TASK 1.18.

Create benchmarks/agent_construction.py:
- Measure Agent() constructor time with 5, 10, 20 tools
- Measure 3-level hierarchy total overhead (3+ constructions)
- Use time.perf_counter_ns for precision
- Run 100 iterations, report mean/p50/p95/p99

Write results to benchmarks/agent_construction.md.

Decision gate (implement in code):
- If per-Agent() < 50ms with 10 tools → document "acceptable" and move on
- If 50-200ms → implement AgentPool in execution_plane/pool.py:
  - Cache Agent objects keyed by config hash
  - Reset conversation state between requests (clear messages, keep tools/model)
  - Wire into engine.py as optional optimization
- If > 200ms → pool mandatory, add TODO to file SDK issue

This is a go/no-go gate. The benchmark determines whether we need
an optimization layer or not. Don't skip it.
```

---

## Milestone 2 Prompts (Tasks 2.01 – 2.19)

Same structure. For each task:
1. Point Claude Code to the D2 section with the implementation code
2. Quote the D3 task block with the ✅ CHECK
3. One task per prompt

Key tasks to pay attention to:

- **2.01 (MCP)**: Connection pool, lifecycle management. Reference D2 for MCPClient usage.
- **2.03 (Skills)**: Three patterns. Reference D1 §5.5 for progressive disclosure spec.
- **2.04 (Graph)**: Conditional edges, cycles, timeout. Reference D1 §5.7.1-5.7.5.
- **2.06 (Workflow)**: NOT a class primitive like Graph. Workflow is `strands_tools.workflow` — a tool an agent calls. The compiler generates a coordinator Agent whose job is to invoke `workflow(action="create")` then `workflow(action="start")`. Read the full TASK 2.06 in D3 carefully — it explains the SDK reality and the compilation approach.
- **2.10 (Policies)**: Reference D2 §7 policy compiler code exactly.
- **2.11 (Cross-validation)**: Policy vs dynamic capabilities conflict. Reference D4 PROBLEM 4.
- **2.16 (DynamoDB registry + compile-on-publish)**: This task now includes the compile-on-publish semantics. On publish: full compilation runs, stores both raw config_blob AND serializable compiled_config. On request: hydrator converts compiled_config to live Agent. No parsing/validation/dep-resolution at request time. Atomic publish with condition expressions. Read the full TASK 2.16 in D3 — it's significantly expanded.
- **2.18 (Access control)**: Auth middleware, rate limiting. Reference D1 §7.5.
- **2.19 (Multi-tenancy)**: tenant_id flow from JWT → invocation_state → session partition.

---

## Milestone 3 Prompts (Tasks 3.01 – 3.16)

Key tasks:

- **3.01 (Governed use_agent)**: Reference D3 Fix 1 code (ContextVar). Critical to get right.
- **3.05 (Evaluation)**: Separate CLI command, NOT part of compile. Reference D4 critique.
- **3.07-3.10 (Deployment targets)**: Reference D1 §11 for target-specific requirements.
- **3.09 (AgentCore)**: All 6 AgentCore services. Reference D1 DR-11.1.2.
- **3.16 (Integration test suite)**: Full end-to-end. Uses mock model. The M3 exit test.

---

## Testing Strategy

| Level | What | Mock | Real |
|---|---|---|---|
| **Unit** | IR, parser, schema, resolver, dep graph, tool wrapper | Everything | Nothing |
| **Integration** | Compiler end-to-end, engine + hydrator | Model provider | Strands SDK |
| **Server** | FastAPI endpoints, SSE, auth | Model provider, session store | FastAPI, routing |
| **End-to-end** | Full request flow through all layers | Model provider | Everything else |
| **Evaluation** | Agent quality against test datasets | Nothing | Model provider (real calls) |

Mock model provider for all CI tests. Real model calls only in evaluation and manual testing.

---

## Verification Protocol

After each milestone, run this checklist:

### Milestone 1 Exit
```bash
pytest tests/ -v                         # All unit + integration tests pass
ruff check strands_platform/             # No lint errors
mypy strands_platform/                   # Type checks pass
strands-platform compile tests/fixtures/config/ --validate   # Exit 0
strands-platform list tests/fixtures/config/                 # Prints 4 agents
# Manual: strands-platform run tests/fixtures/config/ --agent support_lead
# Manual: strands-platform serve tests/fixtures/config/ --port 8080
#         curl POST /agents/support_lead → response
#         curl POST /agents/support_lead/stream → SSE events
```

### Milestone 2 Exit
```bash
pytest tests/ -v                         # All M1 + M2 tests pass
strands-platform compile tests/fixtures/config-governed/ --validate
# Should show: guardrails configured, policies applied, no warnings
strands-platform publish tests/fixtures/config-governed/ --agent support_lead
strands-platform rollback support_lead --version v-001
# Manual: verify auth enforcement (401 without JWT)
# Manual: verify rate limiting (429 after burst)
# Manual: verify subagent_only returns 403
```

### Milestone 3 Exit
```bash
pytest tests/ -v                         # All M1 + M2 + M3 tests pass
strands-platform eval tests/fixtures/config/ --dataset tests/evals/basic.jsonl
# Should report pass/fail per metric
strands-platform deploy tests/fixtures/config/ --target docker
docker build -t platform-test deploy/
docker run -p 8080:8080 platform-test
# curl /agents/support_lead → response from Docker container
```
