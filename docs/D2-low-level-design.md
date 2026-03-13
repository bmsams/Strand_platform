# Strands Agent Platform — Low-Level Design

> **Implements**: Platform Requirements & Architecture v2.0
> **Audience**: Engineers building this system
> **Language**: Python 3.13+
> **Core Dependency**: strands-agents ≥1.0, strands-agents-tools ≥0.2, agentskills, strands-agents-sops, strands-agents-evals

---

## 1. Package Structure

Maps directly to the three-layer architecture and two-plane separation from the requirements.

```
strands_platform/
│
├── control_plane/                    # THE BUILDER SIDE
│   ├── __init__.py
│   ├── registry/
│   │   ├── __init__.py
│   │   ├── models.py                # AgentDefinition, AgentVersion, ToolDef, SkillDef, EndpointBinding
│   │   ├── file_backend.py          # File-based registry (config on disk)
│   │   ├── store_backend.py         # DynamoDB-based registry (multi-team)
│   │   └── api.py                   # GET /registry/agents, publish, rollback
│   ├── parsing/
│   │   ├── __init__.py
│   │   ├── ir.py                    # Frozen IR dataclasses
│   │   ├── json_parser.py           # JSON/YAML → PlatformIR
│   │   └── markdown_parser.py       # Markdown agent defs → PlatformIR
│   ├── schema/
│   │   ├── __init__.py
│   │   ├── v1.py                    # JSON Schema definition
│   │   └── validator.py             # Schema validation + version dispatch
│   ├── resolution/
│   │   ├── __init__.py
│   │   ├── tool_registry.py         # ToolRegistry: builtin + custom + risk levels
│   │   ├── hook_registry.py         # HookRegistry: name → HookProvider class
│   │   ├── tool_resolver.py         # ToolRef → concrete tool object
│   │   ├── condition_resolver.py    # Dotted paths → callables
│   │   ├── mcp_resolver.py          # MCP configs → MCPClient params
│   │   ├── skill_resolver.py        # Discover + validate + create skill tools
│   │   └── sop_resolver.py          # Load + validate + parameterize SOPs
│   └── compilation/
│       ├── __init__.py
│       ├── dependency_graph.py      # Build dep graph, cycle detection, topo sort
│       ├── agent_compiler.py        # AgentIR → agent config dict (for runtime hydration)
│       ├── graph_compiler.py        # WorkflowIR → GraphBuilder instructions
│       ├── swarm_compiler.py        # WorkflowIR → Swarm instructions
│       ├── workflow_compiler.py     # WorkflowIR → workflow tool config
│       ├── tool_wrapper.py          # Async streaming + sync @tool wrapper generation
│       ├── governed_tools.py        # Governed use_agent, swarm, workflow wrappers
│       ├── policy_compiler.py       # tool_policy → hook attachment + tool filtering
│       └── compiler.py              # Orchestrates full pipeline, produces Platform
│
├── execution_plane/                  # THE RUNTIME SIDE
│   ├── __init__.py
│   ├── engine.py                    # AgentEngine: load config → assemble → execute
│   ├── hydrator.py                  # Config dict → live Strands Agent() at request time
│   ├── server.py                    # FastAPI app: endpoints, SSE, routing
│   ├── bindings.py                  # EndpointBinding → FastAPI route generation
│   ├── streaming.py                 # SSE event formatting + multi-agent event forwarding
│   ├── session.py                   # Session manager factory (memory, dynamo, s3, agentcore)
│   └── auth.py                      # Auth middleware (cognito, iam, oauth, api_key)
│
├── governance/                       # POLICIES + GUARDRAILS + AUDIT
│   ├── __init__.py
│   ├── guardrails.py                # Bedrock Guardrails integration hook
│   ├── tool_policy.py               # read_only, safe_write, approval_required, no_internet, tenant_scoped
│   ├── data_governance.py           # PII scanning hook, residency validation
│   ├── cost.py                      # Token budget hook, allocation tags, prompt caching config
│   ├── audit.py                     # Immutable audit log hook, CloudTrail integration
│   └── rate_limiter.py              # Per-endpoint rate limiting middleware
│
├── evaluation/                       # QUALITY GATES
│   ├── __init__.py
│   ├── runner.py                    # Eval pipeline: load dataset, run agent, score
│   └── strands_evals.py             # Strands Evals SDK integration
│
├── deployment/                       # DEPLOYMENT ARTIFACT GENERATION
│   ├── __init__.py
│   ├── lambda_target.py
│   ├── fargate_target.py
│   ├── ec2_target.py
│   ├── agentcore_target.py          # BedrockAgentCoreApp + all AgentCore services
│   ├── docker_target.py
│   └── kubernetes_target.py
│
├── observability/                    # TRACES + LOGS + METRICS
│   ├── __init__.py
│   ├── logging_hook.py              # Built-in structured logging HookProvider
│   └── otel_config.py              # OTEL endpoint configuration
│
├── cli/                              # CLI COMMANDS
│   ├── __init__.py
│   ├── main.py                      # Click/Typer app
│   ├── init_cmd.py                  # strands-platform init
│   ├── compile_cmd.py               # strands-platform compile --validate
│   ├── run_cmd.py                   # strands-platform run --agent/--workflow
│   ├── serve_cmd.py                 # strands-platform serve
│   ├── deploy_cmd.py                # strands-platform deploy
│   ├── list_cmd.py                  # strands-platform list
│   ├── publish_cmd.py               # strands-platform publish
│   └── rollback_cmd.py              # strands-platform rollback
│
├── errors.py                        # Exception hierarchy
├── _version.py
└── __init__.py                      # Public API: compile_config(), Platform
```

---

## 2. Intermediate Representation

All parsers produce the same frozen IR. The compiler operates on IR only, never raw config. The IR maps 1:1 to the object model in requirements Section 4.

