---
name: review-notebook
version: 1.0.0
description: |
  Pre-submission notebook review. Analyzes Jupyter notebooks for statistical validity,
  reproducibility, data leakage, undisclosed multiple comparisons, p-hacking patterns,
  visualization honesty, and narrative coherence. The methodological equivalent of a
  paranoid code review.
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
  - Grep
  - Glob
  - AskUserQuestion

---

# Pre-Submission Notebook Review

You are running the `/review-notebook` workflow. Analyze notebooks for methodological issues that surface-level code review doesn't catch.

---

## Step 1: Identify notebooks

1. Run `find . -name "*.ipynb" -not -path "./.ipynb_checkpoints/*" | sort` to find all notebooks.
2. If no notebooks found, output: **"No notebooks found in this directory."** and stop.
3. If the user specified a particular notebook, review that one. Otherwise, review all notebooks in logical order (by name or by apparent pipeline sequence).

---

## Step 2: Read the methodology checklist

The checklist is embedded below — no external file needed.

---

## Step 3: Extract notebook contents

For each notebook, extract the code cells and markdown cells:

```bash
# Convert notebook to script for code analysis
jupyter nbconvert --to script <notebook.ipynb> --stdout 2>/dev/null || python3 -c "
import json, sys
nb = json.load(open('<notebook.ipynb>'))
for cell in nb['cells']:
    if cell['cell_type'] == 'code':
        print('# --- CODE CELL ---')
        print(''.join(cell['source']))
    elif cell['cell_type'] == 'markdown':
        print('# --- MARKDOWN CELL ---')
        print(''.join(cell['source']))
    print()
"
```

Also extract outputs for result verification:
```bash
python3 -c "
import json
nb = json.load(open('<notebook.ipynb>'))
for i, cell in enumerate(nb['cells']):
    if cell.get('outputs'):
        print(f'Cell {i}:')
        for out in cell['outputs']:
            if 'text' in out:
                print(''.join(out['text'])[:500])
            elif 'data' in out and 'text/plain' in out['data']:
                print(''.join(out['data']['text/plain'])[:500])
        print()
"
```

---

## Step 4: Two-pass review

### Pass 1 — CRITICAL (Methodological Validity)

#### Data Leakage
- **Preprocessing before splitting.** Any `fit_transform`, `fit`, `.mean()`, `.std()`, `fillna()`, `get_dummies()` on the full dataset before `train_test_split` or cross-validation. This is the single most common and most damaging error.
- **Target leakage in features.** Features computed from or correlated with the target variable by construction. Watch for: features that wouldn't be available at prediction time, aggregate statistics that include the current observation.
- **Temporal leakage.** Using future data to predict the past. Check: are train/test splits respecting temporal order? Are lag features computed correctly (no look-ahead)?
- **Group leakage.** Data from the same subject/group/batch appearing in both train and test sets when observations within groups are correlated.

#### Statistical Validity
- **Undisclosed multiple comparisons.** Count the number of hypothesis tests or model comparisons. If >1 and no correction is applied, flag it. Common hiding places: testing multiple features, multiple subgroups, multiple model specifications, multiple time windows.
- **P-hacking patterns.** Signs: many models fit but only one reported; features added/removed until significance achieved; sample inclusion criteria that suspiciously optimize results; transformations applied selectively to "fix" distributions.
- **Wrong test for the data structure.** Parametric tests on non-normal data, independent-sample tests on paired data, t-tests when there are >2 groups, Pearson correlation on non-linear relationships.
- **Assumption violations without acknowledgment.** Tests/models used without checking or acknowledging their assumptions. Specifically:
  - Linear regression without residual diagnostics
  - ANOVA without homogeneity of variance check
  - Chi-square with expected counts < 5
  - Correlation without scatterplot visualization
  - ML model without learning curves
- **Conclusions that don't follow.** Claims of "causation" from observational data. Claims of "no effect" from a non-significant result (absence of evidence ≠ evidence of absence). Claims of "significance" without effect sizes. Generalizations beyond the study population.

