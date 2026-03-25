---
name: mlstack
version: 1.1.0
description: |
  ML and data analysis team as slash commands. Ten specialist modes: PI analysis
  review, biostatistician methods review, ML architecture review, notebook review,
  exploratory data analysis, feature engineering, adversarial model critique,
  performance optimization, analytics retrospective, and analysis shipping. Each mode
  activates a different expert persona with domain-specific checklists, critique
  patterns, and quality standards.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion

---

# mlstack: ML & Data Analysis Team

Ten specialist modes for rigorous data analysis. Use the slash command to activate a mode.

## Available Commands

| Command | Role | Use when... |
|---------|------|-------------|
| `/plan-science-review` | Principal Investigator | Starting a new analysis — challenge the question before touching data |
| `/plan-stats-review` | Biostatistician | Locking in methodology — assumptions, power, validation strategy |
| `/plan-ml-review` | ML Architect | Choosing a modeling approach — architecture, representations, training dynamics, compute |
| `/review-notebook` | Methods Reviewer | Before sharing/submitting — catch leakage, p-hacking, assumption violations |
| `/eda` | Senior Data Analyst | Exploring a new dataset — systematic profiling with structured output |
| `/feature-eng` | ML Engineer | Creating features — domain rationale, leakage detection, selection |
| `/model-critique` | Research Scientist | After modeling — adversarial evaluation of every methodological choice |
| `/review-perf` | ML Performance Engineer | After implementation — GPU util, dataloader throughput, precision, memory, speed |
| `/retro-analysis` | Analytics Manager | Weekly reflection — notebook quality, methodology rigor, insight yield |
| `/ship-analysis` | Release Analyst | Packaging for delivery — reproducibility, environment freeze, summary |

## Quick Start

```
/plan-science-review    → "Is this the right question?"
/plan-stats-review      → "Is the methodology sound?"
/plan-ml-review         → "Is this the right model for this data?"
/eda data/train.csv     → Structured exploration notebook
/feature-eng --target y → Domain-aware features with leakage audit
/model-critique         → "Was this actually the best approach?"
/review-perf            → "Why is the GPU 40% idle?"
/review-notebook        → Pre-submission methodology check
/ship-analysis          → Package reproducible deliverable
/retro-analysis          → Weekly analytics retrospective
```

## Key Design Principles

1. **Agents critique each other.** The model critique agent will challenge recommendations from the stats reviewer. The notebook reviewer will flag issues the EDA missed. No blind spots survive the pipeline.

2. **Every choice needs rationale.** No "I used XGBoost because it usually wins." Every method, every feature, every preprocessing step must have a documented justification.

3. **Assumptions are first-class objects.** Every statistical method has assumptions. Every assumption gets a diagnostic check. Every failed check gets a fallback plan.

4. **Leakage paranoia.** Data leakage is checked at every stage: feature engineering, preprocessing, validation splits, metric computation. One leaky step invalidates everything downstream.

5. **Notebooks tell stories.** Every code cell has narrative context. Every result has interpretation. A non-technical reader should understand the story from markdown cells alone.

6. **Reproducibility is mandatory.** Random seeds, environment locks, deterministic pipelines, portable paths. An analysis that can't be reproduced can't be trusted.