```python
# strands_platform/control_plane/parsing/ir.py

from __future__ import annotations
from dataclasses import dataclass, field
from enum import Enum
from typing import Any


# ═══════════════════════════════════════════════════════
# ENUMS
# ═══════════════════════════════════════════════════════

class ToolRefKind(Enum):
    BUILTIN = "builtin"
    CUSTOM = "custom"
    AGENT = "agent"
    WORKFLOW = "workflow"
    MCP = "mcp"
    HANDOFF = "handoff"
    A2A = "a2a"


class SkillPattern(Enum):
    FILE = "file"
    TOOL = "tool"
    META_TOOL = "meta-tool"


class WorkflowType(Enum):
    GRAPH = "graph"
    SWARM = "swarm"
    WORKFLOW = "workflow"


class ExposureMode(Enum):
    CHAT = "chat"
    API = "api"
    FORM = "form"
    EMBEDDED = "embedded"
    SUBAGENT_ONLY = "subagent_only"
    BACKGROUND = "background"


class ToolPolicy(Enum):
    READ_ONLY = "read_only"
    SAFE_WRITE = "safe_write"
    APPROVAL_REQUIRED = "approval_required"
    NO_INTERNET = "no_internet"
    TENANT_SCOPED = "tenant_scoped"


class RiskLevel(Enum):
    READ_ONLY = "read_only"
    SAFE_WRITE = "safe_write"
    DESTRUCTIVE = "destructive"
    EXTERNAL = "external"


# ═══════════════════════════════════════════════════════
# TOOL & WORKFLOW REFERENCES
# ═══════════════════════════════════════════════════════

@dataclass(frozen=True)
class ToolRef:
    kind: ToolRefKind
    name: str
    description_override: str | None = None
    streaming: bool = True        # Default: async streaming wrapper
    mcp_url: str | None = None
    mcp_transport: str = "sse"
    a2a_endpoint: str | None = None


@dataclass(frozen=True)
class EdgeIR:
    from_node: str
    to_node: str
    condition_path: str | None = None


@dataclass(frozen=True)
class WorkflowTaskIR:
    task_id: str
    agent_ref: str
    description: str | None = None
    depends_on: tuple[str, ...] = ()
    priority: int | None = None


# ═══════════════════════════════════════════════════════
# MODEL CONFIGURATION
# ═══════════════════════════════════════════════════════

@dataclass(frozen=True)
class ModelConfig:
    provider: str = "bedrock"
    model_id: str = "us.anthropic.claude-sonnet-4-20250514-v1:0"
    region: str = "us-west-2"
    custom_class: str | None = None


# ═══════════════════════════════════════════════════════
# AGENT SUB-CONFIGS
# ═══════════════════════════════════════════════════════

@dataclass(frozen=True)
class SOPConfig:
    file: str
    parameters: dict[str, str] = field(default_factory=dict)


@dataclass(frozen=True)
class SkillsConfig:
    directory: str
    pattern: SkillPattern
    additional_tools: tuple[str, ...] = ()
    filter: tuple[str, ...] | None = None


@dataclass(frozen=True)
class DynamicCapabilitiesConfig:
    use_agent_enabled: bool = False
    use_agent_allowed_tools: tuple[str, ...] = ()
    use_agent_allowed_models: tuple[str, ...] = ()
    use_agent_max_per_request: int = 5
    use_agent_inherit_hooks: bool = True
    use_agent_inherit_guardrails: bool = True
    swarm_enabled: bool = False
    swarm_max_size: int = 5
    swarm_allowed_patterns: tuple[str, ...] = ("collaborative",)
    workflow_enabled: bool = False
    workflow_max_tasks: int = 10
    load_tool_enabled: bool = False
    dynamic_mcp_enabled: bool = False
    dynamic_mcp_acknowledge_risk: bool = False


@dataclass(frozen=True)
class HotReloadConfig:
    enabled: bool = False
    directory: str = "./tools"
    allowed_in: tuple[str, ...] = ("development",)


@dataclass(frozen=True)
class UITemplate:
    ui_mode: str = "chat"   # chat | form | workflow | console | dashboard
    ui_features: dict[str, bool] = field(default_factory=dict)


@dataclass(frozen=True)
class MemoryPolicy:
    session_manager: str = "memory"  # memory | dynamodb | s3 | agentcore | valkey
    table_name: str | None = None
    ttl_hours: int = 24
    tenant_scoped: bool = False


@dataclass(frozen=True)
class OutputSchemaConfig:
    """Strands structured output configuration."""
    schema: dict[str, Any] = field(default_factory=dict)


# ═══════════════════════════════════════════════════════
# AGENT DEFINITION IR (requirements Section 4.1)
# ═══════════════════════════════════════════════════════

@dataclass(frozen=True)
class AgentIR:
    """Maps to AgentDefinition in object model.

    This is the complete specification of one agent. Everything the
    compiler needs to produce a Strands Agent or a registry record.
    """
    # Identity (question 1: what is it?)
    agent_id: str
    name: str
    description: str = ""

    # Behavior (question 4: how should it behave?)
    instructions: str | None = None         # Inline system_prompt
    sop: SOPConfig | None = None            # SOP reference (mutually exclusive with instructions)
    output_schema: OutputSchemaConfig | None = None

    # Capabilities (question 2: what can it do?)
    tool_refs: tuple[ToolRef, ...] = ()
    skills: SkillsConfig | None = None
    dynamic_capabilities: DynamicCapabilitiesConfig | None = None
    hot_reload: HotReloadConfig | None = None

    # Model
    model: ModelConfig = field(default_factory=ModelConfig)
    model_fallback: tuple[ModelConfig, ...] = ()

    # Governance (question 3: what is it allowed to do?)
    tool_policy: ToolPolicy | None = None
    hook_refs: tuple[str, ...] = ()

    # Memory
    memory: MemoryPolicy = field(default_factory=MemoryPolicy)

    # Experience (question 5: how is it exposed?)
    exposure_mode: ExposureMode = ExposureMode.API
    ui_template: UITemplate = field(default_factory=UITemplate)

    max_iterations: int = 20


# ═══════════════════════════════════════════════════════
# ENDPOINT BINDING IR
# ═══════════════════════════════════════════════════════

@dataclass(frozen=True)
class EndpointBindingIR:
    binding_id: str
    agent_id: str
    route: str
    auth_policy: str = "none"    # cognito | iam | oauth | api_key | none
    rate_limit_rpm: int | None = None
    ui_mode: str = "none"
    ui_features: dict[str, bool] = field(default_factory=dict)


# ═══════════════════════════════════════════════════════
# WORKFLOW IR
# ═══════════════════════════════════════════════════════

@dataclass(frozen=True)
class WorkflowIR:
    name: str
    workflow_type: WorkflowType
    # Graph
    nodes: dict[str, ToolRef] = field(default_factory=dict)
    edges: tuple[EdgeIR, ...] = ()
    entry_point: str | None = None
    max_node_executions: int | None = None
    execution_timeout: int | None = None
    # Swarm
    agent_refs: tuple[str, ...] = ()
    coordination: str = "collaborative"
    # Workflow (DAG)
    tasks: dict[str, WorkflowTaskIR] = field(default_factory=dict)
    # Shared
    hook_refs: tuple[str, ...] = ()


# ═══════════════════════════════════════════════════════
# PLATFORM-LEVEL CONFIG IR
# ═══════════════════════════════════════════════════════

@dataclass(frozen=True)
class MCPServerConfig:
    name: str
    url: str | None = None
    command: str | None = None
    args: tuple[str, ...] = ()
    env: dict[str, str] = field(default_factory=dict)
    transport: str = "sse"


@dataclass(frozen=True)
class GuardrailsConfig:
    bedrock_guardrail_id: str | None = None
    bedrock_guardrail_version: str = "DRAFT"
    apply_to: str = "all"


@dataclass(frozen=True)
class CostConfig:
    allocation_tags: dict[str, str] = field(default_factory=dict)
    budgets: dict[str, dict[str, int]] = field(default_factory=dict)
    enable_prompt_caching: bool = False


@dataclass(frozen=True)
class DataGovernanceConfig:
    classification_level: str = "internal"
    prohibited_data_patterns: tuple[str, ...] = ()
    data_residency: str | None = None


@dataclass(frozen=True)
class AccessControlConfig:
    default_auth: str = "none"
    cognito_user_pool_id: str | None = None


@dataclass(frozen=True)
class AuditConfig:
    enabled: bool = False
    cloudtrail_integration: bool = False


@dataclass(frozen=True)
class EvaluationDataset:
    file: str
    metrics: tuple[str, ...] = ()
    pass_threshold: dict[str, float] = field(default_factory=dict)


@dataclass(frozen=True)
class EvaluationConfig:
    datasets: dict[str, EvaluationDataset] = field(default_factory=dict)
    run_on: str = "manual"
    judge_model: str | None = None


@dataclass(frozen=True)
class ObservabilityConfig:
    otel_endpoint: str | None = None
    service_name: str = "strands-platform"


@dataclass
class PlatformIR:
    """Top-level container. Only mutable IR object — built incrementally
    from multiple config files (agents.json, workflows.json, bindings.json, platform.json).
    """
    schema_version: str = "1.0"
    agents: dict[str, AgentIR] = field(default_factory=dict)
    workflows: dict[str, WorkflowIR] = field(default_factory=dict)
    bindings: dict[str, EndpointBindingIR] = field(default_factory=dict)
    mcp_servers: dict[str, MCPServerConfig] = field(default_factory=dict)
    global_hooks: tuple[str, ...] = ()
    guardrails: GuardrailsConfig = field(default_factory=GuardrailsConfig)
    cost: CostConfig = field(default_factory=CostConfig)
    data_governance: DataGovernanceConfig = field(default_factory=DataGovernanceConfig)
    access_control: AccessControlConfig = field(default_factory=AccessControlConfig)
    audit: AuditConfig = field(default_factory=AuditConfig)
    evaluation: EvaluationConfig = field(default_factory=EvaluationConfig)
    observability: ObservabilityConfig = field(default_factory=ObservabilityConfig)
    deployment_target: str = "local"
```

