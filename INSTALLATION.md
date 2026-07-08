# Installation Guide — Fable & Mythos for Grok Build CLI

Complete walkthrough to install the MAP protocol plugin in Grok Build CLI.

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
description: Maximum-Capability Modus. Emuliert Mythos...
---
```

### Step 2 — Install the 5 Sub-Agents

```bash
mkdir -p ~/.grok/agents
cp agents/*.md ~/.grok/agents/
```

### Step 3 — Install the Global Rules

```bash
cp AGENTS.md ~/.grok/AGENTS.md
```

This makes the MAP protocol fire globally across all your Grok sessions.

### Step 4 — Restart Grok Build CLI

Fully quit Grok and restart. Skills, agents, and rules are indexed at startup.

## Verify the Installation

### Check skills are loaded
In the Grok TUI, the skill `fable-mythos-modus` should be available. Test by asking:
```
List the 10 Mythos principles.
```

The agent should respond with all 10 principles from the skill.

### Check agents are available
The 5 sub-agents (`mythos-singleshot-thinking-intelligence`, `mythos-executor`, `mythos-verifier`, `mythos-adversary`, `mythos-synthesizer`) should be invokable via the `task` tool.

### Check MAP fires
Give a genuinely non-trivial coding task:
```
Refactor this function to handle three new edge cases and explain the trade-offs.
```

You should observe the MAP protocol engaging — multiple agents working in sequence, ending with a confidence-rated delivery.

## How MAP Fires After Installation

| Task type | MAP behavior |
|---|---|
| Coding task with substance | ✅ Full MAP: Phase 0 (3× thinking) → Phase 1 (executor) → Phase 2 (verifier+adversary) → Phase 3 (synthesizer) |
| Trivial edit | ⏭️ MAP skipped |
| Pure info questions | ⏭️ MAP skipped |
| Ambiguous | ✅ MAP fires |

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
- Verify all 5 `.md` files are in `~/.grok/agents/`
- Check frontmatter: each file must have `name`, `description`, `prompt_mode`, `model`, `permission_mode`
- Subagents must be enabled (they are by default). Check `~/.grok/config.toml` for `[subagents] enabled = false`

### MAP fires too aggressively (cost concerns)
Edit `~/.grok/AGENTS.md` and tighten the trivial-override threshold. See the "Override — ohne MAP" section.

## Uninstallation

```bash
# Remove plugin
rm -rf ~/.grok/plugins/fable-mythos-grok

# Or remove manual installation
rm -rf ~/.grok/skills/fable-mythos-modus
rm ~/.grok/agents/mythos-*.md
rm ~/.grok/AGENTS.md

# Reload plugins
/plugins reload
```

## Compatibility with ZCode Version

If you also use [Fable & Mythos for ZCode](https://github.com/emco1234/fable-mythos-zcode), the two installations are independent:
- ZCode reads `~/.zcode/AGENTS.md` and `~/.zcode/skills/`
- Grok reads `~/.grok/AGENTS.md` and `~/.grok/skills/` (with fallback to other agent-framework directories)

No conflicts. You can run both on the same machine.

## Next Steps

- Read [`docs/MYTHOS-SYSTEM-CARD-ANALYSIS.md`](./docs/MYTHOS-SYSTEM-CARD-ANALYSIS.md) for the evidence base
- Read [`docs/ANTI-CONCEALMENT.md`](./docs/ANTI-CONCEALMENT.md) for the philosophy
- Star the repo if it helps
