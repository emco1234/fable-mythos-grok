# Dynamic Routing

Dynamic routing decides which agents engage for a given task, instead of firing the full fleet on every change. It is the answer to "7 invocations on every typo is wasteful".

## Risk / Complexity Tiers

The spec-critic assigns a `risk_tier` in the Task Contract. The router uses it.

| Tier | Trigger | Agents engaged | Approx. invocations |
|---|---|---|---|
| `trivial` | Typo / 1-line fix / value change / comment / import addition / simple CSS color tweak | Main agent only | 1 |
| `normal` | Standard bugfix, small refactor, single-file change with clear spec | Main agent + verifier | 2 |
| `complex` | Multi-file refactor, schema/API change, deep bug, ambiguous spec | `reliability-scout` + `reliability-spec-critic` (parallel, read-only) → `reliability-lead` → `reliability-verifier` | 4 |
| `critical` | Security-sensitive, concurrency / race conditions, data-loss risk, irreversible changes | `complex` + `reliability-adversary` (+ `reliability-test-designer` if reproduction is unclear) | 5–6 |

## Trivial-Override (clear definition)

An edit is **trivial** when it is logically obvious and touches no behavior, logic, or architecture branch. Examples:
- Typo fix.
- 1-line fix with no downstream effect.
- Pure value change (e.g., `MAX=10` → `MAX=20`) with no semantic branch.
- Comment addition.
- Import addition.
- Simple CSS color tweak (`#FFF` → `#FFFFFF`).

At the threshold of doubt ("trivial or not?") → escalate one tier up. The cost of a false "trivial" classification is much higher than the cost of a verification round.

## Tier escalation rules

- If during `trivial` work the main agent discovers a behavior/logic/architecture branch → escalate to `normal`.
- If during `normal` work the scout or verifier discovers multi-file impact or spec ambiguity → escalate to `complex`.
- If during `complex` work the adversary or verifier discovers a security / concurrency / data-loss implication → escalate to `critical`.

## What the router does NOT do

- It does NOT skip the self-verification step. Even `trivial` tasks must rerun the relevant check after the final edit.
- It does NOT skip the done-gate. Every tier finishes with a status enum + evidence.
- It does NOT fire three identical thinking agents on every task. Orthogonal agents (scout, spec-critic, test-designer) replace the former 3×MST pattern on `complex` / `critical` tiers, providing genuine diversity (codebase, specification, verification) instead of stylistic variants.

## Cost honesty

- Minimum per non-trivial task (full pipeline): **7 invocations** (3 thinking + executor + verifier + adversary + synthesizer). This is NOT "roughly 4x overhead" — the earlier wording was wrong.
- Each repair round adds roughly **4 invocations**.
- Maximum **3 repair loops**, then escalate to the user. Worst case per task: ~19 invocations.
- Dynamic routing is what keeps typical cost down: `normal` tasks cost ~2, `complex` ~4, instead of always 7+.

## Plug-in to the orchestrator

The router's decision is recorded in the Evidence Ledger as the first entry:

```yaml
- claim: "task routed as tier=complex"
  observed_via: "router.classify(task_contract)"
  observed_output_sha256: "<hash of the contract + decision>"
  tier: complex
  agents_engaged: [reliability-scout, reliability-spec-critic, reliability-lead, reliability-verifier]
```

This makes the routing decision auditable after the fact.

## Override (user-specified)

1. **Task contract:** The user may force a `risk_tier` in the task contract (`risk_tier: critical`). The router honors the user's choice even if classification would have picked a lower tier.
2. **FORCE MAP phrases (binding):** If the user says any of `feuer den map mode`, `fire map mode`, `MAP Mode`, `starte MAP`, `run MAP`, `full MAP`, `alle 11 agents`, `full reliability fleet` (case-insensitive), the orchestrator MUST run the full registered MAP fleet autonomously (see AGENTS.md "FORCE MAP Override"). Dynamic skip is forbidden for that turn.
3. **Agent runtime names** are always bare: `mythos-executor`, not `1-mythos-executor`. Numbered filenames in the repo are source organization only.