### Why This Matters

Every field in `AgentIR` maps to a specific question from the platform contract. Every governance config maps to a specific AWS golden pathway requirement. The frozen dataclasses guarantee the IR is a read-only snapshot: the compiler never mutates what the user wrote.

---

## 3. Error Hierarchy

```python
# strands_platform/errors.py

class PlatformError(Exception):
    """Base for all platform errors."""
    pass

class ParseError(PlatformError):
    """Config is syntactically invalid."""
    pass

class SchemaVersionError(PlatformError):
    def __init__(self, found: str, supported: list[str]):
        self.found, self.supported = found, supported
        super().__init__(
            f"Schema version '{found}' not supported. "
            f"Supported: {', '.join(supported)}. Run 'strands-platform migrate'."
        )

class ResolutionError(PlatformError):
    """A reference cannot be resolved."""
    def __init__(self, ref_type: str, ref_name: str, context: str,
                 available: list[str] | None = None):
        self.ref_type, self.ref_name, self.context, self.available = \
            ref_type, ref_name, context, available
        msg = f"Cannot resolve {ref_type} '{ref_name}' referenced by {context}."
        if available:
            msg += f" Available: {', '.join(sorted(available))}"
        super().__init__(msg)

class CircularDependencyError(PlatformError):
    def __init__(self, cycle: list[str]):
        self.cycle = cycle
        super().__init__(f"Circular dependency: {' → '.join(cycle)}")

class ConflictError(PlatformError):
    def __init__(self, name: str, source_a: str, source_b: str, context: str):
        super().__init__(
            f"Conflict for '{name}': found in {source_a} and {source_b}. "
            f"Referenced by {context}."
        )

class PolicyViolationError(PlatformError):
    """Agent config violates its tool_policy."""
    pass

class GovernanceWarning:
    """Non-fatal. Collected during compilation, reported at end."""
    def __init__(self, code: str, message: str):
        self.code, self.message = code, message
    def __repr__(self):
        return f"⚠ [{self.code}] {self.message}"
```

---

## 4. Control Plane: Registry

### 4.1 Registry Interface

```python
# strands_platform/control_plane/registry/models.py

from dataclasses import dataclass, field
from datetime import datetime
from typing import Any, Protocol
import uuid


@dataclass
class AgentVersion:
    version_id: str
    agent_id: str
    config_blob: dict[str, Any]   # Serialized AgentIR
    created_by: str
    created_at: datetime
    status: str = "draft"         # draft | published | deprecated | rolled_back


class RegistryBackend(Protocol):
    """Interface for registry storage backends."""

    def list_agents(self) -> list[dict]:
        """List all agents with current published version."""
        ...

    def get_agent_config(self, agent_id: str, version_id: str | None = None) -> dict:
        """Get agent config. If version_id is None, return published version."""
        ...

    def publish(self, agent_id: str, config: dict, created_by: str) -> AgentVersion:
        """Create new published version. Deprecate previous."""
        ...

    def rollback(self, agent_id: str, target_version_id: str) -> AgentVersion:
        """Set published version to a previous version."""
        ...

    def get_bindings(self, agent_id: str) -> list[dict]:
        """Get all endpoint bindings for an agent."""
        ...
```

