# AgentGate v0.1 — Minimum Viable Product

**Purpose:** Prove that a developer can put AgentGate between two AI agent integrations and one MCP server, then get three enforceable outcomes for tool calls: **ALLOW**, **DENY**, and **REQUIRE_APPROVAL**. This document is the build and acceptance contract for v0.1; [PRODUCT.md](PRODUCT.md) defines the longer-lived product boundary.

## 1. The smallest useful deployment

A single local, self-hosted AgentGate process accepts **Streamable HTTP MCP** connections from multiple registered agent integrations and connects to **one configured Streamable HTTP MCP server**. Target the MCP **2025-11-25** protocol revision for v0.1. Document and test the exact client/server pair used for the example. Reject unsupported protocol versions and MCP capabilities clearly; do not silently pass ungoverned operations through.

The gateway mediates the necessary initialization/session lifecycle, `tools/list`, and `tools/call`. A registered agent authenticates to AgentGate with its own bearer token. AgentGate uses an operator-configured, appropriately scoped upstream credential if the downstream server requires one. The upstream server may therefore see AgentGate's credential rather than the agent's identity; the audit identity is asserted and enforced **at AgentGate**, not propagated as a guaranteed downstream identity.

A policy gives each agent an explicit outcome for each exact tool name. No rule means **DENY**. `tools/list` includes ALLOW and REQUIRE_APPROVAL tools, but hides DENY tools. This list is advisory to the client; `tools/call` is checked again before execution.

## 2. User journeys

| Journey | User actions | Required result |
| --- | --- | --- |
| First connection | Operator creates two agent tokens, configures one server and per-agent policy, starts AgentGate; developer points an MCP client at its URL with one token. | The client initializes and sees only that agent's permitted or approval-gated tools. No change to the upstream server is needed. |
| ALLOW | Agent calls an explicitly allowed tool. | AgentGate writes an allow decision, forwards the call once, returns the upstream result, and records success or upstream failure. |
| DENY | Agent calls a hidden, unknown, or explicitly denied tool by name. | AgentGate returns a clear denial, writes a denial event, and sends **no call** to the upstream server. |
| REQUIRE_APPROVAL | Agent calls an approval-gated tool. Local operator lists and inspects the pending request, then approves or denies it. Agent retries the identical call after approval. | First call is never forwarded. Approval permits **one matching retry**, before expiry, to be forwarded once. Denial or expiry leaves the call blocked. |
| Investigation and revocation | Operator reads audit records, disables an agent in config, restarts the process, and retries. | Records connect decision and result by request ID; disabled agent calls are rejected without upstream traffic. |

No standard MCP client is assumed to pause and resume an interrupted tool call automatically. A developer must make the retry after approval (or use the documented sample client). AgentGate must not claim a seamless human-in-the-loop UX in arbitrary hosts.

## 3. Required capabilities and behavior

### Identity and policy

- Map each independently generated bearer token to exactly one configured agent ID. Never trust an agent ID supplied in tool arguments, a request header other than the credential, or the MCP client name.
- Token generation is local; configuration references environment variable names, not literal secrets. Never print a token after generation, log it, or forward it upstream. Disabling an agent takes effect after a documented config restart in v0.1.
- Policy entries are exact tool-name matches, each set to ALLOW, DENY, or REQUIRE_APPROVAL. Duplicate entries, unknown outcomes, missing secrets, and invalid config stop startup. Unlisted tools and agents are denied. No wildcards, roles, conditions on arguments, or inherited grants.
- A call-time decision is mandatory even when a tool was filtered from discovery. A missing or unreadable policy, broken local approval channel, or unavailable audit sink must not produce a forwarded call.
- Reject malformed requests, unsupported methods/capabilities, mismatched protocol versions, and unauthenticated calls. Preserve the supported MCP lifecycle and upstream tool result semantics for allowed calls. Never turn an upstream failure into a success.

### Approval contract

1. For REQUIRE_APPROVAL, calculate a digest over the canonical tool arguments and bind a pending request to **agent credential identity, server ID, exact tool name, argument digest, and current policy fingerprint**. Give it a random, unguessable ID and a **5-minute expiry**. Do not forward the initial call.
2. Return an MCP tool error (`isError: true`) containing `APPROVAL_REQUIRED` and the pending ID in human-readable text. Document that clients must retry the same tool and arguments. Repeated identical attempts while pending return the same ID; changed arguments create a different request.
3. A trusted local operator can list requests and inspect the complete arguments before deciding. The management interface is a **local Unix socket** in a directory accessible only to the operator. Keep pending arguments **in process memory only**; never put them in the audit log or on disk. A restart discards pending and approved requests.
4. `approve` opens a **single-use**, at-most-5-minute grant for an identical retry by the same credential under the same policy. Consume the grant atomically **before** forwarding. If the policy, credential, arguments, tool, or expiry changes, the grant is invalid. A retry with different arguments requires a new approval.
5. `deny` marks that pending request denied; identical retries remain denied until its original expiry. An expired request cannot be approved. A subsequent attempt after expiry may create a new pending request.
6. Write audit events for pending, operator approval/denial, expiration when observed, retry consumption, and upstream outcome. The CLI operator is identified only as a **local operator**, not as a verified human identity.

An approved tool with a side effect can still execute upstream and then time out before AgentGate receives a result. A retry after the one-use grant was consumed **must not automatically execute again**. The audit outcome in that case is unknown, not proof of no side effect.

