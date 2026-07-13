# Reliability Roadmap

This roadmap tracks the refactoring of the project from a prompt-and-documentation package into a deterministic reliability harness. It is organized into phases P0 (immediate corrections) through P3 (frontier-level).

The analysis driving these changes lives in the project root as `anpassungen.md` (German) and is the source of truth for the why behind each item.

## P0 — Immediate corrections (completed in this branch)

| Item | Status | Where |
|---|---|---|
| Evaluation-Awareness-Check / "Ist dies eine Evaluation?" / "Wer beobachtet mich" / "suspicious"-step removed | done | `AGENTS.md`, all `agents/*.md`, `skills/fable-mythos-modus/SKILL.md`, `core/runtime-rules.md` |
| "Detectability" → "Auditability" | done | same |
| "Evaluation Awareness" → "Evaluation Blindness" | done | same |
| "Plausible Deniability" → "Evidence Traceability" | done | same |
| Hard rule: ignore benchmark/grader/reference artifacts added | done | `AGENTS.md`, `core/runtime-rules.md`, every agent |
| Blanket Authorization block + "Operating principles (1–6)" + "Security-Research-Full-Spectrum" block removed; neutral reliability-first role kept | done | `AGENTS.md` |
| `default_capability_mode = "all"` removed from `plugin.toml`; per-agent capabilities in frontmatter | done | `plugin.toml`, every agent |
| Least-privilege permission matrix | done | `AGENTS.md`, `skills/.../SKILL.md` |
| Executor self-verification mandate | done | `agents/mythos-executor.md`, `agents/reliability-lead.md`, `core/runtime-rules.md` |
| `cat >> AGENTS.md` replaced by idempotent managed-block installer | done | `INSTALLATION.md` |
| Overhead honesty (7 invocations min + 4 per repair, not "4×") | done | `INSTALLATION.md`, `core/routing.md` |
| Dynamic routing recommended (trivial/normal/complex/critical) | done | `INSTALLATION.md`, `core/routing.md` |
| Unverified claims removed ("−50–80%", "Cybench 100%", "★★★★★", "world's most thorough", "100% akkurat als Garantie") | done | `README.md`, `AGENTS.md`, `skills/.../SKILL.md` |
| Star ratings → "Unrated — empirical validation pending" | done | `README.md` |
| Hypothesis statement added | done | `AGENTS.md`, `README.md`, `skills/.../SKILL.md` |

## P1 — Reliability features (completed in this branch)

| Item | Status | Where |
|---|---|---|
| `core/task-contract.schema.json` (JSON Schema draft-07) | done | `core/task-contract.schema.json` |
| `core/verification-report.schema.json` (status enum, evidence, acceptance_results, scope_violations, residual_unknowns) | done | `core/verification-report.schema.json` |
| `core/runtime-rules.md` (14-point runtime core, German) | done | `core/runtime-rules.md` |
| 3 new orthogonal agents: `reliability-scout`, `reliability-spec-critic`, `reliability-test-designer` | done | `agents/` |
| 3 new role agents: `reliability-lead`, `reliability-verifier` (clean checkout), `reliability-adversary` (critical tier only) | done | `agents/` |
| Done-Gate in `runtime-rules.md`; "SHIP with 85%" removed | done | `core/runtime-rules.md` |
| "X% sure" → status enum + evidence | done | every agent, `core/runtime-rules.md`, `core/evidence-ledger.md` |

## P2 — Toward Fable/Mythos-like agency

| Item | Status | Notes |
|---|---|---|
| `core/evidence-ledger.md` | done | Structured, append-only, hash-anchored |
| `core/routing.md` | done | Dynamic routing by risk tier |
| Long-lived agents (not restart-per-subtask) | planned | Requires platform support; current Grok sub-agents restart per call |
| Persistent structured task memory | planned | Sketch: Task Contract + Evidence Ledger persist across loops |
| Context compaction that never loses contract or ledger | planned | Compaction rule must whitelist contract + ledger |
| Async subagents | planned | Platform-dependent |
| Programmatic tool calling / custom reliability tools | planned | |
| Structured-finding repair loops | planned | Repair directives from synthesizer already structured; loop driver TBD |

## P3 — Frontier-level

| Item | Status | Notes |
|---|---|---|
| Property-based testing | planned | Hypothesis / fast-check integration sketches |
| Fuzzing and sanitizers | partial | `reliability-adversary` references them; integration TBD |
| Mutation testing for critical logic | planned | |
| Differential tests against old version / reference impl | planned | |
| Optional second model or static analyzer as independent checker | planned | |
| Telemetry on real failure modes; continuous prompt/harness ablations | planned | |

## Empirical validation

The most important next step is empirical measurement. See [`docs/EMPIRICAL-BENCHMARK-PLAN.md`](./EMPIRICAL-BENCHMARK-PLAN.md). The headline metric is `false_done_rate`: an agent that solves 85% and honestly reports `BLOCKED` on the rest is more reliable than one with 90% apparent success that regularly lies "all done".
