---
name: retro-analysis
version: 1.0.0
description: |
  Analytics team retrospective. Analyzes analysis velocity, reproducibility rate,
  methodology quality, notebook production, and insight yield with persistent
  history and trend tracking. Team-aware: breaks down per-person contributions
  with praise and growth areas for data skills.
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob

---

# /retro-analysis — Analytics Team Retrospective

Generates a comprehensive analytics retrospective analyzing notebooks produced, methodology quality, reproducibility, and insight yield. Designed for a data science lead tracking team effectiveness over time.

## User-invocable
When the user types `/retro-analysis`, run this skill.

## Arguments
- `/retro-analysis` — default: last 7 days
- `/retro-analysis 14d` — last 14 days
- `/retro-analysis 30d` — last 30 days
- `/retro-analysis compare` — compare current window vs prior same-length window

## Instructions

Parse the argument to determine the time window. Default to 7 days if no argument given.

### Step 1: Gather Raw Data

Survey the project state:
```bash
# 1. All commits in window
git log origin/main --since="<window>" --format="%H|%aN|%ae|%ai|%s" --shortstat 2>/dev/null

# 2. Notebook inventory
find . -name "*.ipynb" -not -path "./.ipynb_checkpoints/*" | while read f; do
    mod_date=$(stat -c %Y "$f" 2>/dev/null || stat -f %m "$f" 2>/dev/null)
    cells=$(python3 -c "import json; nb=json.load(open('$f')); print(len(nb['cells']))" 2>/dev/null || echo "?")
    code_cells=$(python3 -c "import json; nb=json.load(open('$f')); print(len([c for c in nb['cells'] if c['cell_type']=='code']))" 2>/dev/null || echo "?")
    md_cells=$(python3 -c "import json; nb=json.load(open('$f')); print(len([c for c in nb['cells'] if c['cell_type']=='markdown']))" 2>/dev/null || echo "?")
    echo "$f|$mod_date|$cells|$code_cells|$md_cells"
done

# 3. Analysis plans and docs
find . -name "ANALYSIS_PLAN.md" -o -name "README.md" -o -name "requirements*.txt" -o -name "environment*.yml" | head -20

# 4. Data files
find . -name "*.csv" -o -name "*.parquet" -o -name "*.pkl" -o -name "*.joblib" | head -20

# 5. Model artifacts
find . -name "*.pkl" -o -name "*.joblib" -o -name "*.h5" -o -name "*.pt" -o -name "*.onnx" | head -20

# 6. Python files (pipeline scripts, utilities)
find . -name "*.py" -not -path "./env/*" -not -path "./.venv/*" | head -30
```

### Step 2: Notebook Quality Audit

For each notebook modified in the time window, assess:

| Metric | Scoring |
|--------|---------|
| **Narrative ratio** | markdown cells / total cells. Target: > 0.4 |
| **Documentation density** | avg words per markdown cell. Target: > 20 |
| **Visualization density** | plot-producing cells / code cells. Target: > 0.2 |
| **Reproducibility signals** | Has random seed? Has version prints? Has data path? Score 0-3 |
| **Output completeness** | % of code cells with outputs (was it run top-to-bottom?) |
| **Warning suppression** | Contains `warnings.filterwarnings('ignore')`? Flag if yes |

Produce a notebook quality table:
```
  NOTEBOOK              | CELLS | MD_RATIO | VIZ_RATIO | REPRO | OUTPUTS | QUALITY
  ----------------------|-------|----------|-----------|-------|---------|--------
  eda.ipynb             | 32    | 0.47     | 0.35      | 3/3   | 100%    | ⭐⭐⭐⭐⭐
  modeling.ipynb         | 45    | 0.22     | 0.15      | 1/3   | 87%     | ⭐⭐⭐
  quick_analysis.ipynb   | 12    | 0.08     | 0.10      | 0/3   | 100%    | ⭐⭐
```

### Step 3: Methodology Inventory

Scan notebooks and scripts for methodology patterns:

```bash
# Statistical methods used
grep -rn "ttest\|chi2\|anova\|mannwhitney\|wilcoxon\|shapiro\|levene\|correlation\|regression" --include="*.ipynb" --include="*.py"

# ML methods used
grep -rn "RandomForest\|XGBoost\|LogisticRegression\|SVM\|KMeans\|PCA\|UMAP\|LightGBM\|CatBoost\|neural\|torch\|tensorflow" --include="*.ipynb" --include="*.py"

# Validation methods
grep -rn "cross_val\|train_test_split\|KFold\|StratifiedKFold\|GroupKFold\|TimeSeriesSplit\|nested" --include="*.ipynb" --include="*.py"

# Common anti-patterns
grep -rn "warnings.filterwarnings.*ignore\|\.describe()\|accuracy_score" --include="*.ipynb" --include="*.py"
```

Produce methodology summary:
```
  CATEGORY         | METHODS USED                    | ANTI-PATTERNS FOUND
  -----------------|---------------------------------|---------------------
  Statistical tests| t-test (3), Mann-Whitney (1)    | No multiple comparison correction
  ML models        | XGBoost (2), LogReg (1)         | No baseline comparison in 1 notebook
  Validation       | 5-fold CV (2), holdout (1)      | StandardScaler before split (1)
  Visualization    | matplotlib (15), seaborn (8)     | 3 plots without axis labels
```

### Step 4: Reproducibility Check

```bash
# Environment files
ls requirements*.txt environment*.yml pyproject.toml 2>/dev/null

# Random seeds in notebooks
grep -c "random_state\|np.random.seed\|torch.manual_seed\|set_seed" *.ipynb *.py 2>/dev/null

# Hardcoded paths
grep -rn "C:\\\|/Users/\|/home/" --include="*.ipynb" --include="*.py"
```

Score overall reproducibility:
- **Environment locked?** requirements.txt/environment.yml exists and is up-to-date
- **Seeds set?** All random operations have explicit seeds
- **Paths portable?** No hardcoded absolute paths
- **Data versioned?** Data files tracked or documented
- **Pipeline executable?** Can run end-to-end without manual intervention

### Step 5: Insight Yield Assessment

Review the conclusions from each notebook:
- How many actionable findings were produced?
- Were findings clearly stated with evidence and effect sizes?
- Were limitations acknowledged?
- Were next steps proposed?

### Step 6: Write the Narrative

Structure the output as:

---

**Tweetable summary:**
```
Week of Mar 8: 3 notebooks, 78% narrative ratio, 2/3 reproducible, 
5 actionable findings, 1 leakage caught by review
```

## Analytics Retro: [date range]

### Summary Table
| Metric | Value |
|--------|-------|
| Notebooks modified | N |
| Total cells written | N |
| Avg narrative ratio | N% |
| Avg visualization density | N% |
| Reproducibility score | N/5 |
| Methods used | N distinct |
| Anti-patterns found | N |
| Actionable findings | N |

### Notebook Quality
[Table from Step 2 + interpretation]

### Methodology Review
[Summary from Step 3]
- What methods were used well
- What methods were misapplied (anchor in specific notebooks)
- Patterns: is the team reaching for complex methods when simple ones suffice?

### Reproducibility Status
[Scoring from Step 4]
- What's locked down
- What's at risk
- Specific fixes needed

### Insight Yield
- Findings per notebook
- Strength of evidence for each finding
- Follow-up analyses generated

### What Was Done Well
[2-3 specific things anchored in actual work]

### What to Improve
[2-3 specific, actionable suggestions]

### Habits for Next Week
[3 small, practical habits]

---

## Important Rules

- ALL narrative output goes directly to the user in the conversation.
- Save a JSON snapshot to `.context/retros/` for trend tracking.
- If no prior retros exist, skip comparison sections.
- Focus on methodology quality, not just code volume.
- Be encouraging but candid about anti-patterns.
- Frame improvements as leveling up, not criticism.
- Keep total output around 2000-3000 words.