### 4.2 File Backend (Dev/Single-Team)

```python
# strands_platform/control_plane/registry/file_backend.py

from pathlib import Path
import json
from datetime import datetime, timezone
from .models import RegistryBackend, AgentVersion


class FileRegistryBackend(RegistryBackend):
    """File-based registry. Config on disk, compiled at startup.

    For single-team dev use. No versioning persistence — versions
    are ephemeral (current config = current version).
    """

    def __init__(self, config_dir: str):
        self._dir = Path(config_dir)
        self._agents: dict[str, dict] = {}
        self._bindings: dict[str, list[dict]] = {}
        self._load()

    def _load(self):
        # Load agents.json (or .yaml)
        for ext in (".json", ".yaml", ".yml"):
            agents_file = self._dir / f"agents{ext}"
            if agents_file.exists():
                raw = json.loads(agents_file.read_text()) if ext == ".json" \
                    else self._load_yaml(agents_file)
                for agent_id, config in raw.get("agents", {}).items():
                    self._agents[agent_id] = config

        # Load bindings.json
        bindings_file = self._dir / "bindings.json"
        if bindings_file.exists():
            raw = json.loads(bindings_file.read_text())
            self._bindings = raw.get("bindings", {})

    def list_agents(self) -> list[dict]:
        return [{"agent_id": k, "status": "published"} for k in self._agents]

    def get_agent_config(self, agent_id: str, version_id: str | None = None) -> dict:
        if agent_id not in self._agents:
            raise KeyError(f"Agent '{agent_id}' not in registry. "
                           f"Available: {sorted(self._agents.keys())}")
        return self._agents[agent_id]

    def publish(self, agent_id: str, config: dict, created_by: str) -> AgentVersion:
        self._agents[agent_id] = config
        return AgentVersion(
            version_id=f"file-{datetime.now(timezone.utc).isoformat()}",
            agent_id=agent_id,
            config_blob=config,
            created_by=created_by,
            created_at=datetime.now(timezone.utc),
            status="published",
        )

    def rollback(self, agent_id: str, target_version_id: str) -> AgentVersion:
        raise NotImplementedError("File backend does not support rollback. Use store backend.")

    def get_bindings(self, agent_id: str) -> list[dict]:
        return self._bindings.get(agent_id, [])
```

### 4.3 Store Backend (Multi-Team Platform)

```python
# strands_platform/control_plane/registry/store_backend.py

import boto3
from datetime import datetime, timezone
import uuid
import json
from .models import RegistryBackend, AgentVersion


class DynamoDBRegistryBackend(RegistryBackend):
    """DynamoDB-backed registry. For multi-team platform use.

    Table schema:
      PK: agent_id
      SK: version_id (or "LATEST" for current published pointer)
      config_blob: JSON string
      created_by: string
      created_at: ISO timestamp
      status: draft | published | deprecated | rolled_back
    """

    def __init__(self, table_name: str, region: str = "us-west-2"):
        self._table = boto3.resource("dynamodb", region_name=region).Table(table_name)

    def get_agent_config(self, agent_id: str, version_id: str | None = None) -> dict:
        if version_id:
            resp = self._table.get_item(Key={"agent_id": agent_id, "sk": version_id})
        else:
            # Get LATEST pointer, then fetch that version
            resp = self._table.get_item(Key={"agent_id": agent_id, "sk": "LATEST"})
            if "Item" not in resp:
                raise KeyError(f"Agent '{agent_id}' has no published version.")
            version_id = resp["Item"]["published_version_id"]
            resp = self._table.get_item(Key={"agent_id": agent_id, "sk": version_id})

        if "Item" not in resp:
            raise KeyError(f"Version '{version_id}' not found for agent '{agent_id}'.")
        return json.loads(resp["Item"]["config_blob"])

    def publish(self, agent_id: str, config: dict, created_by: str) -> AgentVersion:
        version_id = f"v-{uuid.uuid4().hex[:12]}"
        now = datetime.now(timezone.utc).isoformat()

        # Write version record
        self._table.put_item(Item={
            "agent_id": agent_id,
            "sk": version_id,
            "config_blob": json.dumps(config),
            "created_by": created_by,
            "created_at": now,
            "status": "published",
        })

        # Update LATEST pointer
        self._table.put_item(Item={
            "agent_id": agent_id,
            "sk": "LATEST",
            "published_version_id": version_id,
        })

        # Deprecate previous (scan for old published — not ideal at scale,
        # but correct. Production optimization: store previous_version_id in LATEST.)

        return AgentVersion(
            version_id=version_id, agent_id=agent_id,
            config_blob=config, created_by=created_by,
            created_at=datetime.fromisoformat(now), status="published",
        )

    def rollback(self, agent_id: str, target_version_id: str) -> AgentVersion:
        # Verify target exists
        resp = self._table.get_item(Key={"agent_id": agent_id, "sk": target_version_id})
        if "Item" not in resp:
            raise KeyError(f"Version '{target_version_id}' not found.")

        # Update LATEST pointer
        self._table.put_item(Item={
            "agent_id": agent_id,
            "sk": "LATEST",
            "published_version_id": target_version_id,
        })

        # Update status
        self._table.update_item(
            Key={"agent_id": agent_id, "sk": target_version_id},
            UpdateExpression="SET #s = :s",
            ExpressionAttributeNames={"#s": "status"},
            ExpressionAttributeValues={":s": "published"},
        )

        return AgentVersion(
            version_id=target_version_id, agent_id=agent_id,
            config_blob=json.loads(resp["Item"]["config_blob"]),
            created_by="rollback", created_at=datetime.now(timezone.utc),
            status="published",
        )
```

---

## 5. Tool Wrapper Generation

The most critical compilation step. Follows the Strands streaming sub-agent pattern from `strandsagents.com/docs/user-guide/concepts/streaming/#sub-agent-streaming-example`.

