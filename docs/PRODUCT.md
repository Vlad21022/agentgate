# AgentGate — Product Definition

**Status:** Product scope for v0.1  
**Core promise:** **Every AI agent gets an identity, permissions, and an audit trail.**

## 1. Problem

An MCP server exposes tools to AI clients. When several agents use the same server, an operator needs a reliable answer to three questions for each attempted tool call: **which registered agent called, was that agent allowed to call this tool, and what happened?** Today, teams can connect a client directly to a server without a consistent control point across those agents.

AgentGate is a self-hosted, open-source gateway on the path between an MCP client and an MCP server. It identifies the calling agent, applies an explicit tool policy before forwarding a call, and records the decision and result. Its unit of governance is a **registered agent's MCP tool call**, not the model's thoughts, the human user's intent, or every action the agent can take elsewhere.

The promise applies **only to calls routed through AgentGate**. It does not discover or block direct connections to a server, other tools, or out-of-band actions. An agent ID represents a credential issued to one agent integration; v0.1 cannot prove which model, process, or human actually used a stolen or shared credential.

## 2. Target users

| User | Need |
| --- | --- |
| Developer building multiple MCP-connected agents | Give each integration a separate identity and minimum tool access without modifying the downstream MCP server. |
| Small-team operator or security engineer | Review which agent attempted which tool and whether the gateway allowed, denied, or failed the call. |

The initial adopter can run and configure a gateway. v0.1 is intended for a small self-hosted deployment, not a managed enterprise rollout.

## 3. Primary use cases and jobs to be done

1. **Register an agent.** When I connect a new agent to an existing MCP server, I want a distinct gateway credential mapped to a stable agent ID so I can separate its access and activity from other agents.
2. **Grant minimum access.** When an agent needs one or a few tools, I want to allow exactly those tool names for that agent, with everything else denied by default, so a broad server connection does not imply broad tool access.
3. **Stop a forbidden call.** When an agent attempts a tool outside its allowlist, I want AgentGate to reject it before it reaches the downstream server and record the denial.
4. **Investigate a call.** When something unexpected happens, I want to answer which registered agent, server, and tool were involved, when the attempt happened, whether it was allowed, and whether the upstream call succeeded or failed.
5. **Revoke access.** When an integration is retired or its credential is suspected to be compromised, I want to disable that agent's gateway credential and have subsequent calls denied without changing other agents' access.

## 4. Value proposition

AgentGate gives a team a **single, understandable enforcement point** for agent-to-MCP-tool access. It reduces the effort of adding per-agent permissions and a consistent audit record to an MCP server that was not built to provide them. Its value depends on a simple deployment and a policy that an operator can inspect, test, and change.

It does **not** replace the MCP server's own authentication or authorization. Operators must still use appropriately scoped downstream credentials and keep the server from being reached directly where enforcement is required. Gateway identity is not a substitute for end-user identity or downstream permissions.

## 5. Differentiation to pursue

These are product choices, not claims that no other gateway has the features:

- **Agent-first policy:** Rules are expressed as *agent ID → exact server/tool allowlist*, rather than a general API management or model routing configuration.
- **Decision evidence by default:** Every mediated tool attempt yields a structured allow/deny/error record tied to the registered agent and a request ID, with no tool arguments or results logged by default.
- **Small, inspectable surface:** A self-hosted gateway, declarative configuration, and local audit output are sufficient for the first release. The operator can understand the effective policy without a cloud account or dashboard.
- **MCP-aware enforcement:** Tool discovery shown to an agent matches its allowlist, and a direct call to a hidden or unknown tool is still checked at execution time. Hiding a tool is usability; the call-time check is the security boundary.

## 6. v0.1 product boundary

**Supported path:** One gateway deployment fronts **one configured remote MCP server over Streamable HTTP** and accepts multiple registered MCP clients/agents over Streamable HTTP. It mediates the essential connection lifecycle and `tools/list` and `tools/call` flows needed for that path. A request or capability outside the supported surface is rejected clearly rather than forwarded around policy. The supported protocol revision and compatibility limits must be documented and tested before release.

