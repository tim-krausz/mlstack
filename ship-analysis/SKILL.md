---
name: ship-analysis
version: 1.0.0
description: |
  Ship analysis workflow: validate reproducibility, freeze environment, verify all
  outputs regenerate, generate executive summary, archive artifacts, commit and push.
  For a completed analysis, not for deciding what to analyze.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion

---

# Ship: Analysis Packaging & Delivery

You are running the `/ship-analysis` workflow. This packages a completed analysis into a reproducible, shareable deliverable. The user said `/ship-analysis` which means the analysis is done — your job is to validate, package, and deliver.

**Only stop for:**
- No notebooks found (abort)
- Reproducibility failures (stop, show what failed)
- Critical methodology issues found during validation (ask user)
- Missing data files that can't be located

**Never stop for:**
- Minor formatting issues (auto-fix)
- Missing environment file (auto-generate)
- Missing random seeds (flag but continue)

---

## Step 1: Pre-flight

1. Identify all analysis artifacts:
```bash
echo "=== Notebooks ==="
find . -name "*.ipynb" -not -path "./.ipynb_checkpoints/*" | sort

echo "=== Python scripts ==="
find . -name "*.py" -not -path "./env/*" -not -path "./.venv/*" | sort

echo "=== Data files ==="
find . -name "*.csv" -o -name "*.parquet" -o -name "*.tsv" -o -name "*.xlsx" 2>/dev/null | sort

echo "=== Model artifacts ==="
find . -name "*.pkl" -o -name "*.joblib" -o -name "*.h5" -o -name "*.pt" 2>/dev/null | sort

echo "=== Environment files ==="
ls requirements*.txt environment*.yml pyproject.toml Pipfile 2>/dev/null

echo "=== Documentation ==="
ls README.md ANALYSIS_PLAN.md CHANGELOG.md 2>/dev/null
```

2. If no notebooks or scripts found, **abort**: "No analysis artifacts found."

---

## Step 2: Environment Freeze

1. Check if environment specification exists:
```bash
ls requirements*.txt environment*.yml pyproject.toml 2>/dev/null
```

2. If no environment file exists, generate one:
```bash
pip freeze > requirements.txt 2>/dev/null
# Or for conda:
# conda env export > environment.yml
```

3. Validate the environment file captures all imports:
```bash
# Extract all imports from notebooks and scripts
grep -rh "^import \|^from " --include="*.py" 2>/dev/null | sort -u
python3 -c "
import json, re, glob
imports = set()
for f in glob.glob('**/*.ipynb', recursive=True):
    try:
        nb = json.load(open(f))
        for cell in nb['cells']:
            if cell['cell_type'] == 'code':
                for line in ''.join(cell['source']).split('\n'):
                    m = re.match(r'^(?:import|from)\s+(\w+)', line)
                    if m:
                        imports.add(m.group(1))
    except: pass
print('\n'.join(sorted(imports)))
" 2>/dev/null
```

Flag any import not in the requirements file.

---

## Step 3: Reproducibility Validation

For each notebook, attempt to re-execute:

```bash
# Run notebook top-to-bottom
jupyter nbconvert --to notebook --execute --ExecutePreprocessor.timeout=600 \
    --output executed_<name>.ipynb <notebook.ipynb> 2>&1
```

If execution fails:
- Show the error with cell number
- AskUserQuestion: A) Fix and retry, B) Ship with warning, C) Skip this notebook

If execution succeeds:
- Compare outputs: are key numerical results identical?
- Flag any non-deterministic outputs (different random results, timestamps)

---

## Step 4: Quick Methodology Scan

Run a fast scan for critical issues (not a full review — that should have been done already):

```bash
# Leakage patterns
grep -rn "fit_transform\|\.fit(" --include="*.ipynb" --include="*.py" | head -20

# Check if train_test_split comes AFTER preprocessing
# Warning suppression
grep -rn "warnings.filterwarnings.*ignore" --include="*.ipynb" --include="*.py"

# Missing seeds
python3 -c "
import json, glob
for f in glob.glob('**/*.ipynb', recursive=True):
    try:
        nb = json.load(open(f))
        code = ' '.join([''.join(c['source']) for c in nb['cells'] if c['cell_type']=='code'])
        has_random = 'random' in code or 'Random' in code or 'seed' in code.lower()
        has_seed = 'random_state' in code or 'random.seed' in code or 'manual_seed' in code
        if has_random and not has_seed:
            print(f'⚠️ {f}: Uses randomness but no seed found')
    except: pass
"
```

