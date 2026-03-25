# Notebook Review Checklist

## Instructions

Review notebook code and outputs for the issues listed below. Be specific — cite `Cell N` and suggest fixes. Skip anything that's fine. Only flag real problems.

**Two-pass review:**
- **Pass 1 (CRITICAL):** Data Leakage, Statistical Validity, Reproducibility Blockers. These can invalidate conclusions.
- **Pass 2 (INFORMATIONAL):** Visualization Honesty, Narrative Quality, Code Quality. These improve communication but don't invalidate results.

**Output format:**

```
Notebook Review: N issues (X critical, Y informational)

**CRITICAL** (methodology at risk):
- [Cell N] Problem description
  Impact: how this affects conclusions
  Fix: suggested fix

**Issues** (non-blocking):
- [Cell N] Problem description
  Fix: suggested fix
```

If no issues found: `Notebook Review: No issues found.`

Be terse. For each issue: one line problem, one line impact, one line fix.

---

## Review Categories

### Pass 1 — CRITICAL

#### Data Leakage
- `fit_transform()` or `.fit()` on full dataset before train/test split
- Feature engineering using target variable information (directly or via aggregates)
- Temporal features using future data in training set
- Group members (same patient, same device, same user) appearing in both train and test
- Target encoding without proper regularization (leave-one-out or k-fold within training)
- Feature selection on full dataset before splitting

#### Statistical Validity
- Multiple hypothesis tests (>1) without any correction for multiple comparisons
- P-hacking signals: many models fit but only "best" reported; features added/removed until significance achieved
- Wrong statistical test for data structure (parametric on non-normal, independent on paired, etc.)
- Assumptions of statistical methods not checked (no residual plots, no normality checks, no homogeneity tests)
- Conclusions claiming causation from observational data
- Claims of "no effect" from non-significant results (confusion of absence of evidence with evidence of absence)
- Statistical significance reported without effect sizes

#### Reproducibility Blockers
- Random operations without explicit seeds (`random_state`, `np.random.seed`, `torch.manual_seed`)
- Hardcoded absolute paths (`/Users/`, `/home/`, `C:\`)
- Missing package version documentation
- Non-deterministic operations (GPU float32, parallel aggregation) without acknowledgment
- Data files referenced but not present or not described

### Pass 2 — INFORMATIONAL

#### Visualization Honesty
- Truncated y-axes exaggerating differences
- Missing uncertainty (no error bars, no confidence bands)
- Bar plots for continuous distributions (use violin/box/strip)
- Missing axis labels, units, or legends
- Red/green color schemes (accessibility)
- Dual y-axes creating spurious visual correlations

#### Narrative Quality
- Code cells with no explanatory markdown
- Results produced but never interpreted
- No clear story arc (question → data → methods → results → interpretation)
- Technical jargon without definition for intended audience
- Dead code or abandoned experiments left in final notebook

#### Code Quality
- `warnings.filterwarnings('ignore')` suppressing diagnostic information
- Magic numbers without named constants or explanations
- Copy-paste code that should be functions
- `df.describe()` without interpretation
- `.head()` as the only data inspection

---

## Suppressions — DO NOT flag these

- Style preferences (variable naming, import ordering) unless they affect readability
- "Could use a more efficient implementation" when the current one is correct and fast enough
- Missing docstrings on notebook-internal functions (notebooks are self-documenting by nature)
- Use of deprecated functions when the replacement is functionally identical
- Minor formatting inconsistencies between cells