```python
# strands_platform/control_plane/compilation/tool_wrapper.py

from typing import AsyncIterator, Any
from dataclasses import dataclass
from strands import Agent, tool


AUTO_DESCRIPTION_MAX_CHARS = 200


@dataclass
class SubAgentResult:
    """Yielded from async agent tools. Parent receives via tool_stream_event."""
    agent: Agent
    event: dict


def create_streaming_agent_tool(
    agent_config: dict,
    tool_name: str,
    description: str | None = None,
) -> callable:
    """DEFAULT: Async streaming wrapper. Fresh Agent per invocation.

    Follows the exact pattern from Strands streaming docs:
    - @tool async def → AsyncIterator
    - Agent() instantiated inside function body (per-request isolation)
    - stream_async() iteration with SubAgentResult yielding
    - Final result yielded as string

    agent_config is a dict with pre-resolved: name, system_prompt,
    model (Strands model object), tools (resolved tool objects), hooks.
    The Agent() constructor runs at each invocation, not at compile time.
    """
    desc = description or _derive_description(
        agent_config.get("description", ""),
        agent_config.get("system_prompt", ""),
    )

    @tool
    async def agent_tool(query: str) -> AsyncIterator:
        agent = Agent(
            name=agent_config["name"],
            system_prompt=agent_config["system_prompt"],
            model=agent_config["model"],
            tools=agent_config["tools"],
            hooks=agent_config.get("hooks", []),
            callback_handler=None,
        )
        result = None
        async for event in agent.stream_async(query):
            yield SubAgentResult(agent=agent, event=event)
            if "result" in event:
                result = event["result"]
        yield str(result)

    agent_tool.__doc__ = desc
    if hasattr(agent_tool, "tool_spec"):
        agent_tool.tool_spec["name"] = f"{tool_name}_tool"
    return agent_tool


def create_sync_agent_tool(
    agent_config: dict,
    tool_name: str,
    description: str | None = None,
) -> callable:
    """FALLBACK: Synchronous wrapper. For streaming=false or constrained runtimes."""
    desc = description or _derive_description(
        agent_config.get("description", ""),
        agent_config.get("system_prompt", ""),
    )

    @tool
    def agent_tool(query: str) -> str:
        agent = Agent(
            name=agent_config["name"],
            system_prompt=agent_config["system_prompt"],
            model=agent_config["model"],
            tools=agent_config["tools"],
            hooks=agent_config.get("hooks", []),
        )
        return str(agent(query))

    agent_tool.__doc__ = desc
    if hasattr(agent_tool, "tool_spec"):
        agent_tool.tool_spec["name"] = f"{tool_name}_tool"
    return agent_tool


def create_agent_tool(agent_config: dict, tool_name: str,
                      description: str | None = None,
                      streaming: bool = True) -> callable:
    """Factory: pick streaming or sync based on config."""
    if streaming:
        return create_streaming_agent_tool(agent_config, tool_name, description)
    return create_sync_agent_tool(agent_config, tool_name, description)


def _derive_description(description: str, system_prompt: str) -> str:
    """Description from agent description field, falling back to first sentence of prompt."""
    source = description or system_prompt
    if not source:
        return "Delegate to a specialist agent."
    period = source.find(".")
    if period > 0:
        first = source[:period + 1].strip()
    else:
        first = source.split("\n")[0].strip()
    if len(first) > AUTO_DESCRIPTION_MAX_CHARS:
        first = first[:AUTO_DESCRIPTION_MAX_CHARS - 3] + "..."
    return first
```

---

## 6. Governed Dynamic Tools

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

    Boundaries enforced:
    - Only tools from allowed_tool_names
    - Only models from allowed_models
    - Max sub-agents per request
    - Parent hooks inherited (logging, guardrails flow down)
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

---

## 7. Policy Compiler

Translates `tool_policy` into concrete hook attachments and tool filtering.

```python
# strands_platform/control_plane/compilation/policy_compiler.py

from ..parsing.ir import AgentIR, ToolPolicy, ToolRef, ToolRefKind, RiskLevel
from ...governance.tool_policy import (
    ApprovalRequiredHook, NoInternetFilter, ReadOnlyFilter, TenantScopeHook,
)
from ...errors import PolicyViolationError, GovernanceWarning


class PolicyCompiler:
    """Compile tool_policy into hooks and tool filters.

    Each policy maps to specific behaviors:
    - read_only: filter out tools with risk_level > read_only
    - safe_write: filter out destructive tools
    - approval_required: attach BeforeToolCallEvent interrupt hook
    - no_internet: filter out external tools and non-internal MCP
    - tenant_scoped: attach tenant isolation hook
    """

    def compile(self, agent_ir: AgentIR, tool_risk_levels: dict[str, RiskLevel],
                warnings: list) -> tuple[list, list[ToolRef]]:
        """Returns (hooks_to_add, filtered_tool_refs)."""
        if not agent_ir.tool_policy:
            return [], list(agent_ir.tool_refs)

        hooks = []
        filtered = list(agent_ir.tool_refs)
        policy = agent_ir.tool_policy

        if policy == ToolPolicy.READ_ONLY:
            filtered = [t for t in filtered
                        if self._risk(t, tool_risk_levels) == RiskLevel.READ_ONLY]
            hooks.append(ReadOnlyFilter())

        elif policy == ToolPolicy.SAFE_WRITE:
            filtered = [t for t in filtered
                        if self._risk(t, tool_risk_levels) != RiskLevel.DESTRUCTIVE]

        elif policy == ToolPolicy.APPROVAL_REQUIRED:
            hooks.append(ApprovalRequiredHook())
            # Check if any destructive tools lack approval hook
            for t in filtered:
                if self._risk(t, tool_risk_levels) == RiskLevel.DESTRUCTIVE:
                    warnings.append(GovernanceWarning(
                        "APPROVAL_REQUIRED",
                        f"Agent '{agent_ir.agent_id}' has destructive tool '{t.name}' "
                        f"with approval_required policy — human-in-the-loop enforced."
                    ))

        elif policy == ToolPolicy.NO_INTERNET:
            filtered = [t for t in filtered
                        if self._risk(t, tool_risk_levels) != RiskLevel.EXTERNAL
                        and t.kind != ToolRefKind.MCP]
            hooks.append(NoInternetFilter())

        elif policy == ToolPolicy.TENANT_SCOPED:
            hooks.append(TenantScopeHook())

        return hooks, filtered

    def _risk(self, ref: ToolRef, levels: dict[str, RiskLevel]) -> RiskLevel:
        return levels.get(ref.name, RiskLevel.SAFE_WRITE)
```

