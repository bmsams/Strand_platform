# Strands Agent Platform — Requirements & Architecture

> **Version**: 2.0 — Full rewrite
> **Methodology**: EARS (Easy Approach to Requirements Syntax)
> **Target SDK**: Strands Agents SDK 1.x (Python + TypeScript)
> **Standards**: AgentSkills.io, Agent SOPs (RFC 2119), A2A Protocol, MCP

---

## 1. What This Is

An agent platform — not a single agent app. One shared runtime. Many specialized agents. Each agent optionally exposed through its own UI, API, or workflow. All built on the same execution model.

The platform answers six questions for any agent:

1. **What is it?** — Identity: name, purpose, version
2. **What can it do?** — Capabilities: tools, skills, sub-agents
3. **What is it allowed to do?** — Governance: policies, auth, hooks, guardrails
4. **How should it behave?** — Instructions: SOP, system prompt, output schema
5. **How is it exposed?** — Experience: chat, API, form, embedded, subagent-only, background
6. **How is it observed?** — Ops: streaming, traces, cost, logs, evaluations

---

## 2. Core Principle

Do not build every agent as its own custom-coded system.

Build:
- One agent runtime (the Strands execution engine)
- One platform control plane (registry, versioning, publishing)
- One config-driven agent model (JSON/YAML/Markdown definitions)
- Many deployable agent experiences (different frontends, same engine)

A "security reviewer," "BOM builder," "schema crawler," or "Cloudscape prototype builder" are not separate products at the runtime layer. They are different configs, different tools, different skills, different policies, different UI wrappers, different endpoint exposures. Same engine underneath.

**Agents are definitions, not deployments.**

Creating an agent creates: a registry record, a versioned config, an optional route, an optional UI binding. It does NOT create: a new backend codebase, a separate runtime stack, a custom orchestrator. Only do separate deployments when an agent truly needs unique compute or isolation.

---

## 3. Platform Architecture

### 3.1 Three Layers

