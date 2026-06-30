---
name: harden-with-workspace-tests
description: "Use the Sigma Shake workspace test taxonomy to harden any app or service against functional, performance, resiliency, and security failures. Use when adding test coverage, designing a resilience plan, reviewing an app's test gaps, preparing a production-readiness gate, or mapping a change to the canonical workspace test kinds."
---

# Harden With Workspace Tests

## Overview

Use the workspace test taxonomy as a resilience design tool, not just a checklist. Map concrete app risks to test kinds, add the smallest useful tests that exercise those risks, and leave clear evidence for gaps that cannot be automated yet.

Current Sigma Shake workspace source defines 20 canonical kinds. Older language may call this the "19 test types"; treat that as a legacy label and include `dependencies` when the app has third-party packages or supply-chain exposure.

## Workflow

1. Inventory the app surface:
   - Identify runtime, entrypoints, persistent state, external APIs, auth boundaries, queues, schedulers, deployment targets, and the top user or operator journeys.
   - Inspect existing test files and runnable scripts. Prefer `rg --files`, package manifests, Makefiles, CI files, and service-specific test config over assumptions.
   - In Sigma Shake, confirm the live taxonomy from `sigmashake-workspace/src/client/lib/test-catalog.ts`, `sigmashake-workspace/src/types.ts`, and `sigmashake-gov/src/server/handlers/lib/detect-test-kinds.ts` if there is any drift.

2. Build a risk-to-kind matrix:
   - For every touched feature or failure mode, name the failure, user impact, detection signal, and the test kinds that should catch it.
   - Prefer real fault injection and observable assertions over placeholder tests.
   - Mark a kind `n/a` only when the app truly lacks that risk class; otherwise mark it as `missing` with the exact script or file to add next.

3. Add or repair tests in this order:
   - Functional correctness first: `unit`, `integration`, `component`, `api`, `e2e`, `regression`.
   - Boot and environment safety next: `configuration`, `onboarding`.
   - Resilience and recovery next: `chaos`, `failover`, `dr`.
   - Capacity behavior next: `load`, `spike`, `stress`, `soak`, `scalability`.
   - Security posture throughout: `sast`, `dependencies`, `dast`, `pen`.

4. Verify with the narrowest safe commands first:
   - Run targeted tests before full suites.
   - Treat long-running or disruptive kinds (`load`, `stress`, `spike`, `soak`, `scalability`, `chaos`, `failover`, `dr`, `dast`, `pen`) as explicit operations. Use the local project gate or ask before running them when they can affect infrastructure, cost, production data, or workstation stability.
   - In Sigma Shake, route heavy test commands through the project test gate, for example `bun run gate:test <command>`, and honor local power-safety rules.

5. Report the resilience delta:
   - Summarize added tests, commands run, observed pass/fail status, remaining gaps, and the next missing `test:<kind>` script for each gap.
   - Do not claim a kind is covered because a file exists; require either a meaningful test body, a runnable script, or a recorded sentinel/report.

## Canonical Test Kinds

| Category | Kinds | Use For |
|---|---|---|
| Functional | `unit`, `integration`, `component`, `e2e`, `regression`, `api` | Logic, service boundaries, user journeys, endpoint contracts, and breakage that must never return. |
| Performance and scalability | `load`, `stress`, `spike`, `soak`, `scalability` | Expected traffic, breaking points, bursts, leaks/resource drift, and capacity scaling. |
| Resiliency and infrastructure | `chaos`, `failover`, `dr`, `configuration`, `onboarding` | Fault injection, backup takeover, disaster recovery, env/deploy correctness, and first-run/startup readiness. |
| Security and compliance | `sast`, `dast`, `pen`, `dependencies` | Static weaknesses, dynamic exploit probes, adversarial testing, and third-party vulnerability exposure. |

## Naming Conventions

Prefer `package.json` scripts named `test:<kind>` because the workspace can map them directly. Common examples:

```json
{
  "scripts": {
    "test:unit": "...",
    "test:integration": "...",
    "test:api": "...",
    "test:configuration": "...",
    "test:onboarding": "...",
    "test:failover": "...",
    "test:load": "...",
    "test:audit": "..."
  }
}
```

Use file and directory names the workspace detector can recognize:

- Directories under `test/`, `tests/`, `__tests__/`, or `Tests/`: `unit`, `integration`, `component`, `e2e`, `regression`, `api`, `load`, `stress`, `spike`, `soak`, `scalability`, `chaos`, `failover`, `dr`, `configuration`, `config`, `onboarding`, `startup`, `sast`, `dast`, `pen`, `pentest`, `dependencies`, `audit`.
- JavaScript/TypeScript: `*.kind.test.ts`, `*.kind.spec.ts`, and the same pattern for `tsx`, `js`, `mjs`, or `cjs`.
- Elixir: `*_<kind>_test.exs`.
- Python: `test_<kind>.py` or `test_<kind>_*.py`.
- Rust: `tests/<kind>.rs` or `tests/<kind>_*.rs`.
- Go: `*_<kind>_test.go`.
- Swift and C#: `<Name><Kind>Tests.swift` / `<Name><Kind>Tests.cs`.

Flat tests still count as `unit`, but kind-specific names make resilience gaps visible.

## Acceptance Gates

- Every changed critical path has at least one targeted functional test and one regression guard.
- Every endpoint or external contract has `api` coverage or a documented reason it is not applicable.
- Every startup, config, packaging, or environment change has `configuration` and `onboarding` coverage.
- Every persistence, queue, worker, daemon, scheduler, or multi-region change has at least one recovery-oriented test among `chaos`, `failover`, or `dr`.
- Every performance-sensitive change defines a budget and has the cheapest useful capacity test; long-lived services should add `soak` when leaks or drift are plausible.
- Every auth, permission, input-parsing, secret-handling, or dependency change has relevant security coverage.
- A missing kind is not a failure by itself, but an unexplained missing kind on a production-critical app is a resilience gap.

## Output Format

When asked for a plan or review, return a compact table:

| Risk / Change | Test Kinds | Evidence Present | Gap | Next Test / Command |
|---|---|---|---|---|

Then list:
- Commands run and results.
- Tests added or changed.
- Remaining risks that require manual, staged, or production-like validation.