---

## 8. Execution Plane: Engine and Hydrator

The execution plane loads agent configs and produces live Strands objects at request time.

```python
# strands_platform/execution_plane/engine.py

from strands import Agent
from ..control_plane.registry.models import RegistryBackend
from .hydrator import Hydrator
from typing import Any
import time
import asyncio


# Request timeout — prevents runaway requests from holding resources
DEFAULT_REQUEST_TIMEOUT_SECONDS = 300


class AgentEngine:
    """The shared runtime engine. Loads agent configs from registry,
    hydrates into Strands Agents, executes.

    For file-based registry: agents compiled at startup, cached.
    For store-based registry: agents loaded per request, cached with TTL.

    Session IDs flow through to Agent invocation for conversation persistence.
    Request-level timeout prevents runaway execution.
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
        """Synchronous invoke with session support."""
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

```python
# strands_platform/execution_plane/hydrator.py

from strands import Agent
from strands.models.bedrock import BedrockModel
from .session import create_session_manager
from typing import Any


class Hydrator:
    """Turns an agent config dict into a live Strands Agent.

    Called per-request (for store backend) or once at startup (for file backend).
    The config dict contains ONLY serializable primitives: model params (not objects),
    tool names (not functions), hook names (not instances). Objects are rebuilt here.

    Returns (agent, session_id) tuple so the engine can pass session_id at invocation.
    """

    def __init__(self, tool_resolver, hook_resolver):
        self._tools = tool_resolver
        self._hooks = hook_resolver
        # Cache model providers — stateless, safe to reuse across requests
        self._model_cache: dict[str, Any] = {}

    def hydrate(self, config: dict, session_id: str | None = None) -> tuple[Agent, str | None]:
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

---

## 9. Execution Plane: Server and Streaming

```python
# strands_platform/execution_plane/server.py

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
        _check_agent_accessible(agent_id, engine)
        body = await request.json()
        result = engine.invoke(agent_id, body["message"], body.get("session_id"))
        return {"response": result, "agent": agent_id}

    @app.post("/agents/{agent_id}/stream")
    async def stream_agent(request: Request, agent_id: str = Path(...)):
        _check_agent_accessible(agent_id, engine)
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
        _check_agent_accessible(agent_id, engine, require_background=True)
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
                             require_background: bool = False):
    """Validate agent exists and isn't subagent_only."""
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

```python
# strands_platform/execution_plane/streaming.py

import json
from .tool_wrapper_types import SubAgentResult


def format_sse_event(event: dict) -> str | None:
    """Convert a Strands streaming event to SSE format.

    Returns None for events we don't forward to clients.
    """
    # Text output
    if "data" in event:
        return f"data: {json.dumps({'type': 'text', 'text': event['data']})}\n\n"

    # Tool usage
    if "current_tool_use" in event:
        tool = event["current_tool_use"]
        if tool.get("name"):
            return f"data: {json.dumps({'type': 'tool_call', 'tool': tool['name']})}\n\n"

    # Sub-agent streaming events
    if "tool_stream_event" in event:
        stream_data = event["tool_stream_event"].get("data")
        if isinstance(stream_data, SubAgentResult):
            inner = stream_data.event
            # Forward text from sub-agent
            if "data" in inner:
                return f"data: {json.dumps({'type': 'sub_agent_text', 'agent': stream_data.agent.name, 'text': inner['data']})}\n\n"
            if "current_tool_use" in inner:
                tool = inner["current_tool_use"]
                if tool.get("name"):
                    return f"data: {json.dumps({'type': 'sub_agent_tool', 'agent': stream_data.agent.name, 'tool': tool['name']})}\n\n"

    # Multi-agent events (Graph/Swarm)
    for event_type in ("multiagent_node_start", "multiagent_node_stream",
                       "multiagent_node_stop", "multiagent_handoff",
                       "multiagent_result"):
        if event_type in event or event.get("type") == event_type:
            return f"data: {json.dumps({'type': event_type, 'data': _safe_serialize(event)})}\n\n"

    # Final result
    if "result" in event:
        return f"data: {json.dumps({'type': 'result', 'data': str(event['result'])})}\n\n"

    return None


def _safe_serialize(obj) -> dict:
    """Best-effort serialization of event data for SSE."""
    if isinstance(obj, dict):
        return {k: str(v) for k, v in obj.items() if k != "result"}
    return {"raw": str(obj)}
```

---

## 10. Compilation Orchestrator

```python
# strands_platform/control_plane/compilation/compiler.py

from ..parsing.ir import PlatformIR, WorkflowType, ExposureMode
from ..resolution.tool_registry import ToolRegistry
from ..resolution.hook_registry import HookRegistry
from ..resolution.skill_resolver import SkillResolver
from ..resolution.sop_resolver import SOPResolver
from .dependency_graph import build_dependency_graph, topological_sort
from .agent_compiler import AgentCompiler
from .graph_compiler import GraphCompiler
from .swarm_compiler import SwarmCompiler
from .workflow_compiler import WorkflowCompiler
from .policy_compiler import PolicyCompiler
from ...errors import GovernanceWarning
from ...execution_plane.engine import AgentEngine
from ...execution_plane.hydrator import Hydrator
from ..registry.file_backend import FileRegistryBackend

from typing import Any


class Platform:
    """Compiled output. The shared runtime ready to serve."""

    def __init__(self, agents, workflows, registry, ir, warnings, engine):
        self.agents = agents
        self.workflows = workflows
        self.registry = registry
        self.warnings = warnings
        self._ir = ir
        self._engine = engine

    def get(self, name: str):
        if name in self.agents:
            return self.agents[name]
        if name in self.workflows:
            return self.workflows[name]
        available = sorted(list(self.agents) + list(self.workflows))
        raise KeyError(f"'{name}' not found. Available: {', '.join(available)}")

    def serve(self, port: int = 8080, host: str = "0.0.0.0"):
        from ...execution_plane.server import create_app
        import uvicorn
        app = create_app(self._engine, self._ir)
        uvicorn.run(app, host=host, port=port)


