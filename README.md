# mlstack

**mlstack turns Claude Code and Codex from one generic assistant into a team of ML/data specialists you can summon on demand.**

Ten opinionated workflow skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and [Codex](https://developers.openai.com/codex). Analysis planning, statistical review, ML architecture review, notebook authoring, exploratory data analysis, feature engineering, model critique, performance optimization, retrospectives, and analysis shipping — all as slash commands.

### Without mlstack
- The agent takes your analysis request literally — it never asks if you're framing the problem correctly
- It will fit whatever model you asked for, even when the data violates that model's assumptions
- "Explore this dataset" gives inconsistent depth every time
- The agent writes code but never explains why it chose that approach over alternatives
- You still do methodology review by hand: check distributions, validate assumptions, eyeball residuals
- Notebooks are walls of code with no narrative thread

### With mlstack

| Skill | Mode | What it does |
|-------|------|-------------|
| `/plan-science-review` | Principal Investigator | Rethink the research question. Challenge framing, identify confounds, map the causal structure before anyone touches data. |
| `/plan-stats-review` | Biostatistician / Methods Lead | Lock in the analysis plan: study design, power analysis, assumption checks, multiple comparison strategy, pre-registration. |
| `/plan-ml-review` | ML Architect | Challenge the modeling strategy across data modalities: architecture choices, representation strategy, training dynamics, compute-performance tradeoffs. |
| `/review-notebook` | Paranoid methods reviewer | Find the statistical sins that pass a cursory glance but invalidate conclusions. Not a style nitpick pass. |
| `/eda` | Senior Data Analyst | Systematic exploratory analysis with structured reports, distribution profiles, missingness maps, and relationship matrices. |
| `/feature-eng` | ML Engineer | Feature engineering, selection, and transformation with domain-aware rationale and leakage detection. |
| `/model-critique` | ML Research Scientist | Adversarial model evaluation. Challenge every modeling choice: was this the right algorithm, the right metric, the right validation strategy? |
| `/review-perf` | ML Performance Engineer | Review training/inference code for efficiency: GPU utilization, data loading throughput, memory footprint, precision, unnecessary computation. |
| `/retro-analysis` | Analytics Manager | Team-aware retrospective: analysis velocity, reproducibility, insight yield, and per-person growth. |
| `/ship-analysis` | Release Analyst | Package analysis into a reproducible deliverable: freeze environment, validate outputs, generate executive summary, archive artifacts. |

## Who this is for

You're a data scientist, ML engineer, or quantitative researcher who uses Claude Code or Codex as a force multiplier. You want your AI assistant to think like a team of specialists — not just write code, but challenge your methodology, catch statistical errors before they become retracted papers, and produce analyses that would survive peer review.

## Install

**Requirements:** [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and/or [Codex](https://developers.openai.com/codex), [Git](https://git-scm.com/).

### Quick install

Clone and run setup:

```bash
git clone https://github.com/tim-krausz/mlstack.git ~/.claude/skills/mlstack
cd ~/.claude/skills/mlstack && ./setup
```

The `setup` script:
- Registers all skills with Claude Code (`~/.claude/skills/`) and Codex (`~/.codex/skills/`)
- Creates or updates `CLAUDE.md` and `AGENTS.md` in your working directory with the skill list
- Dynamically discovers skills from the repo — no hardcoded list to maintain

### Platform-specific install

If you only use one platform:

```bash
# Claude Code only
cd ~/.claude/skills/mlstack && ./setup --claude

# Codex only
git clone https://github.com/tim-krausz/mlstack.git ~/.codex/skills/mlstack
cd ~/.codex/skills/mlstack && ./setup --codex
```

### Add to a project so teammates get it

```bash
cp -Rf ~/.claude/skills/mlstack .claude/skills/mlstack
rm -rf .claude/skills/mlstack/.git
cd .claude/skills/mlstack && ./setup --working-dir ../../../
```

This copies mlstack into your repo and runs setup targeting the project root. The setup script will create or update `CLAUDE.md` and `AGENTS.md` with the skill list.

### What gets installed

- Skill symlinks in `~/.claude/skills/` (Claude Code) and/or `~/.codex/skills/` (Codex)
- `## mlstack skills` section auto-injected into `CLAUDE.md` and `AGENTS.md`
- Nothing touches your PATH or runs in the background

### Coexistence with gstack

mlstack is designed to coexist with [gstack](https://github.com/garrytan/gstack). All skill names are unique across both stacks — no collisions. If you have both installed, all skills from both stacks are available simultaneously.

## Upgrading

```bash
cd ~/.claude/skills/mlstack
git pull origin main
./setup
```

The setup script re-discovers skills and updates all symlinks and documentation files. If you added new skills or renamed existing ones, setup handles it automatically.

To update a project-level install:

```bash
cp -Rf ~/.claude/skills/mlstack .claude/skills/mlstack
rm -rf .claude/skills/mlstack/.git
cd .claude/skills/mlstack && ./setup --working-dir ../../../
```

## Uninstalling

```bash
# Remove skill symlinks and the mlstack directory
cd ~/.claude/skills/mlstack && SKILLS=$(ls -d */ | sed 's/\///g'); cd ~
for s in $SKILLS; do rm -f ~/.claude/skills/$s ~/.codex/skills/$s; done
rm -rf ~/.claude/skills/mlstack ~/.codex/skills/mlstack
```

Then remove the `## mlstack skills` section from your `CLAUDE.md` and `AGENTS.md`.

## Setup script reference

```
Usage: ./setup [OPTIONS]

Options:
  --claude          Register skills with Claude Code only
  --codex           Register skills with Codex only
  --no-docs         Skip updating CLAUDE.md / AGENTS.md
  --working-dir DIR Update CLAUDE.md / AGENTS.md in DIR (default: $PWD)
  --help            Show this help message
```

Without `--claude` or `--codex`, setup registers with all detected platforms.

The script is idempotent — run it as many times as you want. It replaces the `## mlstack skills` section in-place if it already exists, preserving all other content in your documentation files.

## How I use these skills

### The analysis lifecycle

1. **Start with `/plan-science-review`** before touching data. This is your PI asking: "Is this the right question? What would actually constitute evidence? What are the confounders you haven't thought about?" It forces you to articulate the causal model before fitting anything.

2. **Lock in methods with `/plan-stats-review`**. Your biostatistician reviews the analysis plan: sample size adequate? Assumptions checkable? Multiple comparisons handled? This catches the methodological landmines before you step on them.

3. **Choose your modeling strategy with `/plan-ml-review`**. Your ML architect asks: is this the right representation for this data? Is a pretrained backbone appropriate? Is this architecture justified for this dataset size? Should you learn features end-to-end or hand-engineer them? This catches the modeling decisions that are invisible to classical statistics.

4. **Explore with `/eda`**. Systematic exploration — not just `df.describe()`. Distribution profiling, missingness analysis, multicollinearity checks, target leakage scans. Produces a structured report you can reference throughout the project.

5. **Engineer features with `/feature-eng`**. Domain-aware feature creation with explicit rationale for every transformation. Leakage detection built in. Tracks which features survived selection and why.

6. **Critique with `/model-critique`** after training. Your adversarial reviewer asks: "Was nested cross-validation actually necessary here? Did you check if a simple baseline beats this? Your pipeline assumed normality but these features violate that assumption." This is the review that saves you from publishing embarrassing results.

7. **Optimize with `/review-perf`** on your training and inference code. Your performance engineer profiles the implementation and finds the bottlenecks: GPU sitting idle waiting on the dataloader, full FP32 weights that should be BF16, redundant forward passes on frozen backbones. Every suggestion comes with an estimated speedup and a concrete code fix.

8. **Review notebooks with `/review-notebook`** before sharing. Catches the sins: p-hacking patterns, undisclosed multiple comparisons, leaky preprocessing, conclusions that don't follow from the evidence.

9. **Ship with `/ship-analysis`**. Freeze the environment, validate all outputs reproduce, generate an executive summary, archive everything. Your analysis is now a citable, reproducible artifact.

10. **Reflect with `/retro-analysis`**. Track analysis velocity, reproducibility rate, insight yield, and methodology quality over time.

## License

MIT
