# Validation plan — AgentGate v0.1

**Status:** Required evidence before implementation of the proposed [WEDGE.md](WEDGE.md) pivot. Do not treat documentation review as completed validation.

## Question and hypothesis

Can a developer responsible for a configured MCP gateway verify, with less setup than current alternatives, that a denied tool call produces **zero observed calls** at a controlled upstream MCP server while an allowed control produces **one**? The tool tests the gateway's configured path; it does not prove all paths are protected.

## Evidence to collect

| Gate | Procedure | Artifact and pass threshold |
| --- | --- | --- |
| User need | Interview **five** unrelated MCP gateway maintainers or policy owners. Ask for their last policy change, how they checked forwarding, any misconfiguration, available CI test, and willingness to use this check. Avoid pitching a solution first. | Anonymized notes and count of **at least two** people reporting this exact unsolved job. Expressions of abstract interest are insufficient. |
| Existing workflow | Run documented quickstarts of **Archestra, agentgateway, ActionProxy, and one of ContextForge or Docker**. For the same allowed and denied test calls, record setup steps, actual downstream observations, time, and gaps. Also run the official MCP conformance suite and Inspector where applicable; inspect and try OWASP's regression harness. | Reproducible commands, versions, configuration excerpts without secrets, logs, and a comparison table. **Two named gateways** must lack an equivalently quick upstream-observed test for a standalone product to be justified. |
| Protocol feasibility | Test a disposable, harmless upstream with **two independent gateways** on a documented **2026-07-28 Streamable HTTP** path, using each gateway's normal configuration. Validate request headers/body agreement, tool discovery, allowed call, denied direct call, and the auth/error shapes seen by a test client. | Exact tested versions/configuration and observed call counts. No modification to gateway code or production server; if a gateway lacks that revision, mark it unsupported, not failed. |
| Five-minute utility | Give an external developer a documented working gateway plus instructions to set a disposable upstream. Time from invoking the verification workflow to a readable ALLOW/DENY result; also note any prerequisite setup separately. | A credible result in **under five minutes after setup**, including an ALLOW call observed once and a denied call observed zero times during the bounded observation interval. No silent pass on broken wiring. |

## Recording rules

- Record **observed**, **claimed**, and **inferred** separately. Capture client response and upstream observation independently; an error response alone is not a passing DENY.
- Use unique test IDs, harmless tools, fake arguments and disposable credentials. State the observation interval; zero observed calls is bounded evidence, not proof of eternal non-execution.
- Record unexpected direct access, authentication failure, wrong protocol revision, unsupported policy semantics, incomplete setup and uncertain side-effect outcome as limitations or inconclusive results.
- Compare the official [conformance suite](https://github.com/modelcontextprotocol/conformance), [Inspector](https://github.com/modelcontextprotocol/inspector), [OWASP harness](https://github.com/OWASP/Agent-Security-Regression-Harness), and vendor tools against the **same job**, rather than claiming they lack features without testing.

## Decision gate

**GO to v0.1 implementation** only when all four gates pass and a reviewer can reproduce the evidence. **NO-GO** while the customer evidence, head-to-head comparison, or two-gateway feasibility result is missing. If an existing project's workflow already meets the job adequately, contribute a scenario or documentation to that project and **STOP** AgentGate as an independent product.

Store results in a dated follow-up to this plan or linked issue before changing [IMPLEMENTATION_READINESS.md](IMPLEMENTATION_READINESS.md). Validation is a product decision, not implementation work.
