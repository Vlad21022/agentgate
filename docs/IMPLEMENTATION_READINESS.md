# Implementation readiness — AgentGate v0.1

## Unresolved blockers

1. **User demand unverified:** Five interviews have not been completed; there are no two unrelated external developers confirming the exact unsolved upstream-observation job.
2. **Competitive baseline unmeasured:** The Archestra, agentgateway, ActionProxy and ContextForge/Docker quickstarts have not been run head-to-head with official conformance, Inspector, OWASP and vendor tests. We cannot claim a material speed or evidence advantage.
3. **Two-gateway feasibility unverified:** No reproducible 2026-07-28 Streamable HTTP run has shown ALLOW once and DENY zero upstream on two independent existing gateways, using their normal configurations.
4. **Five-minute utility unverified:** No external developer has timed the workflow after setup. These are the explicit proceed thresholds in [VALIDATION_PLAN.md](VALIDATION_PLAN.md), not optional release polish.

## Assumptions we are accepting

- The first user can configure their **existing** gateway to route to a disposable local MCP server and supply one test identity without production secrets.
- v0.1 may test only **2026-07-28 Streamable HTTP** with an unauthenticated loopback fixture and an optional test bearer token to the gateway; any unsupported gateway/revision is reported honestly.
- A unique run ID and a bounded observation interval provide limited evidence for the tested route; they cannot prove that every route is protected or that a delayed side effect will never occur.
- The programming language and build tooling can be chosen in the first implementation task after GO; there is no existing codebase or CI to preserve.

## Exact v0.1 scope

A **local CLI verifier** and harmless **two-tool MCP fixture** for one operator-configured existing gateway and one configured path. Test `allow_probe` by confirming one successful client call and exactly one observed upstream invocation. Call `deny_probe` directly even if absent from `tools/list`; require a denial/unknown-tool response and zero observed upstream invocations during the stated interval. Report tool visibility separately, correlate by unique run ID, redact test credentials, and emit PASS/FAIL/INCONCLUSIVE with human-readable and optional JSON evidence. Use only 2026-07-28 Streamable HTTP. No runtime proxy, policy engine, identities, approvals, audit platform, dashboards, cloud, enterprise integration, stdio, OAuth upstream, or universal host support.

## First 10 implementation tasks in dependency order

1. After the validation GO, select one language, supported runtime, formatter, package workflow, and document reproducible local commands.
2. Establish a minimal project/CLI skeleton with versioned configuration, safe environment-variable token input and explicit 0/1/2 exit statuses.
3. Implement the 2026-07-28 request/response client boundary using a current official SDK where suitable; reject unsupported versions and validate required request metadata/headers.
4. Implement a loopback-only disposable MCP fixture exposing `allow_probe` and `deny_probe`, each harmless and deterministic.
5. Add run-scoped fixture arrival observation, unique nonce correlation, and strict detection of unexpected or duplicate calls.
6. Add verifier preflight for fixture identity/reachability, gateway URL, tool names, token handling and isolation of one active run; return INCONCLUSIVE on broken wiring.
7. Add discovery and the ALLOW positive control; require a successful response and exactly one matching fixture arrival.
8. Add direct DENY invocation regardless of discovery; capture response and fixture arrivals independently across a bounded observation interval.
9. Implement PASS/FAIL/INCONCLUSIVE verdicts, concise text/JSON evidence and full secret/argument redaction.
10. Write end-to-end regression tests for good path, denied-but-forwarded, always-denying, malformed/version/auth errors, timeouts, duplicates and contamination; run the official revision-specific conformance checks and document results on two named gateways.

## GO / NO-GO decision

**NO-GO for implementation as of 2026-10-05.** The four evidence blockers above are critical because [WEDGE.md](WEDGE.md) makes this an unvalidated pivot rather than a proven product. The repository is document-ready but **not implementation-ready**. Complete [VALIDATION_PLAN.md](VALIDATION_PLAN.md), record reproducible evidence, and update this decision to GO before task 1. If existing tools already meet the job adequately, STOP this standalone product and contribute there.
