# Installation Guide — Reliability Harness for Grok Build CLI

Complete walkthrough to install the reliability-harness plugin in Grok Build CLI.

## Prerequisites

- **[Grok Build CLI](https://x.ai/cli)** installed (v0.2.91+ recommended)
- Authenticated with a Grok account (SuperGrok or X Premium Plus for beta access)
- Git installed

Verify Grok is installed:
```bash
grok --version
```

## Option A — Install as Grok Plugin (Recommended)

This installs the framework as a native Grok plugin in `~/.grok/plugins/`.

```bash
# Clone into Grok's plugins directory
git clone https://github.com/emco1234/fable-mythos-grok.git ~/.grok/plugins/fable-mythos-grok
```

Then in the Grok Build CLI TUI, reload plugins:
```
/plugins reload
```

Verify:
```
/plugins list
```

You should see `fable-mythos-grok` in the list.

## Option B — Manual Installation

If you prefer manual control over each component:

### Step 1 — Install the Skill

```bash
mkdir -p ~/.grok/skills/fable-mythos-modus
cp skills/fable-mythos-modus/SKILL.md ~/.grok/skills/fable-mythos-modus/SKILL.md
```

Verify the frontmatter:
```bash
head -4 ~/.grok/skills/fable-mythos-modus/SKILL.md
```

Expected:
```
---
name: fable-mythos-modus
description: Reliability-first operating mode...
---
```

### Step 2 — Install the Skill (the working control surface in Grok Build Beta)

```bash
mkdir -p ~/.grok/skills/fable-mythos-modus
cp -r skills/fable-mythos-modus/* ~/.grok/skills/fable-mythos-modus/
```

Grok Build CLI **auto-discovers** the skill from `~/.grok/skills/<name>/SKILL.md` (frontmatter `name` must match the directory name). After a Grok restart, `grok inspect` lists it under **Skills**. This is the part that actually shapes the main agent's behavior in the current Grok Build beta — the reliability rules (Task Contract, Evaluation Blindness, Auditability, self-verification, Done Gate) live in the skill body and in `AGENTS.md`, and they apply to every session.

#### Important — honest status of custom subagents in Grok Build (v0.2.x beta)

Grok Build's **built-in** subagents — `general-purpose`, `explore`, `plan` — are what you get in the TUI subagent picker. They are bundled under `~/.grok/bundled/` (with an internal `manifest.json` + `agents/*.md` + `roles/*.toml` layout that is **not documented for third-party plugins**).

This repository ships `agents/*.md` + matching `roles/*.toml` + a `plugin.toml` that declares them under `[plugin.components]`. In the **current Grok Build beta** these are **not yet surfaced as selectable subagents** in the TUI picker: `grok inspect` reports the plugin as enabled with "1 agents" rather than 11, and the official docs do not document a working mechanism for third-party plugins to register custom subagents. The `gsd-*` entries some users see in the picker originate from **skills with workflows**, not from plugin `agents/*.md` files — a different mechanism.

What this means in practice for the current beta:

- The **built-in** subagents (`general-purpose`, `explore`, `plan`) remain the invokable subagents.
- The **`fable-mythos-modus` skill + `AGENTS.md`** (this is what loads reliably) govern how the main agent and those built-in subagents behave — Evaluation Blindness, Auditability, Task Contract, self-verification, Done Gate.
- The 11 `agents/*.md` + `roles/*.toml` files are kept in the repo as a **forward reference**: they are structured exactly like Grok's bundled agents, so if/when Grok Build documents and ships third-party custom subagents, this plugin is already in the right shape and only needs the manifest wiring that the docs will specify.

If a later Grok Build release adds documented third-party subagent support, update this section and remove the beta caveat.

##### What you can verify today

```bash
grok inspect
```

Under **Skills** you should see `fable-mythos-modus`. Under **Agents** you see Grok's built-ins (`general-purpose`, `explore`, `plan`). The reliability behavior is applied via the skill + `AGENTS.md`, not via custom subagent entries.

Also confirm the plugin is enabled in `~/.grok/config.toml`:

```toml
[plugins]
enabled = ["fable-mythos-grok", ...]
```

In most setups the auto-scan of `~/.grok/plugins/` discovers the plugin and adds the entry itself, but if `grok inspect` reports the plugin as disabled, add the name explicitly and reload.

#### How `permission_mode` maps to least-privilege intent

Grok's named modes are the only mechanism that actually gates tool access. The agent files use:

| `permission_mode` | Tools available in Grok | Agents | Original least-privilege intent |
|---|---|---|---|
| `plan` | read-only (read, grep, glob) + planning | `mythos-singleshot-thinking-intelligence`, `mythos-synthesizer`, `reliability-scout`, `reliability-spec-critic` | READ-ONLY — exact match |
| `default` | full toolset (read, edit, write, bash, grep, glob) | `mythos-executor`, `reliability-lead`, `reliability-test-designer` | FULL write — exact match |
| `default` | full toolset (see above) | `mythos-verifier`, `mythos-adversary`, `reliability-verifier`, `reliability-adversary` | intended read+bash-only (see limitation below) |

**Known limitation (honest):** Grok Build CLI does **not** expose a read+exec-only mode. The four verifier/adversary agents conceptually need only `read` + `bash` (run tests/fuzzing in an isolated worktree), but the narrowest Grok mode that includes `bash` is `default`, which also grants edit/write. These agents are therefore pinned to `default` and disciplined by their system prompt to never edit main source. If Grok later adds a read+exec-only mode, these four agents should be moved to it; this is tracked as the closest available approximation.

### Step 3 — Install the Global Rules (idempotent)

**Do not blindly append to `AGENTS.md`** — repeated runs would duplicate rules and grow context unboundedly. Instead, the install uses a managed block delimited by HTML comment markers:

```bash
# Idempotent merge — replaces only the managed block, leaves the rest of
# the user's AGENTS.md untouched.
python3 - <<'PY'
from pathlib import Path
import re

src = Path("AGENTS.md").read_text(encoding="utf-8")
dst_path = Path.home() / ".grok" / "AGENTS.md"
dst_path.parent.mkdir(parents=True, exist_ok=True)

START = "<!-- reliability-harness:start -->"
END = "<!-- reliability-harness:end -->"
block = f"{START}\n{src}\n{END}\n"

existing = dst_path.read_text(encoding="utf-8") if dst_path.exists() else ""
pattern = re.compile(re.escape(START) + r".*?" + re.escape(END) + r"\n?", re.DOTALL)

if pattern.search(existing):
    new = pattern.sub(lambda m: block, existing)
else:
    sep = "\n\n" if existing and not existing.endswith("\n\n") else ""
    new = existing + sep + block

dst_path.write_text(new, encoding="utf-8")
print(f"Merged {len(src)} bytes into {dst_path} (managed block).")
PY
```

Why a managed block?
- Running install twice produces the same file content (idempotent).
- Uninstall is a single regex removal of the block.
- The rest of the user's `AGENTS.md` is never touched.

### Step 4 — Restart Grok Build CLI

Fully quit Grok and restart. Skills, agents, and rules are indexed at startup.

## Verify the Installation

### Check skills are loaded
In the Grok TUI, the skill `fable-mythos-modus` should be available. Test by asking:
```
Summarize the runtime core and the done-gate.
```

The agent should respond with the 14-point runtime core and the done-gate conditions.

### Check agents are available
The sub-agents should be invokable via the `task` tool. The 5 core agents are `mythos-singleshot-thinking-intelligence`, `mythos-executor`, `mythos-verifier`, `mythos-adversary`, `mythos-synthesizer`. The 6 optional orthogonal agents are `reliability-scout`, `reliability-spec-critic`, `reliability-test-designer`, `reliability-lead`, `reliability-verifier`, `reliability-adversary`.

## How the Protocol Fires After Installation

| Task type | Behavior | Invocations |
|---|---|---|
| Coding task with substance | Full pipeline (Phase 0–3) | ≥7 invocations |
| Trivial edit | Skipped (main agent only) | 1 |
| Pure info / read-only questions | Skipped | 1 |
| Ambiguous ("trivial or not?") | Pipeline fires | ≥7 |

### Honest overhead (read this before installing)

- Minimum per non-trivial task: **7 sub-agent invocations** (3 thinking + executor + verifier + adversary + synthesizer). The earlier "roughly 4x overhead" wording was incorrect and is withdrawn.
- Each repair round adds roughly **4 invocations** (executor rework + verifier + adversary + synthesizer).
- Maximum **3 repair loops**, then escalate to the user. Worst case per task: ~19 invocations.
- Tokens and latency scale accordingly.

### Recommended: dynamic routing (not full fleet on every task)

To avoid firing the full fleet on small tasks, route by complexity/risk:

| Tier | Trigger | Agents engaged | Approx. invocations |
|---|---|---|---|
| `trivial` | Typo / 1-line / value change / comment | Main agent only | 1 |
| `normal` | Standard bugfix, small refactor | Main agent + verifier | 2 |
| `complex` | Multi-file refactor, schema/API change, deep bug | 2 read-only scouts (parallel) → lead → verifier | 4 |
| `critical` | Security-sensitive, concurrency, data-loss risk | complex + adversary | 5 |

The full 7-agent pipeline is reserved for the `critical` tier or for tasks the router classifies as genuinely ambiguous. This is the same routing table that appears in [`core/routing.md`](./core/routing.md) and in `AGENTS.md`.

### Trivial-Override (clear definition)

An edit is **trivial** when it is logically obvious and touches no behavior, logic, or architecture branch. Examples: typo fix, 1-line fix, pure value change, comment addition, import addition, simple CSS color tweak. At the threshold of doubt ("trivial or not?") → the pipeline fires.

## Troubleshooting

### Plugin doesn't appear after `/plugins reload`
- Check the directory: `ls ~/.grok/plugins/fable-mythos-grok/`
- Ensure `plugin.toml` is at the root of the plugin directory
- Check file permissions (Grok needs read access)

### Skill not discovered
- Verify `~/.grok/skills/fable-mythos-modus/SKILL.md` exists
- Check the frontmatter: `name` field must match the directory name
- Grok also checks other skill directories — if you have the ZCode version installed there, it may conflict. Either remove it or rely on Grok's deduplication (higher-priority location wins)

### Sub-agents not invokable
- Verify all `.md` files are in `~/.grok/agents/`
- Check frontmatter: each file must have `name`, `description`, `prompt_mode`, `model`, `permission_mode`, `agents_md`. Tool access is governed by `permission_mode` (a Grok named mode such as `plan` or `default`), **not** by a `tools` array — a `tools:` key in the frontmatter is not a Grok-supported field and should be removed.
- Subagents must be enabled (they are by default). Check `~/.grok/config.toml` for `[subagents] enabled = false`
- Do NOT expect an `agent` block in the global config with N entries — agent files in `~/.grok/agents/` are auto-discovered.

### Agents missing from the TUI subagent picker (plugin install)

In the **current Grok Build beta (v0.2.x)**, third-party plugins **cannot** register custom selectable subagents in the TUI picker — the official docs do not document a working mechanism for it. Only Grok's **built-in** subagents (`general-purpose`, `explore`, `plan`) appear in the picker. This is a documented beta limitation, not a configuration error in your install.

What still works and is what you should rely on:
- The **`fable-mythos-modus` skill** loads and shapes main-agent + built-in-subagent behavior (Evaluation Blindness, Auditability, Task Contract, self-verification, Done Gate). Verify with `grok inspect` → the skill appears under **Skills**.
- The **`AGENTS.md`** global rules apply to every session.
- The built-in subagents `general-purpose`, `explore`, `plan` remain available for delegation.

The `agents/*.md` + `roles/*.toml` files shipped here are kept as a **forward reference** in the layout Grok uses for its own bundled agents. If a future Grok Build release documents third-party custom subagents, this plugin is already structured to match and will only need the manifest wiring the docs specify.

Do **not** spend time trying to make the 11 custom agents appear in the picker in the current beta — it is not supported.

### Pipeline fires too aggressively (cost concerns)
- Tighten the trivial-override threshold in `~/.grok/AGENTS.md` (see "Trivial-Override" above).
- Use dynamic routing (see table above) instead of forcing the full 7-agent fleet on every task.

### Duplicate `AGENTS.md` content after reinstall
- You skipped the managed-block install (Step 3) and used `cat AGENTS.md >> ~/.grok/AGENTS.md`. Revert to the managed-block installer; it is idempotent.

## Uninstallation

```bash
# Remove plugin
rm -rf ~/.grok/plugins/fable-mythos-grok

# Or remove manual installation
rm -rf ~/.grok/skills/fable-mythos-modus
rm ~/.grok/agents/mythos-*.md
rm ~/.grok/agents/reliability-*.md

# Remove the managed block from AGENTS.md (idempotent — leaves the rest intact)
python3 - <<'PY'
from pathlib import Path
import re

dst_path = Path.home() / ".grok" / "AGENTS.md"
if not dst_path.exists():
    raise SystemExit(0)
text = dst_path.read_text(encoding="utf-8")
START, END = "<!-- reliability-harness:start -->", "<!-- reliability-harness:end -->"
pattern = re.compile(re.escape(START) + r".*?" + re.escape(END) + r"\n?", re.DOTALL)
new = pattern.sub("", text).rstrip() + "\n"
dst_path.write_text(new, encoding="utf-8")
print(f"Removed managed block from {dst_path}.")
PY

# Reload plugins
/plugins reload
```

## Compatibility with ZCode Version

If you also use [Fable & Mythos for ZCode](https://github.com/emco1234/fable-mythos-zcode), the two installations are independent:
- ZCode reads `~/.zcode/AGENTS.md` and `~/.zcode/skills/`
- Grok reads `~/.grok/AGENTS.md` and `~/.grok/skills/` (with fallback to other agent-framework directories)

No conflicts. You can run both on the same machine.

## Next Steps

- Read [`core/runtime-rules.md`](./core/runtime-rules.md) for the 14-point runtime core
- Read [`core/task-contract.schema.json`](./core/task-contract.schema.json) for the task-contract schema
- Read [`docs/RELIABILITY-ROADMAP.md`](./docs/RELIABILITY-ROADMAP.md) for the phase plan
- Read [`docs/EMPIRICAL-BENCHMARK-PLAN.md`](./docs/EMPIRICAL-BENCHMARK-PLAN.md) for the validation plan
- Read [`docs/ANTI-CONCEALMENT.md`](./docs/ANTI-CONCEALMENT.md) for the philosophy