If CRITICAL issues found (data leakage patterns), AskUserQuestion.
If only warnings, note them and continue.

---

## Step 5: Generate Executive Summary

Create `SUMMARY.md` with:

```markdown
# Analysis Summary

## Question
[Extracted from notebook markdown or ANALYSIS_PLAN.md]

## Key Findings
1. [Finding with effect size and confidence interval]
2. [Finding with effect size and confidence interval]
3. [Finding with effect size and confidence interval]

## Methods
- **Data:** [description, n observations, n features]
- **Primary method:** [method name, brief justification]
- **Validation:** [strategy used]
- **Key metrics:** [metric = value (95% CI: lower, upper)]

## Limitations
- [Limitation 1]
- [Limitation 2]

## Reproducibility
- **Environment:** [requirements.txt / environment.yml]
- **Random seeds:** [set / not set]
- **Re-execution:** [passed / failed with notes]
- **Runtime:** [estimated total execution time]

## Artifacts
| File | Description | Status |
|------|-------------|--------|
| notebooks/eda.ipynb | Exploratory analysis | ✅ Validated |
| notebooks/modeling.ipynb | Model training & evaluation | ✅ Validated |
| models/final_model.pkl | Trained model artifact | ✅ Present |
| requirements.txt | Environment specification | ✅ Generated |

## How to Reproduce
1. Clone this repository
2. Create environment: `pip install -r requirements.txt`
3. Run notebooks in order: eda.ipynb → modeling.ipynb
4. Expected runtime: ~X minutes on [hardware description]
```

---

## Step 6: Archive & Organize

Ensure clean project structure:
```
project/
├── README.md                    # Project overview
├── SUMMARY.md                   # Executive summary (generated)
├── ANALYSIS_PLAN.md             # Pre-registered analysis plan
├── requirements.txt             # Environment specification
├── data/
│   ├── raw/                     # Original data (never modified)
│   └── processed/               # Cleaned data
├── notebooks/
│   ├── 01_eda.ipynb             # Exploratory analysis
│   ├── 02_feature_engineering.ipynb
│   └── 03_modeling.ipynb
├── src/                         # Reusable pipeline code
│   ├── features.py
│   ├── models.py
│   └── evaluation.py
├── models/                      # Trained model artifacts
└── reports/                     # Generated figures and tables
    └── figures/
```

Create any missing directories. Move files into canonical locations if needed (AskUserQuestion for ambiguous cases).

---

## Step 7: Commit & Push

```bash
# Stage everything
git add -A

# Create descriptive commit
git commit -m "$(cat <<'EOF'
analysis: [brief description of the analysis]

Summary: [1-2 sentence description of key findings]
Methods: [primary method used]
Validation: [validation strategy]
Reproducibility: [passed/warnings]

Artifacts: N notebooks, N scripts, N model files
Environment: requirements.txt generated
EOF
)"

# Push
git push -u origin $(git branch --show-current)
```

---

## Step 8: Final Report

Output to the user:

```
=== ANALYSIS SHIPPED ===

📊 Summary: [one-line description]
📁 Artifacts: N notebooks, N scripts, N model files
✅ Reproducibility: [passed / N warnings]
📋 Executive summary: SUMMARY.md
🔗 [commit hash or PR URL]

Key findings:
1. [finding]
2. [finding]
3. [finding]

Methodology notes:
- [any warnings or limitations flagged during shipping]
```

---

## Important Rules

- **Never skip reproducibility validation.** If notebooks can't re-execute, that's a shipping blocker.
- **Never modify analysis results.** You package what exists, you don't change conclusions.
- **Auto-generate what's missing** (requirements.txt, SUMMARY.md) but don't overwrite what exists.
- **Flag but don't block on methodology.** Full methodology review should have been done with `/review-notebook` or `/model-critique`. This is a final sanity check, not a full review.
- **The goal is: user says `/ship-analysis`, next thing they see is a validated, packaged, reproducible deliverable.**