class Compiler:
    """Full compilation pipeline. Receives PlatformIR, produces Platform."""

    def __init__(self, tool_registry: ToolRegistry, hook_registry: HookRegistry,
                 dry_run: bool = False):
        self._tools = tool_registry
        self._hooks = hook_registry
        self._dry_run = dry_run
        self._warnings: list[GovernanceWarning] = []

    def compile(self, ir: PlatformIR) -> Platform:
        # Phase: Dependency graph + cycle detection + topo sort
        deps = build_dependency_graph(ir)
        compile_order = topological_sort(deps)

        # Resolve global hooks
        global_hooks = [self._hooks.resolve(h, "global") for h in ir.global_hooks]

        # Build resolvers
        skill_resolver = SkillResolver(self._tools, self._warnings)
        sop_resolver = SOPResolver(self._warnings)
        policy_compiler = PolicyCompiler()

        # Compile agents and workflows in dependency order
        compiled_agents: dict[str, Any] = {}
        compiled_configs: dict[str, dict] = {}  # Agent config dicts for hydration
        compiled_workflows: dict[str, Any] = {}
        tool_cache: dict[str, Any] = {}

        agent_compiler = AgentCompiler(
            tool_resolver=self._tools, hook_registry=self._hooks,
            skill_resolver=skill_resolver, sop_resolver=sop_resolver,
            policy_compiler=policy_compiler,
            compiled_agents=compiled_agents, compiled_configs=compiled_configs,
            compiled_workflows=compiled_workflows, tool_cache=tool_cache,
            global_hooks=global_hooks, guardrails=ir.guardrails,
            warnings=self._warnings,
        )

        for name in compile_order:
            if name in ir.agents:
                agent, config = agent_compiler.compile(ir.agents[name])
                compiled_agents[name] = agent
                compiled_configs[name] = config

            elif name in ir.workflows:
                wf_ir = ir.workflows[name]
                if wf_ir.workflow_type == WorkflowType.GRAPH:
                    compiled_workflows[name] = GraphCompiler(
                        compiled_agents, compiled_workflows,
                        self._hooks, self._warnings
                    ).compile(wf_ir)
                elif wf_ir.workflow_type == WorkflowType.SWARM:
                    compiled_workflows[name] = SwarmCompiler(
                        compiled_agents, self._hooks, self._warnings
                    ).compile(wf_ir)
                elif wf_ir.workflow_type == WorkflowType.WORKFLOW:
                    compiled_workflows[name] = WorkflowCompiler(
                        compiled_agents, self._hooks, self._warnings
                    ).compile(wf_ir)

        # Governance warnings
        if ir.guardrails.bedrock_guardrail_id is None and ir.deployment_target != "local":
            self._warnings.append(GovernanceWarning(
                "NO_GUARDRAILS",
                "No Bedrock Guardrails configured. Recommended for production."
            ))

        # Build engine
        registry = FileRegistryBackend(".")  # Default; overridden by serve/deploy
        hydrator = Hydrator(self._tools, self._hooks)
        engine = AgentEngine(registry, hydrator)

        return Platform(
            agents=compiled_agents,
            workflows=compiled_workflows,
            registry=registry,
            ir=ir,
            warnings=self._warnings,
            engine=engine,
        )
```

---

## 11. Dependency Graph

Unchanged from previous LLD — Kahn's algorithm with alphabetical tie-breaking. Handles agents, workflows, and cross-references. Detects cycles. O(V+E).

Reference: `strands_platform/control_plane/compilation/dependency_graph.py` — same implementation as before, extended to include `WorkflowType.WORKFLOW` task agent refs in the dependency set.

---

## 12. Sequence Diagrams

### 12.1 Compilation (Control Plane)

```
CLI                    Parser         Compiler        AgentCompiler     PolicyCompiler
 │                       │               │                │                 │
 │ compile config/       │               │                │                 │
 │──────────────────────>│               │                │                 │
 │                       │ load files    │                │                 │
 │                       │ resolve ${ENV}│                │                 │
 │                       │ validate schema                │                 │
 │            PlatformIR │               │                │                 │
 │<──────────────────────│               │                │                 │
 │                       │               │                │                 │
 │ compile(ir)           │               │                │                 │
 │──────────────────────────────────────>│                │                 │
 │                       │               │ dep graph      │                 │
 │                       │               │ topo sort      │                 │
 │                       │               │                │                 │
 │                       │               │ for each entity (leaves first):  │
 │                       │               │                │                 │
 │                       │               │ compile(agent) │                 │
 │                       │               │───────────────>│                 │
 │                       │               │                │ resolve SOP/prompt
 │                       │               │                │ discover skills │
 │                       │               │                │ resolve tools   │
 │                       │               │                │ compile policy  │
 │                       │               │                │────────────────>│
 │                       │               │                │  filter tools   │
 │                       │               │                │  add hooks      │
 │                       │               │                │<────────────────│
 │                       │               │                │ build agent_config
 │                       │               │                │ create @tool wrapper
 │                       │               │ (agent, config)│                 │
 │                       │               │<───────────────│                 │
 │                       │               │                │                 │
 │              Platform │               │                │                 │
 │<──────────────────────────────────────│                │                 │
```

### 12.2 Request (Execution Plane, Streaming)

```
Client             Server          Engine          Hydrator         Strands Agent
  │                  │                │                │                │
  │ POST /agents/    │                │                │                │
  │   lead/stream    │                │                │                │
  │─────────────────>│                │                │                │
  │                  │ invoke_stream  │                │                │
  │                  │───────────────>│                │                │
  │                  │                │ get_agent()    │                │
  │                  │                │ (from cache    │                │
  │                  │                │  or registry)  │                │
  │                  │                │                │                │
  │                  │                │ hydrate(config)│                │
  │                  │                │───────────────>│                │
  │                  │                │                │ Agent(...)     │
  │                  │                │         agent  │                │
  │                  │                │<───────────────│                │
  │                  │                │                │                │
  │                  │                │ stream_async(msg)               │
  │                  │                │───────────────────────────────>│
  │                  │                │                │               │
  │  SSE: text       │  format_sse   │     event: data                │
  │<─────────────────│<──────────────│<───────────────────────────────│
  │                  │                │                │               │
  │  SSE: tool_call  │  format_sse   │     event: tool_use            │
  │<─────────────────│<──────────────│<───────────────────────────────│
  │                  │                │                │               │
  │  SSE: sub_agent  │  format_sse   │     event: tool_stream (SubAgentResult)
  │<─────────────────│<──────────────│<───────────────────────────────│
  │                  │                │                │               │
  │  SSE: result     │  format_sse   │     event: result              │
  │<─────────────────│<──────────────│<───────────────────────────────│
