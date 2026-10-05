# Threat model — AgentGate v0.1 verifier

**System:** local, disposable MCP authorization regression test described in [ARCHITECTURE.md](ARCHITECTURE.md). It is **not** a runtime security boundary for production agents. The protected assets are test accuracy, temporary gateway credentials, and the developer's machine. The upstream fixture must be harmless.

## Trust boundaries

| Boundary | Untrusted or fallible input | Security rule |
| --- | --- | --- |
| CLI → gateway | URL, responses, tool metadata, denial shape, timeouts | Do not equate a returned error or missing discovery entry with blocked execution. |
| Gateway → fixture | Calls, arguments, retries, delayed events | Count arrivals at the fixture independently; correlate unique run nonce and tool; unexpected/duplicate arrivals cannot yield PASS. |
| Environment → CLI | Test credential, fixture binding, output path | Read test token from environment; never print it. Use least-privileged disposable credentials. No production secrets or real side-effect tools. |
| Observer → verdict | Local fixture journal and observation window | If the journal is missing, reset, contaminated by concurrent runs, or unreachable, return INCONCLUSIVE. Results assert only the observed route and interval. |

## Concrete threats and required tests

| Threat / mistaken inference | Mitigation and test expectation |
| --- | --- |
| Always-denying, disconnected, or wrong gateway produces a “safe” zero count. | ALLOW positive control must succeed and arrive exactly once at the same fixture. Wrong target/run ID or absent control yields FAIL/INCONCLUSIVE, never PASS. |
| Gateway responds DENY but forwards `deny_probe` anyway. | Any matching arrival at fixture is FAIL, regardless of response or audit record. |
| Denied tool hidden from discovery but callable by name. | Invoke `deny_probe` directly; record list visibility separately. |
| Host/client sends an invalid 2026 request and is rejected before policy evaluation. | Validate revision and required headers/metadata; positive control must traverse the route. Invalid wire/auth returns INCONCLUSIVE. |
| Late forwarding or race near test completion. | Use a specified bounded observation window, isolate per-run nonce, and state that zero calls is time-limited evidence. Do not assert permanent non-execution. |
| Other process spoofs the fixture or contaminates its counter. | Loopback fixture, unique unguessable run ID, single-run isolation, and refuse unexpected records. A malicious local administrator is outside scope. |
| Secrets in CLI output, JSON evidence, crash logs or shell history. | Accept token by environment variable name, redact URLs/headers/arguments where sensitive; use synthetic arguments; test all output modes. |
| Fixture exposed on a network or accidentally pointed at a real service. | Default to loopback, refuse non-loopback binding in v0.1, use explicit fixture identity check and harmless tool names; never execute arbitrary commands or user-supplied tool implementations. |
| Gateway bypass or another client route still reaches production upstream. | State that the result covers only the exercised configured path. Do not issue a whole-system assurance badge. |

**Out of scope:** compromised OS or gateway process, production OAuth delegation, actual external side effects, coding-agent shell and native tools, all other transports/revisions, distributed multi-instance behavior, and proof against malicious test operators. A lost response cannot establish whether a real side effect occurred. The reporting language must preserve this limitation even for a PASS.

**Release security bar:** deterministic integration tests for the positive/negative controls, denied forward despite error response, broken observation, invalid headers/version, duplicate/delayed calls, concurrent contamination, and secret redaction. Review the actual implementation against the current [MCP transport](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http) and [tool invocation](https://modelcontextprotocol.io/specification/2026-07-28/server/tools) contract.