**Identity:** Each agent integration receives its own gateway credential and stable ID. The gateway validates the credential on each request, never accepts an agent ID asserted only in the request body, and supports disabling a credential. Provisioning is an operator action. Secrets are never written to the audit log.

**Permissions:** A declarative, deny-by-default policy maps each registered agent to an allowlist of exact tool names on that one server. An unknown agent, unknown tool, missing rule, malformed request, or policy loading failure cannot result in a forwarded tool call. Both the advertised tool list and actual calls are governed. No wildcard grants or argument-dependent rules in v0.1.

**Audit:** For each attempted tool call, record timestamp, request ID, agent ID when authenticated (otherwise an unauthenticated marker), server ID, tool name when parseable, decision, outcome, and a non-sensitive error category. Keep the full tool arguments, tool results, prompts, and upstream credentials out of logs by default. Local structured, append-only event output is enough for v0.1; document retention and file access as the operator's responsibility. If the audit event cannot be written, do not forward an otherwise allowed call. An audit entry is evidence of what the gateway observed, not proof that an external side effect did or did not occur.

**Operator experience:** Provide one example configuration for two agents with different tool permissions, setup instructions, and a short demonstration of an allowed call, denied call, revocation, and corresponding audit events. No web UI is required.

## 7. Non-goals: what AgentGate must not become

- **An agent framework, orchestrator, or autonomous worker.** It does not plan tasks, host agents, choose models, or execute workflows.
- **A general LLM gateway.** It does not route model inference, manage prompts or tokens, evaluate model quality, or bill for usage.
- **A universal API gateway or MCP server marketplace.** It does not discover arbitrary servers, aggregate hundreds of connectors, transform tools, or offer a plugin catalog in v0.1.
- **A replacement OAuth provider or enterprise IAM product.** It does not issue end-user identities, implement SSO/SCIM, administer organizations, or reinterpret a downstream server's OAuth scopes.
- **A content safety, DLP, or prompt-injection firewall.** It does not inspect arguments for secrets, judge intent, classify harmful content, or claim to make untrusted tool output safe.
- **A human approval platform.** No approval queues, chat notifications, exception workflows, or policy suggestions from an LLM.
- **A compliance or forensics suite.** A local audit record is not tamper-evident storage, a SIEM, legal evidence, a certification, or a guarantee of complete activity capture.
- **An endpoint or network security control.** It cannot govern clients that bypass it or actions outside the supported MCP tool path.
- **A hosted SaaS in v0.1.** No multi-tenant control plane, billing, analytics dashboard, or organization hierarchy.
- **A broad protocol proxy in v0.1.** No stdio bridge, resources, prompts, sampling, elicitation, multi-server routing, or promised support for every MCP extension.

A proposed feature belongs in v0.1 only if it is necessary to establish agent identity, enforce the exact tool allowlist, or record the mediated decision reliably on the supported path.

## 8. v0.1 success criteria

The release is ready when all of these can be demonstrated from a clean setup:

1. **Connection:** A documented MCP client connects through AgentGate to one documented Streamable HTTP MCP server, discovers an allowed tool, and completes an allowed call without changing that server.
2. **Isolation:** Two agents with different credentials see different allowed tool lists. A tool allowed to agent A but denied to agent B is never invoked upstream when B attempts a direct `tools/call`.
3. **Fail closed:** Missing, invalid, disabled, or wrong-agent credentials; unknown tools; and missing or invalid policy are denied. An unavailable audit sink prevents forwarding an allowed tool call.
4. **Traceability:** Allowed, denied, failed-upstream, and unauthenticated attempts produce distinguishable local records with request IDs. No credential, tool argument, or tool result appears in those records by default.
5. **Usability:** An operator following the README and example policy can configure two agents, demonstrate a denial, revoke one credential, and locate the corresponding audit events in **15 minutes** on a supported environment.
6. **Honesty:** Documentation states supported MCP revision and methods, threat assumptions, downstream credential behavior, bypass limits, and audit limitations. Tests cover the policy boundary and the documented success path.

**Explicitly not a v0.1 success metric:** number of supported servers, number of policy features, dashboard polish, enterprise compliance claims, or AI-driven threat detection.
