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

### Step 2 — Install the Sub-Agents

```bash
mkdir -p ~/.grok/agents
cp agents/*.md ~/.grok/agents/
```

**No manual agent creation needed.** Grok Build CLI **auto-discovers** every `.md` file in `~/.grok/agents/` from its frontmatter (`name`, `description`, `prompt_mode`, `model`, `permission_mode`, `agents_md`). You do **not** need to create agents by hand in the TUI, and there is **no** `agent` block to register in any config file — the copy command above is the entire install. If a guide (older or third-party) tells you to create the 5 / 11 agents manually through the Grok UI, that is outdated: filesystem drop-in is the supported path.

Each agent file declares its capabilities in frontmatter via the **`permission_mode`** field, which names a Grok-managed mode. Grok Build CLI controls tool access through these named modes — **not** through a per-agent tool list. There is **no** `tools:` array in the frontmatter and **no** blanket `default_capability_mode = "all"`. See the permission matrix in [`AGENTS.md`](./AGENTS.md).

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