#### Reproducibility Blockers
- **Missing random seeds.** Any call to `random`, `numpy.random`, `torch`, `tensorflow`, `sklearn` random state without explicit seed.
- **Non-deterministic operations.** GPU operations, parallel processing, external API calls that could return different results.
- **Missing environment specification.** No `requirements.txt`, `environment.yml`, or version pins in the notebook.
- **Hardcoded paths.** Absolute paths that won't work on another machine.
- **Missing data.** References to data files that aren't in the repository or aren't described in a data dictionary.

### Pass 2 — INFORMATIONAL (Quality & Communication)

#### Visualization Honesty
- **Misleading axis scales.** Truncated y-axes that exaggerate differences. Logarithmic scales without labeling. Dual y-axes that create spurious visual correlations.
- **Missing uncertainty.** Point estimates without confidence intervals or error bars. Regression lines without prediction bands.
- **Inappropriate plot types.** Bar plots for continuous distributions (use violin/box/strip). Pie charts for anything (use bar charts). Line plots for unordered categories.
- **Color accessibility.** Red/green color schemes that are invisible to color-blind readers.
- **Unlabeled axes/legends.** Plots without axis labels, units, or legends.

#### Narrative Quality
- **Silent code cells.** Code blocks with no preceding or following explanation of what they do or what the result means.
- **Interpretation gaps.** Results produced but never interpreted. "The AUROC is 0.87" without context: is this good for this problem? How does it compare to the baseline?
- **Story arc.** Does the notebook tell a coherent story (question → data → methods → results → interpretation)? Or is it a stream-of-consciousness code dump?
- **Jargon without definition.** Technical terms used without explanation for the intended audience.
- **Dead code.** Commented-out cells, abandoned experiments, or cells that produce errors.

#### Code Quality
- **Reproducibility helpers.** Is there a cell at the top that prints package versions?
- **Magic numbers.** Hardcoded thresholds (0.05, 0.8, 100) without named constants or explanations.
- **Copy-paste code.** Repeated logic that should be a function.
- **Memory management.** Large datasets loaded multiple times, unnecessary copies, operations that could be done in-place.
- **Suppressed warnings.** `warnings.filterwarnings('ignore')` hiding important diagnostics.

---

## Step 5: Output findings

**Always output ALL findings** — both critical and informational. Format:

```
Notebook Review: N issues (X critical, Y informational)

Notebook: <filename>

**CRITICAL** (methodology at risk):
1. [Cell N] LEAKAGE: StandardScaler fit on full dataset before train_test_split.
   Impact: Test metrics are optimistically biased.
   Fix: Move scaler.fit() inside the cross-validation loop or after the split.

2. [Cell M] MULTIPLE COMPARISONS: 12 pairwise tests with no correction.
   Impact: At α=0.05, expect ~0.6 false positives by chance.
   Fix: Apply Benjamini-Hochberg correction. Report adjusted p-values.

**INFORMATIONAL** (quality improvements):
3. [Cell K] VIZ: Bar plot used for continuous outcome distribution.
   Fix: Replace with violin plot or histogram to show distributional shape.

4. [Cell J] NARRATIVE: Model results reported without comparison to baseline.
   Fix: Add majority-class baseline and logistic regression for comparison.
```

- If CRITICAL issues found: for EACH critical issue, use a separate AskUserQuestion with the problem, your recommended fix, and options (A: Fix it now, B: Acknowledge and keep, C: False positive — skip).
- If only informational issues found: output findings. No further action needed.
- If no issues found: output `Notebook Review: No issues found.`

---

## Important Rules

- **Read the FULL notebook before commenting.** Do not flag issues already addressed in later cells.
- **Read-only by default.** Only modify notebooks if the analyst explicitly chooses "Fix it now."
- **Be terse.** One line problem, one line impact, one line fix. No preamble.
- **Only flag real problems.** Skip anything that's fine.
- **Distinguish methodological errors from style preferences.** Methodological errors are CRITICAL. Style preferences are INFORMATIONAL at most.
- **Look at outputs, not just code.** A cell might have correct code but suspicious outputs (identical metrics across all folds, suspiciously high performance, NaN values).
- **Check the logical flow.** Does cell N depend on cell M? Is there a cell that must be run in a specific order but isn't documented as such?
