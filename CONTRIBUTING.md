# Contributing to AgentGate

AgentGate is currently a **proposed local MCP authorization verification tool**. Read [docs/WEDGE.md](docs/WEDGE.md), [docs/VALIDATION_PLAN.md](docs/VALIDATION_PLAN.md), [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md), and [docs/IMPLEMENTATION_READINESS.md](docs/IMPLEMENTATION_READINESS.md) before proposing code. The original gateway concept in PRODUCT.md/MVP.md is historical.

## Repository layout

| Path | Purpose |
| --- | --- |
| `LICENSE` | AGPL-3.0 license. |
| `docs/PRODUCT.md`, `docs/MVP.md` | Original gateway proposal, superseded for current v0.1. |
| `docs/ECOSYSTEM.md`, `docs/WEDGE.md` | Protocol/competitor evidence and pivot decision. |
| `docs/VALIDATION_PLAN.md`, `docs/ARCHITECTURE.md`, `docs/THREAT_MODEL.md`, `docs/IMPLEMENTATION_READINESS.md` | Current validation, bounded design, threats, and implementation gate. |
| `src/`, `tests/`, `examples/` | **Planned, not yet present.** Add only after the GO decision; implementation, verification tests, disposable configuration respectively. |

There is no implementation or established language/build toolchain yet. Do not present placeholder commands as working setup instructions.

## Conventions and test expectations

- Keep changes confined to the chosen v0.1 job: a local verifier and harmless upstream fixture. A policy engine, approval UX, production gateway, dashboards, cloud services and integrations are outside scope.
- When implementation starts, choose one language and formatter/linter in the first change; document exact commands in README. Use explicit types at protocol boundaries, small modules, descriptive names, and deterministic, non-sensitive error messages. Avoid logging credentials or full tool arguments.
- Treat MCP revisions as distinct wire contracts. Pin and cite the supported revision; use an official SDK where appropriate and the [official conformance suite](https://github.com/modelcontextprotocol/conformance) for protocol checks. Do not claim compatibility from a unit test or from a different revision.
- Meaningful integration tests must observe the fixture independently: ALLOW arrives once, DENY never arrives during the declared interval, and a denied client response paired with an upstream arrival **fails**. Broken setup/observation must be inconclusive. Cover malformed/auth/version cases, duplicates, isolation and redaction per [THREAT_MODEL.md](docs/THREAT_MODEL.md).
- Keep test credentials disposable; commit no tokens, local event records, generated logs or sensitive arguments. Document limitations and results precisely.

## Contribution workflow

1. Before code, attach validation evidence required by [VALIDATION_PLAN.md](docs/VALIDATION_PLAN.md) and obtain the recorded GO decision. Documentation and reproducible research contributions are welcome now.
2. Open an issue describing the precise failure or user job, expected observable behavior, supported MCP revision, and a minimal safe reproduction. Check whether the official MCP or an existing gateway project is the better upstream home.
3. Submit a focused pull request with a short rationale, changed docs, and tests for altered behavior. Run the documented formatter, static checks, integration tests and revision-specific conformance checks once a toolchain exists; paste commands/results in the PR.
4. For security findings, avoid public exploit details until maintainers can assess the report. Do not submit real credentials or data. A release claim requires reproducible upstream-observation evidence, not merely a green client response.

All contributions are under the repository's AGPL-3.0 license. There is currently no separate CLA or guaranteed support/response time.
