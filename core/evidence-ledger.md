# Evidence Ledger

The Evidence Ledger is the structured, append-only record of every observed fact that supports a claim in the verification report. It replaces uncalibrated "X% sure" statements with concrete, reproducible evidence.

## Purpose

- Every claim ("tests pass on clean checkout", "the bug reproduces at base commit", "scope is clean") must point to an entry in the ledger.
- Entries are produced by actually running a command / reading a file / observing a tool — never by speculating.
- A reviewer (human or agent) can reproduce any ledger entry from its `observed_via` command in a matching environment.

## Entry shape (mirrors `core/verification-report.schema.json#evidence`)

```yaml
- claim: "all unit tests pass on clean checkout"
  observed_via: "cd worktree-clean && pnpm test --silent"
  observed_output_sha256: "<64-hex>"
  observed_at: "2026-07-13T12:34:56Z"
  pre_existing: false
```

Required fields:
- `claim` — the factual statement being supported.
- `observed_via` — exact command or action that produced the evidence.
- `observed_output_sha256` — SHA-256 of the observed output. Anchors the claim to a real run.

Optional fields:
- `observed_at` — ISO-8601 timestamp.
- `pre_existing` — true if the behavior was already present at `base_commit`.

## Hard rules

1. **Never claim a result you did not observe.** If you did not run the command, the ledger entry does not exist.
2. **Pre-existing vs. newly caused must be separated.** A failure present at `base_commit` is not caused by the patch; record `pre_existing: true`.
3. **Hash the output, not the expectation.** The SHA-256 covers what the tool actually printed, not what you hoped it would print.
4. **Append-only within a task.** Do not rewrite a ledger entry to make a failed check look passing. If a check is re-run after a repair, both runs stay in the ledger.
5. **The synthesizer cannot override a ledger entry.** A failing entry forces a non-`VERIFIED` status regardless of narrative.

## Relationship to the Done-Gate

`VERIFIED` requires that every MUST-requirement maps to a ledger entry showing success. Any missing or failing entry forces `PARTIALLY_VERIFIED`, `BLOCKED`, or `UNVERIFIED`.

## Relationship to "X% sure"

"X% sure" is deprecated as a completion signal. The replacement is:
- A status enum value (`VERIFIED` / `PARTIALLY_VERIFIED` / `BLOCKED` / `UNVERIFIED`).
- A non-empty `evidence` list.
- An honest `residual_unknowns` list of items that could not be verified.

A reviewer reading the ledger should be able to tell exactly what was checked, what passed, what failed, and what was not checked — without trusting any prose summary.
