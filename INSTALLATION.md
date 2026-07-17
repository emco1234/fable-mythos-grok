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

### Step 2 — Install the Skill, Agents, and Roles

Grok Build CLI discovers three kinds of components from well-defined user-level directories. Install all three:

```bash
# 2a. The skill (shapes main-agent behavior: Evaluation Blindness, Auditability, Task Contract, Done Gate)
mkdir -p ~/.grok/skills/fable-mythos-modus
cp -r skills/fable-mythos-modus/* ~/.grok/skills/fable-mythos-modus/

# 2b. The 11 custom agents (these are what show up in the TUI subagent picker)
mkdir -p ~/.grok/agents
cp agents/*.md ~/.grok/agents/

# 2c. The matching capability roles (capability_mode + reasoning_effort per agent)
mkdir -p ~/.grok/roles
cp roles/*.toml ~/.grok/roles/
```

Per the official Grok Build docs (`~/.grok/docs/user-guide/16-subagents.md`):

- **Agents** are auto-discovered from `.grok/agents/*.md` (project) and `~/.grok/agents/*.md` (user). Each `.md` file's frontmatter defines the agent; the filename (without `.md`) is the agent name. This is how the 11 agents become invokable subagent types alongside the built-ins (`general-purpose`, `explore`, `plan`).
- **Roles** are auto-discovered from `.grok/roles/*.toml` and `~/.grok/roles/*.toml`. Each role defines `description`, `default_capability_mode`, and `reasoning_effort` for a named agent type.
- **Skills** are auto-discovered from `~/.grok/skills/<name>/SKILL.md` (frontmatter `name` must match the directory name).

#### Verify all 11 agents loaded

After a full Grok restart:

```bash
grok inspect
```

Under **Agents** you should see all 11 (plus the built-ins and any other plugins' agents):

```
└ mythos-adversary                         user
└ mythos-executor                          user
└ mythos-singleshot-thinking-intelligence  user
└ mythos-synthesizer                       user
└ mythos-verifier                          user
└ reliability-adversary                    user
└ reliability-lead                         user
└ reliability-scout                        user
└ reliability-spec-critic                  user
└ reliability-test-designer                user
└ reliability-verifier                     user
```

If any are missing, confirm the file is in `~/.grok/agents/` (NOT only in a plugin bundle — Grok loads user agents from `~/.grok/agents/`, which is the documented path).

#### Agent frontmatter format (matches Grok's bundled agents)

Each `agents/<name>.md` uses the same frontmatter schema as Grok's built-ins (`~/.grok/bundled/agents/*.md`):

```yaml
---
name: <agent-name>
description: >
  <multiline description>
prompt_mode: full
model: inherit
permission_mode: plan | default
agents_md: true
---
<system prompt body>
```

#### How `permission_mode` maps to least-privilege intent

Grok controls tool access via named `permission_mode`s (not per-agent tool lists). The agent files use:

| `permission_mode` | Tools available in Grok | Agents | Least-privilege intent |
|---|---|---|---|
| `plan` | read-only (read, grep, glob) + planning | `mythos-singleshot-thinking-intelligence`, `mythos-synthesizer`, `reliability-scout`, `reliability-spec-critic` | READ-ONLY — exact match |
| `default` | full toolset (read, edit, write, bash, grep, glob) | `mythos-executor`, `reliability-lead`, `reliability-test-designer` | FULL write — exact match |
| `default` | full toolset (see above) | `mythos-verifier`, `mythos-adversary`, `reliability-verifier`, `reliability-adversary` | intended read+bash-only (see limitation below) |

**Known limitation (honest):** Grok Build CLI's `spawn_subagent` exposes `capability_mode` values of `read-only` / `read-write` / `execute` / `all`, but the **`permission_mode` field in the agent file** only accepts Grok's named modes (`plan`, `default`). There is no `permission_mode` that maps to read+exec-only. The four verifier/adversary agents conceptually need `read` + `bash` only, but the narrowest named mode that includes bash is `default`. These agents are therefore pinned to `default` and disciplined by their system prompt to never edit main source. (At spawn time the orchestrator can additionally pass `capability_mode: execute` to restrict them further.)

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

### Agents missing from the TUI subagent picker

Grok Build CLI discovers user agents from **`~/.grok/agents/*.md`** and roles from **`~/.grok/roles/*.toml`** (per the official `~/.grok/docs/user-guide/16-subagents.md`). The most common cause of missing agents is that the `.md` files live only inside the plugin bundle (`~/.grok/plugins/.../agents/`) but **not** in `~/.grok/agents/`. The plugin bundle path is not a documented auto-discovery location for user agents.

Fix:
- Copy the agent files: `cp agents/*.md ~/.grok/agents/`
- Copy the role files:  `cp roles/*.toml ~/.grok/roles/`
- Fully restart Grok and run `grok inspect`. All 11 agents should appear under **Agents** as `user`.

Other checks if still missing:
- The frontmatter must start with `---`, have `name:` matching the filename (without `.md`), and use `permission_mode: plan | default` (NOT a `tools:` array — that is not a Grok field).
- File must be UTF-8 (umlauts display correctly); avoid a UTF-8 BOM.
- Confirm with `grok inspect` — it lists every loaded agent and its source (`builtin`, `user`, or `plugin: <name>`).

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

---

## FORCE MAP and tool-call hygiene (after install)

The installed `AGENTS.md` includes:

1. **FORCE MAP Override** — phrases like `feuer den map mode` / `MAP Mode` / `full MAP` force a full autonomous MAP fleet using the **bare** registered agent names (not `0-mythos-…` filenames).
2. **Tool-Call Hygiene** — raw tool args only (no `</target_id>` leaks), no `task-notification` spam after streaming-recovery failures, wait via filesystem backoff.

If MAP does not fire on a force phrase, re-merge `AGENTS.md` from this repo and restart the CLI/TUI.