```

### 12.3 Multi-Agent Workflow Streaming

```
Client             Server          Engine          Graph              Agent Nodes
  │                  │                │                │                │
  │ POST /workflows/ │                │                │                │
  │ pipeline/stream  │                │                │                │
  │─────────────────>│                │                │                │
  │                  │                │ stream_async   │                │
  │                  │                │───────────────>│                │
  │                  │                │                │                │
  │ SSE: node_start  │                │   multiagent_node_start        │
  │ (node: classify) │                │   (node_id: classify)          │
  │<─────────────────│<───────────────│<───────────────│                │
  │                  │                │                │ Agent.stream   │
  │ SSE: node_stream │                │   multiagent_node_stream       │
  │ (text from       │                │   (nested agent events)        │
  │  classifier)     │                │                │                │
  │<─────────────────│<───────────────│<───────────────│<───────────────│
  │                  │                │                │                │
  │ SSE: handoff     │                │   multiagent_handoff           │
  │ (classify→billing)               │   (from→to)    │                │
  │<─────────────────│<───────────────│<───────────────│                │
  │                  │                │                │                │
  │ SSE: node_start  │                │   multiagent_node_start        │
  │ (node: billing)  │                │   (node_id: billing)           │
  │<─────────────────│<───────────────│<───────────────│                │
  │  ...             │                │                │                │
  │                  │                │                │                │
  │ SSE: result      │                │   multiagent_result            │
  │<─────────────────│<───────────────│<───────────────│                │
```

---

## 13. Concurrency Model

**Compilation**: Single-threaded, deterministic. Alphabetical topo sort for reproducibility.

**Runtime — Strands controls it**: The platform introduces zero concurrency primitives. Strands 1.0 manages async agent loops, parallel tool execution, and swarm parallelism. Our `@tool async def` wrappers use `stream_async()` which runs on Strands' asyncio event loop.

**Server — FastAPI + Uvicorn**: Each HTTP request runs on the asyncio event loop. Agent invocations are async. The `Agent()` constructor is called per-request inside tool wrappers and by the hydrator, providing state isolation. Model provider objects (BedrockModel, etc.) are thread-safe and shared.

**Background agents**: Fire-and-forget via `asyncio.create_task`. No separate process.

**Store-based registry**: DynamoDB reads are async-safe. Agent config cache has a TTL (default 60s) — stale reads are bounded.

---

## 14. Caching Strategy

| What | Key | Lifetime | Thread-safe |
|---|---|---|---|
| Tool wrappers (`@tool` functions) | agent_id | Process lifetime | Yes — closures over immutable config |
| Registry agent configs | `agent_id:version_id` | TTL (60s default) | Yes — dict read is atomic in CPython |
| Skill discovery results | skills directory path | Process lifetime | Yes — frozen data |
| Model provider objects | `provider:model_id:region` | Process lifetime | Yes — stateless |

**NOT cached**: LLM responses (that's the model provider's job), session state (that's the session manager's job), MCP server connections (managed by Strands MCPClient lifecycle).

---

## 15. Design Decisions

### D1: Agents are definitions, not deployments
Creating an agent creates a registry record and a versioned config. It does not create a new runtime. The shared engine loads configs and produces Strands Agents at request time. This is the core platform principle from the architecture document.

### D2: Two registry backends, same compilation pipeline
File-based for dev (compiled at startup, no persistence). Store-based for production (loaded per request, versioned, rollback-capable). The compiler produces the same IR and the same Strands objects regardless of backend. The difference is when hydration happens.

### D3: Async streaming @tool wrappers by default
Matches the Strands docs sub-agent streaming pattern exactly. Fresh `Agent()` per invocation inside the tool body. `stream_async()` with `SubAgentResult` yielding. Synchronous fallback via `streaming: false`.

### D4: Per-invocation Agent instantiation
The compiler resolves config (prompt, model params, tools, hooks) at compile time. The `Agent()` constructor runs inside the `@tool` body at each invocation. This gives per-request state isolation without sacrificing compile-time validation.

### D5: Governance flows downward
Dynamic sub-agents inherit parent hooks. Bedrock Guardrails inherit. Tool policies inherit. If the parent is governed, its children are governed. Enforced by governed tool wrappers, not by Strands natively.

### D6: Policy compiles to hooks + filters
`tool_policy: "approval_required"` doesn't add a custom runtime check. It attaches a standard `BeforeToolCallEvent` hook that uses Strands' interrupt mechanism. `no_internet` doesn't add a firewall. It filters out external tools at compile time. Policies are compile-time decisions expressed through standard Strands primitives.

### D7: Exposure mode determines endpoint generation
`subagent_only` → no routes. `background` → async trigger endpoint. `chat` → SSE streaming + UI config metadata. The server doesn't guess — it reads the exposure mode and generates exactly the right endpoints.

### D8: SSE for all streaming, multi-agent events included
Every Strands event type (text, tool_use, tool_stream_event, multiagent_node_start/stream/stop/handoff/result) is forwarded to clients via SSE. The streaming module is a formatter, not a filter. Clients decide what to render.

### D9: Frozen IR, mutable Platform
The IR is immutable after parsing — a snapshot of what the user wrote. The Platform is mutable at runtime (cache, sessions, etc.). You can always diff IR against Platform to see what the compiler changed.

### D10: No platform wrapper classes at Strands boundary
`platform.agents["lead"]` returns a real `strands.Agent`. `platform.workflows["pipeline"]` returns a real `Graph` or `Swarm`. No subclasses, no proxies. If Strands adds a feature tomorrow, it works immediately.