```
┌──────────────────────────────────────────────────────────────┐
│  Layer 3: EXPERIENCE LAYER                                    │
│  How people consume the agent                                 │
│                                                               │
│  Chat shell · Form shell · Workflow shell · Console shell     │
│  REST endpoint · Webhook · Scheduled job · VS Code extension  │
│  Embedded widget · Admin page · Subagent-only (no UI)         │
│                                                               │
│  Same underlying runtime, different front doors               │
├──────────────────────────────────────────────────────────────┤
│  Layer 2: AGENT DEFINITION LAYER                              │
│  What makes each agent different                              │
│                                                               │
│  name · purpose · instructions · system_prompt / SOP          │
│  allowed tools · allowed skills · hooks/policies              │
│  model defaults · memory behavior · output schema             │
│  exposure mode · UI hints · deployment options                │
│                                                               │
│  This is where specialization lives                           │
├──────────────────────────────────────────────────────────────┤
│  Layer 1: RUNTIME LAYER                                       │
│  The shared execution engine                                  │
│                                                               │
│  model calls · tool calling · skills · hooks · memory         │
│  streaming · sub-agents · observability · auth context         │
│  retries · approvals · guardrails · cost tracking             │
│                                                               │
│  Stays the same for every agent                               │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 Control Plane vs Execution Plane

**Control Plane** — the builder side (Agent Builder Studio):
- Create agent
- Edit config
- Validate config
- Version agent
- Publish agent
- Assign tools, skills, hooks
- Generate endpoint/UI bindings
- Set auth and tenant policy

**Execution Plane** — the runtime side (shared agent engine):
- Receive requests
- Load the published agent version
- Assemble context (prompt + skills Phase 1 metadata + SOP parameters)
- Execute tools / sub-agents (async streaming via Strands `stream_async`)
- Stream results (SSE to clients, multi-agent events)
- Log traces (OTEL → CloudWatch/Jaeger)
- Enforce policy (guardrails, data governance, rate limits)

### 3.3 Mapping to Strands SDK

The runtime layer IS the Strands SDK. We don't wrap it, re-implement it, or abstract over it:

| Runtime Capability | Strands SDK API |
|---|---|
| Model calls | `Agent(model=BedrockModel(...))` |
| Tool calling | `Agent(tools=[...])`, `@tool` decorator |
| Skills | `agentskills.discover_skills()`, progressive disclosure |
| Hooks | `HookProvider`, `HookRegistry`, lifecycle events |
| Memory | Strands session managers (DynamoDB, S3, AgentCore Memory) |
| Streaming | `agent.stream_async()`, `tool_stream_event`, multi-agent events |
| Sub-agents (static) | `@tool async def` + `Agent()` per invocation + `yield SubAgentResult` |
| Sub-agents (dynamic) | `strands_tools.use_agent`, `strands_tools.swarm` |
| Observability | Strands OTEL instrumentation, AgentCore Observability |
| Auth context | AgentCore Identity (OAuth, Cognito, IAM) |
| Retries | `AfterModelCallEvent.retry` via hooks |
| Approvals | `BeforeNodeCallEvent.interrupt()` in graph hooks |
| Guardrails | Bedrock Guardrails + `BeforeToolCallEvent` hooks |
| Multi-agent orchestration | `GraphBuilder`, `Swarm`, `Workflow` tool, Handoffs, A2A |

---

## 4. Object Model

### 4.1 Core Entities

**AgentDefinition** — what the agent is:

```
agent_id            unique identifier
name                human-readable name
description         purpose description (used in agent-as-tool docstrings)
instructions        system_prompt string OR sop reference
model_profile       model provider + model_id + region + fallback chain
tool_policy         allowed tools list + risk levels
skill_refs          skills directory + pattern + filter
hook_refs           hooks to attach (logging, guardrails, custom)
memory_policy       session manager config + tenant scoping
output_schema       structured output schema (optional)
exposure_mode       chat | api | form | embedded | subagent_only | background
ui_template         UI shell config (features, layout hints)
dynamic_capabilities  governed runtime sub-agent/tool creation
published_version   current live version ID
```

**AgentVersion** — immutable published config:

```
version_id          unique, auto-generated
agent_id            parent agent
config_blob         complete frozen agent config (IR snapshot)
created_by          who published this version
created_at          timestamp
status              draft | published | deprecated | rolled_back
```

**ToolDefinition** — reusable tools:

```
tool_id             unique identifier
name                tool name (matches @tool function name)
description         what it does (LLM reads this to decide when to call)
schema              input/output JSON schema
scope               global | team | agent-specific
risk_level          read_only | safe_write | destructive | external
source              builtin | custom | mcp
```

**SkillDefinition** — AgentSkills.io packages:

```
skill_id            unique, derived from SKILL.md name field
name                kebab-case, max 64 chars
description         max 1024 chars (Phase 1 discovery)
directory           path to skill directory
version             from metadata
```

**EndpointBinding** — how the agent is exposed:

```
binding_id          unique identifier
agent_id            which agent
route               URL path (e.g., /agents/cloudscape-builder)
auth_policy         cognito | iam | oauth | api_key | none
rate_limit_policy   requests per minute/hour
ui_mode             chat | form | workflow | console | dashboard | none
ui_features         file_upload, streaming, artifact_panel, schema_form, etc.
```

**Session** — runtime conversation/execution state:

```
session_id          unique per conversation
agent_id            which agent
agent_version_id    which version was active
user_id             who is talking
messages            conversation history
state               agent state dict
created_at          timestamp
ttl                 expiration
```

**Trace** — observability record:

```
trace_id            OTEL trace ID
agent_id            which agent
session_id          parent session
spans               model calls, tool calls, sub-agent calls
duration_ms         total execution time
token_usage         input/output token counts
cost                estimated cost
guardrail_hits      any guardrail activations
```

### 4.2 Entity Relationships

```
AgentDefinition  1──N  AgentVersion     (versioned configs)
AgentDefinition  1──N  EndpointBinding  (multiple exposure modes)
AgentDefinition  N──N  ToolDefinition   (many agents share tools)
AgentDefinition  N──N  SkillDefinition  (many agents share skills)
AgentDefinition  1──N  Session          (runtime conversations)
Session          1──N  Trace            (execution records)
```

---

## 5. Functional Requirements

### 5.1 Agent Definition (EARS)

**FR-5.1.1** — The platform shall accept agent definitions in JSON, YAML, and Markdown configuration formats.

**FR-5.1.2** — Each agent definition shall support the fields defined in the AgentDefinition object model (Section 4.1).

**FR-5.1.3** — WHEN an agent definition specifies `instructions` as a string, the compiler shall use it as the Strands Agent `system_prompt`.

**FR-5.1.4** — WHEN an agent definition specifies `instructions` as an SOP reference (`{"sop": {"file": "...", "parameters": {...}}}`), the compiler shall load the `.sop.md` file, substitute parameters, and use the result as the Strands Agent `system_prompt`.

**FR-5.1.5** — IF an SOP defines a parameter as `(required)` and it is not provided, THEN the compiler shall reject the configuration with an error identifying the missing parameter.

**FR-5.1.6** — WHEN a `defaults` block is present, the compiler shall apply its values (model, hooks, memory, policies) to every agent that does not explicitly override those fields.

**FR-5.1.7** — WHEN an agent definition specifies `output_schema`, the compiler shall configure Strands structured output for that agent.

### 5.2 Agent-as-Tool (Sub-Agent Composition)

**FR-5.2.1** — WHEN an agent's tools list contains `{"agent": "<name>"}`, the compiler shall generate an async generator `@tool` function following the Strands streaming sub-agent pattern: fresh `Agent()` per invocation, `stream_async()` iteration, `SubAgentResult` event yielding.

**FR-5.2.2** — The generated wrapper shall instantiate `Agent()` inside each tool invocation using pre-resolved config (prompt, model, tools, hooks) — not close over a pre-built Agent instance. This provides per-request state isolation.

**FR-5.2.3** — The generated wrapper shall yield streaming events so the parent agent receives them as `tool_stream_event` data, enabling real-time streaming to clients.

**FR-5.2.4** — The wrapper's docstring shall be derived from the sub-agent's `description` field, falling back to the first sentence of its instructions (truncated to 200 characters).

**FR-5.2.5** — WHEN an agent-as-tool reference includes a `description` override, the compiler shall use that instead.

**FR-5.2.6** — WHEN an agent-as-tool reference includes `"streaming": false`, the compiler shall generate a synchronous `@tool` returning only the final string result.

**FR-5.2.7** — WHEN an agent's `exposure_mode` is `"subagent_only"`, the platform shall NOT generate any endpoint binding for it — it is only callable as a tool by other agents.

**FR-5.2.8** — IF a sub-agent throws an exception during execution within a tool wrapper, THEN the wrapper shall catch the exception, yield an error event containing the sub-agent's identity and a summary of the exception, and return a structured error string that the parent agent can reason about — rather than propagating the exception and crashing the parent.

### 5.3 Tool Resolution

**FR-5.3.1** — The platform shall resolve tool references in this priority order:
1. Built-in Strands tools (`strands_tools` package)
2. Custom registered tools (Python `@tool` functions in tool registry)
3. Agent references (`{"agent": "<name>"}`)
4. Workflow references (`{"workflow": "<name>"}`)
5. MCP server references (`{"mcp": "<url_or_name>"}`)
6. Handoff references (`{"handoff": "user"}`)

**FR-5.3.2** — IF a tool name exists in both built-in and custom registries, THEN the compiler shall reject the configuration with an error identifying the conflict.

**FR-5.3.3** — IF a tool reference cannot be resolved, THEN the compiler shall reject with an error listing the unresolved name, the agent that requires it, and available alternatives.

**FR-5.3.4** — Each tool in the ToolDefinition registry shall include a `risk_level` field. WHEN a tool has `risk_level: "destructive"`, the platform shall log a warning if attached to an agent without an approval hook.

### 5.4 MCP Server Integration

**FR-5.4.1** — WHEN a tool reference is `{"mcp": "<url>"}` with an HTTP URL, the compiler shall create an inline MCP server connection via SSE transport.

**FR-5.4.2** — WHEN a tool reference is `{"mcp": "<name>"}` without an HTTP prefix, the compiler shall resolve against the `mcp_servers` config block.

**FR-5.4.3** — Named MCP server definitions shall support: `url`, `command`, `args`, `env` (with `${VAR}` interpolation), `transport` (sse|stdio).

**FR-5.4.4** — IF an `${VAR}` environment variable is not set, THEN the compiler shall reject with an error identifying the missing variable.

**FR-5.4.5** — The compiler shall produce MCP connections using Strands' native `MCPClient`.

### 5.5 Agent Skills (AgentSkills.io)

**FR-5.5.1** — The platform shall support the AgentSkills.io open standard: skill directories with SKILL.md (YAML frontmatter + markdown), optional `scripts/`, `references/`, `assets/`.

**FR-5.5.2** — The compiler shall implement progressive disclosure:
- Phase 1 (startup): inject skill metadata (~100 tokens/skill) into system prompt via `generate_skills_prompt()`
- Phase 2 (activation): load full SKILL.md when skill is triggered
- Phase 3 (execution): load resource files on demand

**FR-5.5.3** — Agent skill config shall specify `pattern`: `"file"` (LLM reads via `file_read`), `"tool"` (explicit `skill()` tool), or `"meta-tool"` (isolated sub-agent per skill via `use_skill()`).

**FR-5.5.4** — WHERE `skills.filter` is specified, only named skills shall be available.

**FR-5.5.5** — WHERE `skills.additional_tools` is specified on meta-tool pattern, those tools shall be provided to skill sub-agents.

**FR-5.5.6** — WHEN the compiler discovers a skill, it shall validate against AgentSkills.io spec: name (kebab-case, max 64 chars), description (max 1024 chars), directory name matches name field, no path traversal in resource references, resource files under 10MB.

**FR-5.5.7** — IF a skill fails validation, THEN the compiler shall log a warning and skip it rather than halting.

### 5.6 Agent SOPs

**FR-5.6.1** — The platform shall support Strands Agent SOPs (`.sop.md`) as structured instruction sets with RFC 2119 constraints (MUST, SHOULD, MAY), parameters, and explicit steps.

**FR-5.6.2** — SOP content with parameters substituted shall become the agent's `system_prompt`.

**FR-5.6.3** — WHERE the SOP-to-Skill conversion feature is enabled, the compiler shall convert SOPs to AgentSkills.io format via `strands-agents-sops skills`.

### 5.7 Multi-Agent Workflows

#### 5.7.1 Graph (Deterministic Routing)

**FR-5.7.1** — WHEN a workflow has `"type": "graph"`, the compiler shall produce a Strands `GraphBuilder` → `build()` graph.

**FR-5.7.2** — Graph config shall support: `nodes` (agent or workflow refs keyed by ID), `edges` (with `from`, `to`, optional `condition`), `entry_point`, optional `max_node_executions`, optional `execution_timeout`.

**FR-5.7.3** — Edge `condition` values shall be dotted Python paths resolved via `importlib` to callables.

**FR-5.7.4** — Graph nodes shall support agents, swarms, other graphs, and A2A remote agents (`{"a2a": "<endpoint>"}`).

**FR-5.7.5** — WHEN `max_node_executions` is set, the compiler shall call `builder.set_max_node_executions()`. WHEN `execution_timeout` is set, the compiler shall call `builder.set_execution_timeout()`.

#### 5.7.2 Swarm (Autonomous Collaboration)

**FR-5.7.6** — WHEN a workflow has `"type": "swarm"`, the compiler shall produce a Strands `Swarm`.

**FR-5.7.7** — Swarm config shall support: `agents` (array of agent names), `coordination` (collaborative|competitive|hybrid).

#### 5.7.3 Workflow (DAG Pipeline)

**FR-5.7.8** — WHEN a workflow has `"type": "workflow"`, the compiler shall produce a configuration using the Strands `workflow` tool with pre-defined tasks and dependencies.

**FR-5.7.9** — Each task shall support: `task_id`, `agent` reference, `description`, `depends_on` (array of task IDs), `priority`.

**FR-5.7.10** — Tasks without `depends_on` shall be eligible for parallel execution.

#### 5.7.4 Handoff

**FR-5.7.11** — WHEN an agent's tools contain `{"handoff": "user"}`, the compiler shall include Strands' `handoff_to_user` tool.

#### 5.7.5 Composition

**FR-5.7.12** — A graph node shall accept a swarm or workflow as a child. A workflow-as-tool or swarm-as-tool reference shall compile to `@tool async def` wrappers identical to agent-as-tool.

### 5.8 Dynamic Capabilities (Runtime Sub-Agent Creation)

**FR-5.8.1** — WHEN `dynamic_capabilities.use_agent.enabled` is true, the compiler shall provide a governed `use_agent` tool constrained by `allowed_tools`, `allowed_models`, `max_sub_agents_per_request`.

**FR-5.8.2** — WHEN `inherit_hooks` is true, dynamically created sub-agents shall receive the parent's hooks (including guardrails).

**FR-5.8.3** — IF a sub-agent requests a tool or model not in the allowed list, THEN the governed tool shall return an error listing alternatives.

**FR-5.8.4** — IF `max_sub_agents_per_request` is exceeded, THEN the governed tool shall return an error and refuse further creation.

**FR-5.8.5** — WHEN `dynamic_capabilities.swarm.enabled` is true, the compiler shall provide a governed `swarm` tool constrained by `max_swarm_size` and `allowed_patterns`.

**FR-5.8.6** — WHEN `dynamic_capabilities.workflow.enabled` is true, the compiler shall provide a governed `workflow` tool constrained by `max_tasks` and `allowed_tools`.

**FR-5.8.7** — WHEN `dynamic_capabilities.load_tool.enabled` is true, the compiler shall include `load_tool` from `strands_tools`.

**FR-5.8.8** — WHEN `dynamic_capabilities.dynamic_mcp.enabled` is true, the compiler shall include the Dynamic MCP Client and emit a security warning.

**FR-5.8.9** — IF `dynamic_mcp` is enabled in production, THEN the compiler shall reject unless `acknowledge_risk: true` is set.

**FR-5.8.10** — IF an agent has both a `tool_policy` and `dynamic_capabilities`, THEN the compiler shall validate that `allowed_tools` in `dynamic_capabilities` is a subset of the tools that survive policy filtering. IF `tool_policy` is `no_internet` and `dynamic_capabilities.use_agent.allowed_tools` includes `web_search`, the compiler shall reject the configuration with an error explaining the contradiction.

### 5.9 Hooks

**FR-5.9.1** — The platform shall support Strands `HookProvider` attachment at three levels: global (all agents), per-agent, per-workflow (orchestrator level).

**FR-5.9.2** — WHILE both global and agent-level hooks are defined, the compiler shall merge them additively.

**FR-5.9.3** — The platform shall support these stable Strands hook events: `BeforeInvocationEvent`, `AfterInvocationEvent`, `BeforeModelCallEvent`, `AfterModelCallEvent`, `BeforeToolCallEvent`, `AfterToolCallEvent`, `BeforeNodeCallEvent`, `AfterNodeCallEvent`.

**FR-5.9.4** — IF a hook subscribes to events not in the installed SDK, THEN the compiler shall emit a warning.

### 5.10 Hot Reload

**FR-5.10.1** — WHEN `hot_reload.enabled` is true, the compiler shall set `Agent(load_tools_from_directory=True)`.

**FR-5.10.2** — IF hot reload is enabled and the deployment target is not in `allowed_in`, THEN the compiler shall disable it and emit a warning.

---

## 6. Exposure & Experience Requirements

### 6.1 Exposure Modes

**FR-6.1.1** — Each agent definition shall specify an `exposure_mode`:

| Mode | Behavior |
|---|---|
| `chat` | SSE streaming endpoint + chat UI shell |
| `api` | Request-response + SSE streaming endpoints, no UI |
| `form` | Schema-driven form UI + structured output |
| `embedded` | Widget endpoint consumable by other apps |
| `subagent_only` | No endpoint, only callable as tool by other agents |
| `background` | Invocable by schedule or event trigger, no interactive endpoint |

**FR-6.1.2** — WHEN `exposure_mode` is `"subagent_only"`, the platform shall NOT generate any endpoint binding.

**FR-6.1.3** — WHEN `exposure_mode` is `"background"`, the platform shall generate an async invocation endpoint (fire-and-forget) and support scheduled triggers via EventBridge or cron.

### 6.2 Endpoint Bindings

**FR-6.2.1** — Each exposed agent shall have an EndpointBinding with: `route`, `auth_policy`, `rate_limit_policy`, `ui_mode`.

**FR-6.2.2** — One agent may have multiple endpoint bindings (e.g., internal API + public chat + embedded widget).

**FR-6.2.3** — Routes shall follow the pattern `/agents/{agent_id}` with optional version pinning `/agents/{agent_id}/v/{version_id}`.

### 6.3 Streaming Endpoints

**FR-6.3.1** — The platform shall expose two endpoint variants per exposed agent: `POST /agents/{id}` (request-response) and `POST /agents/{id}/stream` (SSE).

**FR-6.3.2** — The SSE endpoint shall forward all Strands streaming events: `data` (text), `current_tool_use`, `tool_stream_event` (sub-agent events), `result`.

**FR-6.3.3** — The SSE endpoint for workflows shall forward multi-agent events: `multiagent_node_start`, `multiagent_node_stream`, `multiagent_node_stop`, `multiagent_handoff`, `multiagent_result`.

### 6.4 UI Shell Configuration

**FR-6.4.1** — Agent config shall support `ui_template` with `ui_mode` and `ui_features`:

```json
{
  "ui_template": {
    "ui_mode": "chat",
    "ui_features": {
      "file_upload": true,
      "streaming": true,
      "artifact_panel": true,
      "markdown_rendering": true
    }
  }
}
```

**FR-6.4.2** — The platform shall expose UI configuration metadata at `GET /agents/{id}/ui-config` so frontend shells can self-configure.

**FR-6.4.3** — Supported UI modes: `chat` (conversational), `form` (schema-driven input/output), `workflow` (multi-step wizard), `console` (streaming terminal), `dashboard` (metrics/traces view).

---

## 7. Governance Requirements (AWS Golden Pathway)

### 7.1 Bedrock Guardrails

**FR-7.1.1** — WHERE `guardrails.bedrock_guardrail_id` is specified, the compiler shall configure Bedrock Guardrails on all model invocations in scope.

**FR-7.1.2** — WHEN a response is blocked by guardrails, the runtime shall return a safe fallback and log the activation.

**FR-7.1.3** — IF no guardrails are configured for a production deployment, THEN the compiler shall emit a warning.

### 7.2 Policies

**FR-7.2.1** — Each agent shall support a `tool_policy` specifying access levels:

| Policy | Behavior |
|---|---|
| `read_only` | Agent can only use tools marked as read-safe |
| `safe_write` | Agent can use read + write tools, no destructive |
| `approval_required` | Write/destructive tools require human approval via interrupt hook |
| `no_internet` | Block tools that make external network calls |
| `tenant_scoped` | Memory and data access scoped to authenticated tenant |

**FR-7.2.2** — WHEN `tool_policy` is `"approval_required"`, the compiler shall attach a `BeforeToolCallEvent` hook that interrupts on write/destructive tools.

**FR-7.2.3** — WHEN `tool_policy` is `"no_internet"`, the compiler shall exclude tools with `risk_level: "external"` and block MCP servers pointing to non-internal domains.

### 7.3 Cost Management

**FR-7.3.1** — WHERE `cost.allocation_tags` is specified, the compiler shall apply tags to all Bedrock API calls for cost attribution.

**FR-7.3.2** — WHEN per-agent `max_tokens_per_request` is set, the runtime shall terminate execution when exceeded.

**FR-7.3.3** — WHEN `cost.enable_prompt_caching` is true, the compiler shall enable Bedrock prompt caching on supported models.

### 7.4 Data Governance

**FR-7.4.1** — WHEN `data_governance.prohibited_data_patterns` is specified, the platform shall include a HookProvider that scans tool I/O for matches and blocks or redacts.

**FR-7.4.2** — WHEN `data_governance.data_residency` is specified, the compiler shall verify all model regions and MCP endpoints are within the zone.

**FR-7.4.3** — IF a resource is outside the residency zone, THEN the compiler shall reject with an error.

### 7.5 Access Control

**FR-7.5.1** — WHERE `access_control` is specified, the platform shall enforce auth on all exposed endpoints.

**FR-7.5.2** — WHEN per-endpoint `rate_limit` is specified, the server shall enforce it and return HTTP 429 when exceeded.

**FR-7.5.3** — The platform shall support Cognito, IAM, OAuth, and API key auth methods.

### 7.6 Audit

**FR-7.6.1** — WHERE `audit.enabled` is true, the platform shall log all invocations, tool calls, guardrail activations, and access decisions to an immutable audit log.

**FR-7.6.2** — WHEN the deployment target is `agentcore`, the audit log shall integrate with CloudTrail.

### 7.7 Model Resilience

**FR-7.7.1** — WHERE `model.fallback` is specified as an array, the runtime shall attempt each in order on failure.

**FR-7.7.2** — WHEN failover occurs, the runtime shall log the failed model, error, and fallback used.

### 7.8 Evaluation (Strands Evals SDK)

**FR-7.8.1** — The platform shall integrate with the Strands Evals SDK, supporting evaluators: output, trajectory, interactions, helpfulness, faithfulness, goal success rate, tool selection accuracy, tool parameter accuracy, and custom.

**FR-7.8.2** — WHERE `evaluation.run_on` is `"compile"`, the platform shall execute evals during compilation and fail if any metric is below threshold.

**FR-7.8.3** — The platform shall support user simulation via Strands Evals simulators for automated testing.

---

## 8. Versioning & Registry Requirements

### 8.1 Agent Versioning

**FR-8.1.1** — WHEN an agent config is published, the platform shall create an immutable `AgentVersion` record with a unique `version_id`, `created_by`, `created_at`, and `status: "published"`.

**FR-8.1.2** — The execution plane shall always load the `published` version of an agent unless a specific version is pinned in the endpoint binding.

**FR-8.1.3** — WHEN a new version is published, the previous version's status shall change to `deprecated`.

**FR-8.1.4** — The platform shall support rollback: changing an agent's `published_version` to a previous `AgentVersion`.

### 8.2 Agent Registry

**FR-8.2.1** — The platform shall support two registry backends:
- **File-based** (default): config files on disk, compiled at startup. For single-team/dev use.
- **Store-based**: DynamoDB or similar, loaded at request time. For multi-team platform use.

**FR-8.2.2** — WHEN using the store-based registry, the execution plane shall load agent config from the registry at each request (with caching), not at process startup.

**FR-8.2.3** — The registry shall expose a `GET /registry/agents` endpoint listing all agents with their current published version, exposure mode, and status.

---

## 9. Compilation Requirements

### 9.1 Pipeline

**CR-9.1.1** — The compiler shall execute: Parse → Validate Schema → Resolve References → Build Dependency Graph → Detect Cycles → Topological Sort → Hydrate → Validate Runtime.

**CR-9.1.2** — WHEN parsing, the compiler shall produce frozen, immutable IR dataclasses. The IR is a snapshot of what the user wrote; the compiler never mutates it.

**CR-9.1.3** — The topological sort shall use Kahn's algorithm with alphabetical tie-breaking for deterministic compilation order across runs.

### 9.2 Dependency Resolution

**CR-9.2.1** — IF the dependency graph contains a cycle, THEN the compiler shall reject with an error showing the full cycle path.

**CR-9.2.2** — IF a reference targets a nonexistent entity, THEN the compiler shall reject with an error listing available alternatives.

### 9.3 Schema Validation

**CR-9.3.1** — The compiler shall validate configs against a published JSON Schema.

**CR-9.3.2** — The config shall include `schema_version`. IF the version is unsupported, THEN the compiler shall reject with migration guidance.

### 9.4 Compilation Output

**CR-9.4.1** — The compiler shall produce a `Platform` object with `.agents`, `.workflows`, `.registry`, `.serve()`, `.get()`.

**CR-9.4.2** — Compiled output shall consist entirely of standard Strands SDK objects. No platform wrapper classes between user and Strands at runtime.

---

## 10. CLI Requirements

**CLI-10.1** — `strands-platform init <name>` — scaffold project with directory structure, example config, placeholders.

**CLI-10.2** — `strands-platform compile <path> --validate` — parse, resolve, validate without model calls. For CI/CD.

**CLI-10.3** — `strands-platform run <path> --agent <name>` — compile and start interactive chat with named agent.

**CLI-10.4** — `strands-platform run <path> --workflow <name>` — compile and execute workflow with stdin or `--input`.

**CLI-10.5** — `strands-platform serve <path> --port <port>` — compile and start FastAPI server with all endpoints.

**CLI-10.6** — `strands-platform deploy <path> --target <target>` — generate deployment artifacts (lambda, fargate, ec2, agentcore, docker, kubernetes).

**CLI-10.7** — `strands-platform list <path>` — print all agents, workflows, skills, SOPs, bindings, and dependency graph.

**CLI-10.8** — `strands-platform publish <path> --agent <name>` — create a new AgentVersion in the registry.

**CLI-10.9** — `strands-platform rollback <agent_name> --version <version_id>` — set published version to a previous version.

**CLI-10.10** — IF an invalid path is provided, THEN the CLI shall exit non-zero with an error.

---

## 11. Deployment Requirements

### 11.1 Targets

**DR-11.1.1** — The platform shall support: local (CLI/Python), Lambda, Fargate, EC2, AgentCore, Docker, Kubernetes, App Runner.

**DR-11.1.2** — WHEN target is `agentcore`, the platform shall: wrap agents in `BedrockAgentCoreApp`, enable AgentCore Observability, configure AgentCore Memory, connect AgentCore Gateway for MCP, apply AgentCore Policy, configure AgentCore Identity.

### 11.2 Shared Runtime Deployment

**DR-11.2.1** — The platform shall support a shared runtime deployment where a single service (Lambda or Fargate) serves all agents. Routing is by `agent_id` in the request path, not by separate deployments.

**DR-11.2.2** — The shared runtime shall load agent configs from the registry (file-based or store-based) and compile on demand with caching.

### 11.3 Session Management

**DR-11.3.1** — Supported session backends: `memory` (in-process), `dynamodb`, `s3`, `agentcore`, `valkey` (Redis).

**DR-11.3.2** — WHEN `memory_policy.tenant_scoped` is true, session data shall be isolated per authenticated tenant.

### 11.4 Observability

**DR-11.4.1** — The platform shall include a built-in logging hook for structured logs.

**DR-11.4.2** — WHERE OTEL is configured, the platform shall use Strands' native OTEL instrumentation.

**DR-11.4.3** — WHERE AgentCore deployment, the platform shall enable AgentCore Observability automatically.

---

## 12. Non-Functional Requirements

**NFR-12.1** — WHILE compiling 20 agents and 5 workflows, the compiler shall complete in under 5 seconds excluding network I/O.

**NFR-12.2** — The platform shall depend only on the public Strands SDK API surface. No internal/private APIs.

**NFR-12.3** — Skills shall comply with AgentSkills.io for portability across Claude, Codex, VS Code Copilot.

**NFR-12.4** — SOPs shall comply with Strands Agent SOP format for portability across Claude Code, Kiro, Cursor.

**NFR-12.5** — `compile --validate` shall exit 0 on success, non-zero on failure, with structured output for CI/CD.

**NFR-12.6** — WHERE `--dry-run` is specified, no network requests shall be made.

**NFR-12.7** — The platform shall support the Strands Evals SDK for quality measurement.

**NFR-12.8** — `Agent()` construction with 10 tools SHALL complete in under 200ms. IF construction exceeds 50ms, an Agent instance pool with conversation state reset SHALL be implemented to avoid per-request construction overhead.

---

## 13. Implementation Milestones

| Milestone | Scope | Tasks | Deliverables |
|---|---|---|---|
| **1: Usable Engine** (weeks 1-4) | Single agents, CLI, API server | 1.01–1.18 | JSON parser, IR, tool resolver, dep graph, async streaming @tool wrappers, engine + hydrator with session support, FastAPI server with wildcard routes + SSE, CLI (compile/run/serve/list), Agent() construction benchmark |
| **2: Governed Platform** (weeks 5-10) | Multi-agent, skills, SOPs, governance, registry | 2.01–2.19 | MCP connection pool, AgentSkills.io (3 patterns), SOPs, Graph/Swarm/Workflow compilers, policies, Bedrock Guardrails, cost/data governance/audit, DynamoDB registry with compile-on-publish + versioning, access control, multi-tenancy |
| **3: Dynamic + Deploy** (weeks 11-14) | Runtime creation, evals, deployment | 3.01–3.16 | Governed use_agent/swarm/workflow, load_tool, hot reload, Strands Evals pipeline, Lambda/Fargate/AgentCore/Docker/K8s targets, YAML/Markdown parsers, init scaffolding, end-to-end test suite |

---

## 14. Project Structure Convention

```
my-platform/
├── config/
│   ├── agents.json              # Agent definitions
│   ├── workflows.json           # Graph/Swarm/Workflow definitions
│   ├── bindings.json            # Endpoint bindings (exposure config)
│   └── platform.json            # Defaults, governance, deployment, observability
├── skills/                      # AgentSkills.io standard
│   ├── web-research/
│   │   ├── SKILL.md
│   │   ├── scripts/
│   │   └── references/
│   └── pdf-processing/
│       └── SKILL.md
├── sops/                        # Strands Agent SOPs
│   ├── code-review.sop.md
│   └── research-workflow.sop.md
├── tools/                       # Custom @tool functions
│   ├── __init__.py
│   └── custom.py
├── conditions/                  # Graph routing functions
│   ├── __init__.py
│   └── routing.py
├── hooks/                       # HookProvider classes
│   ├── __init__.py
│   └── custom_guardrails.py
├── tests/
│   ├── test_compilation.py
│   └── evals/
│       └── support_quality.jsonl
├── platform.py                  # Entry point: registry + compile + serve
└── requirements.txt
```

---

## 15. The Platform Contract

For any agent on the platform:

| Question | Config Answer | Runtime Implementation |
|---|---|---|
| **What is it?** | `agent_id`, `name`, `description` | Registry record + AgentVersion |
| **What can it do?** | `tools`, `skills`, sub-agent refs | Strands tools + Skills progressive disclosure |
| **What is it allowed to do?** | `tool_policy`, `guardrails`, `data_governance`, `dynamic_capabilities` | Bedrock Guardrails + HookProviders + governed tools |
| **How should it behave?** | `instructions` (SOP or prompt), `output_schema` | Strands system_prompt + structured output |
| **How is it exposed?** | `exposure_mode`, `ui_template`, EndpointBindings | FastAPI routes + SSE streaming + UI config metadata |
| **How is it observed?** | `observability`, `audit`, `cost` | OTEL traces + audit log + cost allocation tags |

If the platform can answer these consistently for every agent, it's a real platform. The agents are deployable definitions on top of it.

---

## 16. Glossary

| Term | Definition |
|---|---|
| **Agent** | Strands `Agent` instance: model + prompt + tools |
| **Agent-as-Tool** | Async `@tool` wrapper with `stream_async()` yielding sub-agent events |
| **AgentVersion** | Immutable snapshot of an agent's config at publish time |
| **Control Plane** | Builder-side service: create, edit, validate, version, publish agents |
| **Execution Plane** | Runtime-side engine: load agent, execute, stream, trace, enforce policy |
| **Exposure Mode** | How an agent is consumed: chat, api, form, embedded, subagent_only, background |
| **Endpoint Binding** | Route + auth + rate limit + UI config for one agent exposure |
| **Graph** | Deterministic workflow via `GraphBuilder` with conditional edges and cycles |
| **Swarm** | Autonomous collaboration via `Swarm` with shared memory and handoffs |
| **Workflow** | DAG pipeline via `workflow` tool with tasks and dependencies |
| **Handoff** | Explicit transfer to human via `handoff_to_user` |
| **Skill** | AgentSkills.io capability package: SKILL.md + resources |
| **SOP** | Standard Operating Procedure: structured markdown with RFC 2119 constraints |
| **Progressive Disclosure** | 3-phase loading: metadata → instructions → resources |
| **Dynamic Capabilities** | Runtime sub-agent/tool creation via governed `use_agent`/`swarm`/`workflow` |
| **Governed Tool** | Wrapper around a Strands dynamic tool that enforces allowed-tools/models/limits |
| **Hook** | Strands `HookProvider` intercepting lifecycle events |
| **MCP** | Model Context Protocol: standard for external tool servers |
| **A2A** | Agent-to-Agent protocol: cross-framework agent interop |
| **IR** | Intermediate Representation: frozen dataclasses between parse and hydrate |
| **Platform** | Compiled output: all agents, workflows, and registry ready to run |
| **Registry** | Store of AgentDefinitions and AgentVersions (file-based or DynamoDB) |

---

## 17. References

| Resource | URL |
|---|---|
| Strands Agents SDK | https://github.com/strands-agents/sdk-python |
| Strands Documentation | https://strandsagents.com/docs/ |
| Strands Streaming (Sub-Agent Pattern) | https://strandsagents.com/docs/user-guide/concepts/streaming/#sub-agent-streaming-example |
| Strands Multi-Agent Patterns | https://strandsagents.com/docs/user-guide/concepts/multi-agent/multi-agent-patterns/ |
| Strands Graph | https://strandsagents.com/docs/user-guide/concepts/multi-agent/graph/ |
| Strands Swarm | https://strandsagents.com/docs/user-guide/concepts/multi-agent/swarm/ |
| Strands Workflow | https://strandsagents.com/docs/user-guide/concepts/multi-agent/workflow/ |
| Strands Hooks | https://strandsagents.com/docs/user-guide/concepts/agents/hooks/ |
| Strands Evals SDK | https://strandsagents.com/docs/user-guide/evals-sdk/quickstart/ |
| Strands Tools | https://github.com/strands-agents/tools |
| Strands Agent SOPs | https://github.com/strands-agents/agent-sop |
| AgentSkills.io Specification | https://agentskills.io/specification |
| AgentSkills for Strands | https://github.com/aws-samples/sample-strands-agents-agentskills |
| Bedrock AgentCore | https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-bedrock-agentcore.html |
| AWS GenAI Prescriptive Guidance | https://docs.aws.amazon.com/prescriptive-guidance/latest/strategy-enterprise-ready-gen-ai-platform/introduction.html |
| AWS GenAI Landing Zone | https://aws.amazon.com/blogs/apn/rapidly-build-a-generative-ai-landing-zone-on-aws/ |
| Strands 1.0 Announcement | https://aws.amazon.com/blogs/opensource/introducing-strands-agents-1-0-production-ready-multi-agent-orchestration-made-simple/ |