### Audit and local security

Write append-only JSON Lines containing timestamp, event/request IDs, registered agent ID or unauthenticated marker, server ID, tool name when parseable, decision, approval ID when applicable, outcome, and error category. No credentials, full arguments, or tool results in the audit by default. Ensure the allow/pending decision is durable before forwarding or returning it. If an audit write fails, fail closed for new tool calls. The local operator owns file permissions and retention; no tamper-evidence or compliance claim.

Bind the MCP listener to loopback by default. v0.1 assumes the operator controls the host, the socket directory, and the upstream network path. Remote exposure requires an operator-managed secure transport and access boundary; that deployment is outside the v0.1 walkthrough. AgentGate cannot stop bypass traffic to the upstream server.

## 4. Expected CLI contract

The executable is `agentgate`. Commands have `--help`, concise errors on stderr, and nonzero exit status on failure. The example is supported on Linux and macOS; Windows and containers are postponed.

| Command | Expected behavior |
| --- | --- |
| `agentgate token generate` | Prints one cryptographically random bearer token to stdout once. Does not save or log it. Operator places it in a named environment variable. |
| `agentgate config check --config agentgate.yaml` | Validates syntax, referenced environment variables, duplicate IDs/tool rules, policy outcomes, and safe local paths without starting the gateway. Exits 0 only for valid config; never prints secret values. |
| `agentgate serve --config agentgate.yaml` | Starts the MCP endpoint and local management socket, prints only non-secret bind locations, and fails startup if policy, audit file, socket, or credentials cannot be initialized. |
| `agentgate approvals list --socket ./var/agentgate.sock` | Lists pending IDs, agent, tool, age, and expiry; does not display arguments. |
| `agentgate approvals show <id> --socket ./var/agentgate.sock` | Shows agent, tool, complete arguments, and expiry **only to the local operator**; output may contain sensitive data. |
| `agentgate approvals approve <id> --socket ./var/agentgate.sock` | Approves an existing pending request once and prints its remaining expiry. Unknown, denied, already consumed, or expired IDs fail nonzero. |
| `agentgate approvals deny <id> --socket ./var/agentgate.sock` | Denies an existing pending request and prints confirmation. Subsequent approval of it fails nonzero. |

The CLI does not add an administrator web API. The local socket has restrictive permissions; access to it is the v0.1 operator authority. The CLI must show enough context to avoid approving by an opaque ID alone.

## 5. Configuration example

This is the **intended** v0.1 schema, not an assertion that commands already exist. The two agent tokens and optional upstream token are supplied through the environment. The named tools are example names; the quickstart's demo MCP server must expose them.

```yaml
listen: 127.0.0.1:8765
server:
  id: demo
  url: http://127.0.0.1:8766/mcp
  authorization_env: DEMO_MCP_TOKEN # omit if upstream needs no token
agents:
  - id: researcher
    token_env: AGENTGATE_RESEARCHER_TOKEN
    enabled: true
    tools:
      search: ALLOW
      publish_report: REQUIRE_APPROVAL
  - id: observer
    token_env: AGENTGATE_OBSERVER_TOKEN
    enabled: true
    tools:
      search: ALLOW
      publish_report: DENY
audit:
  path: ./var/audit.jsonl
approvals:
  socket: ./var/agentgate.sock
  ttl_seconds: 300
```

Example outcomes: `researcher/search → ALLOW`; `observer/publish_report → DENY`; `researcher/publish_report → REQUIRE_APPROVAL`. Any unlisted tool also results in DENY. The client endpoint is `http://127.0.0.1:8765/mcp` with `Authorization: Bearer <that agent's token>`.

## 6. Acceptance criteria

A clean-install walkthrough and automated integration tests must prove:

1. Two tokens authenticate as distinct configured agents. Missing, invalid, or disabled credentials and unsupported protocol operations are rejected without upstream tool calls.
2. `tools/list` for each agent matches its policy; calling a DENY or unlisted tool directly still results in zero upstream calls.
3. ALLOW forwards exactly one call with unchanged arguments and returns the upstream result. Upstream errors remain errors. The audit distinguishes authorization from upstream outcome.
4. REQUIRE_APPROVAL returns a pending ID without forwarding. The CLI shows the exact arguments. A local approval allows exactly one identical retry; a second retry needs a new approval.
5. Changed arguments, changed/disabled credential, changed policy after restart, expired grants, denied IDs, and process restart cannot reuse an approval. Two concurrent identical retries cannot both consume one grant.
6. No secret, argument, or result appears in default logs or audit. The management socket is local and restricted; audit or approval subsystem failure fails closed.
7. A developer following the quickstart demonstrates all three outcomes, an approval denial, revocation, and audit inspection in **15 minutes** on a supported machine.

## 7. Explicitly postponed

No dashboard, hosted control plane, cloud infrastructure, enterprise SSO, OAuth/SCIM provider, billing, organization hierarchy, multi-server routing, stdio transport, Windows support, policy wildcards or argument rules, permanent approvals, persistent approval queues, remote approvers, chat notifications, automatic host resume, DLP/content inspection, prompt-injection defense, SIEM export, compliance certification, or integrations beyond the single demo server/client used for validation.

**Scope rule:** If a feature is unnecessary to demonstrate one of ALLOW, DENY, or REQUIRE_APPROVAL safely on the supported path, it is not part of v0.1.
