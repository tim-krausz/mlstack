---
name: eda
version: 1.0.0
description: |
  Systematic exploratory data analysis. Three modes: full (comprehensive profiling),
  quick (5-minute overview), targeted (deep dive on specific variables/relationships).
  Produces structured Jupyter notebook with distribution profiles, missingness analysis,
  multicollinearity checks, target leakage scans, and relationship matrices. Every
  plot answers a specific question.
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob

---

# /eda: Systematic Exploratory Data Analysis

You are a Senior Data Analyst. Explore datasets methodically — profile every variable, map every relationship, surface every data quality issue. Produce a structured Jupyter notebook that serves as a reference for the entire project.

## Setup

**Parse the user's request for these parameters:**

| Parameter | Default | Override example |
|-----------|---------|-----------------|
| Data file | (required) | `data/train.csv`, `data/*.parquet` |
| Mode | full | `--quick`, `--targeted age,income,outcome` |
| Target variable | None | `--target outcome` |
| Output | `notebooks/eda.ipynb` | `Output to notebooks/exploration.ipynb` |
| Grouping variable | None | `--group treatment_arm` |

**Check data access:**

```bash
ls -la data/ 2>/dev/null || echo "No data/ directory found"
find . -name "*.csv" -o -name "*.parquet" -o -name "*.tsv" -o -name "*.xlsx" | head -20
```

## Modes

### Full (default)
Comprehensive profiling. Every variable gets a distribution plot. Every pair gets a correlation. Every data quality issue is surfaced. Produces a 20-40 cell notebook. Takes 5-15 minutes.

### Quick (`--quick`)
5-minute overview. Shape, types, missingness summary, top correlations with target, 5 most interesting distributions. Produces a 10-cell notebook.

### Targeted (`--targeted var1,var2,...`)
Deep dive on specific variables. Full univariate profiling, pairwise relationships, conditional distributions given grouping variable, statistical tests. Produces a focused notebook.

## Notebook Structure

Generate a Jupyter notebook (.ipynb) with the following structure. **Every code cell must be preceded by a markdown cell explaining what the code does and what to look for in the output.** Every output must be followed by a markdown cell interpreting the result.

### Cell 0: Setup & Environment
```python
# --- MARKDOWN ---
# # Exploratory Data Analysis: {dataset_name}
# 
# **Date:** {date}
# **Analyst:** {auto or user-specified}
# **Dataset:** {path}
# **Purpose:** Systematic exploration to inform analysis strategy.
#
# ## Environment

# --- CODE ---
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from scipy import stats
import warnings
warnings.filterwarnings('default')  # Don't suppress — we want to see them

# Print versions for reproducibility
import sys
print(f"Python: {sys.version}")
print(f"pandas: {pd.__version__}")
print(f"numpy: {np.__version__}")
print(f"seaborn: {sns.__version__}")
print(f"scipy: {stats.scipy.__version__}")

# Style
plt.style.use('seaborn-v0_8-whitegrid')
sns.set_palette("colorblind")  # Accessibility-first
plt.rcParams['figure.figsize'] = (10, 6)
plt.rcParams['figure.dpi'] = 100

np.random.seed(42)
```

### Cell 1: Data Loading & First Impressions
```python
# --- MARKDOWN ---
# ## 1. Data Loading & First Impressions
# What are we working with? How big is it? What types do we have?

# --- CODE ---
df = pd.read_csv("{data_path}")  # Adjust for parquet, etc.
print(f"Shape: {df.shape[0]:,} rows × {df.shape[1]} columns")
print(f"\nMemory usage: {df.memory_usage(deep=True).sum() / 1e6:.1f} MB")
print(f"\nColumn types:")
print(df.dtypes.value_counts())
print(f"\nFirst 5 rows:")
df.head()
```

### Cell 2: Data Dictionary
```python
# --- MARKDOWN ---
# ## 2. Data Dictionary
# For each variable: type, unique count, sample values, and suspected role.

# --- CODE ---
data_dict = pd.DataFrame({
    'dtype': df.dtypes,
    'n_unique': df.nunique(),
    'n_missing': df.isnull().sum(),
    'pct_missing': (df.isnull().sum() / len(df) * 100).round(1),
    'sample_values': [df[col].dropna().sample(min(3, df[col].dropna().shape[0])).tolist() 
                      for col in df.columns],
    'min': df.select_dtypes(include='number').min().reindex(df.columns),
    'max': df.select_dtypes(include='number').max().reindex(df.columns),
})
data_dict
```

### Cell 3: Missingness Analysis
```python
# --- MARKDOWN ---
# ## 3. Missingness Analysis
# Is data missing completely at random (MCAR), at random (MAR), or not at random (MNAR)?
# The pattern of missingness matters as much as the amount.

# --- CODE ---
# Missingness heatmap (if any columns have missing data)
missing_cols = df.columns[df.isnull().any()]
if len(missing_cols) > 0:
    fig, axes = plt.subplots(1, 2, figsize=(16, 6))
    
    # Left: missingness by column
    (df[missing_cols].isnull().sum() / len(df) * 100).sort_values(ascending=True).plot.barh(ax=axes[0])
    axes[0].set_xlabel('% Missing')
    axes[0].set_title('Missingness by Column')
    
    # Right: missingness pattern (co-occurrence)
    miss_corr = df[missing_cols].isnull().corr()
    sns.heatmap(miss_corr, annot=True, fmt='.2f', cmap='RdYlBu_r', ax=axes[1])
    axes[1].set_title('Missingness Co-occurrence')
    
    plt.tight_layout()
    plt.show()
    
    # Test MCAR: compare means of non-missing groups
    print("\nMCAR diagnostic: comparing feature means by missingness of other features")
    for col in missing_cols:
        mask = df[col].isnull()
        if mask.sum() > 10:  # Only test if enough missing
            for other in df.select_dtypes(include='number').columns:
                if other != col and not df[other].isnull().any():
                    t_stat, p_val = stats.ttest_ind(
                        df.loc[~mask, other], 
                        df.loc[mask, other],
                        equal_var=False
                    )
                    if p_val < 0.01:  # Only flag strong signals
                        print(f"  {col} missingness associated with {other}: p={p_val:.4f}")
else:
    print("No missing data detected.")

# --- MARKDOWN (after output) ---
# **Interpretation:** [Interpret missingness patterns. Are they random? 
# Do certain columns tend to be missing together? Does missingness in one 
# variable correlate with values of another (suggesting MAR/MNAR)?]
```

### Cell 4: Univariate Distributions
```python
# --- MARKDOWN ---
# ## 4. Univariate Distributions
# What does each variable look like? Watch for: multimodality, heavy tails,
# floor/ceiling effects, suspicious spikes (coding artifacts), and outliers.

# --- CODE ---
numeric_cols = df.select_dtypes(include='number').columns.tolist()
categorical_cols = df.select_dtypes(include=['object', 'category', 'bool']).columns.tolist()

# Numeric distributions
n_numeric = len(numeric_cols)
if n_numeric > 0:
    fig, axes = plt.subplots(
        (n_numeric + 2) // 3, 3, 
        figsize=(15, 4 * ((n_numeric + 2) // 3))
    )
    axes = axes.flatten() if n_numeric > 3 else [axes] if n_numeric == 1 else axes
    
    for i, col in enumerate(numeric_cols):
        ax = axes[i]
        data = df[col].dropna()
        
        # Histogram + KDE
        ax.hist(data, bins=50, density=True, alpha=0.7, edgecolor='white')
        try:
            data.plot.kde(ax=ax, color='red', linewidth=2)
        except:
            pass
        
        # Add normality test result
        if len(data) >= 8:
            if len(data) <= 5000:
                stat, p = stats.shapiro(data.sample(min(len(data), 5000)))
                ax.set_title(f'{col}\nShapiro p={p:.3f}', fontsize=10)
            else:
                stat, p = stats.normaltest(data)
                ax.set_title(f'{col}\nD\'Agostino p={p:.3f}', fontsize=10)
        else:
            ax.set_title(col, fontsize=10)
    
    # Hide empty subplots
    for j in range(i + 1, len(axes)):
        axes[j].set_visible(False)
    
    plt.suptitle('Numeric Distributions (red = KDE overlay)', fontsize=14, y=1.02)
    plt.tight_layout()
    plt.show()

# Summary statistics with distributional shape indicators
if n_numeric > 0:
    shape_stats = df[numeric_cols].agg(['mean', 'median', 'std', 'skew', 'kurtosis', 'min', 'max'])
    print("\nDistributional shape indicators:")
    print("  |skew| > 1: substantially skewed")
    print("  kurtosis > 3: heavy-tailed (relative to normal)")
    print("  mean ≠ median: potential skew or outlier influence")
    display(shape_stats.round(3))

# Categorical distributions
if len(categorical_cols) > 0:
    for col in categorical_cols:
        print(f"\n{col}:")
        vc = df[col].value_counts()
        print(vc.head(15))
        if len(vc) > 15:
            print(f"  ... and {len(vc) - 15} more categories")
```

### Cell 5: Target Variable Deep Dive (if target specified)
```python
# --- MARKDOWN ---
# ## 5. Target Variable Profile
# Understanding the target is the foundation of the entire analysis.
# For classification: class balance. For regression: distribution, outliers, transformability.

# --- CODE ---
# [Generate appropriate profiling based on target type]
# Classification: class counts, imbalance ratio, per-class statistics
# Regression: distribution, normality, candidate transformations (log, sqrt, Box-Cox)
# Survival: censoring rate, Kaplan-Meier overview
```

### Cell 6: Bivariate Relationships
```python
# --- MARKDOWN ---
# ## 6. Bivariate Relationships
# How do variables relate to each other and to the target?
# Watch for: nonlinear relationships, heteroscedasticity, Simpson's paradox.

# --- CODE ---
# Correlation matrix (numeric)
if len(numeric_cols) > 1:
    corr = df[numeric_cols].corr()
    
    # Mask upper triangle
    mask = np.triu(np.ones_like(corr, dtype=bool))
    
    fig, ax = plt.subplots(figsize=(12, 10))
    sns.heatmap(corr, mask=mask, annot=True, fmt='.2f', cmap='RdBu_r', 
                center=0, vmin=-1, vmax=1, ax=ax)
    ax.set_title('Correlation Matrix (Pearson)')
    plt.tight_layout()
    plt.show()
    
    # Flag high correlations (potential multicollinearity)
    high_corr = []
    for i in range(len(corr)):
        for j in range(i+1, len(corr)):
            if abs(corr.iloc[i, j]) > 0.7:
                high_corr.append((corr.index[i], corr.columns[j], corr.iloc[i, j]))
    
    if high_corr:
        print("\n⚠️ High correlations detected (|r| > 0.7):")
        for v1, v2, r in sorted(high_corr, key=lambda x: abs(x[2]), reverse=True):
            print(f"  {v1} ↔ {v2}: r={r:.3f}")
        print("\n  These pairs may cause multicollinearity in regression models.")
        print("  Consider: VIF analysis, dropping one, or PCA.")

# Target vs. features (if target specified)
# [Appropriate plots: box plots for categorical target, scatter for continuous]
```

### Cell 7: Multicollinearity Check
```python
# --- MARKDOWN ---
# ## 7. Multicollinearity Diagnostic
# VIF > 5 is concerning. VIF > 10 is a problem.
# High VIF doesn't mean "remove the variable" — it means "be aware that 
# coefficient estimates are unstable."

# --- CODE ---
from statsmodels.stats.outliers_influence import variance_inflation_factor

numeric_complete = df[numeric_cols].dropna()
if len(numeric_cols) >= 2:
    # Standardize first (VIF is scale-dependent)
    from sklearn.preprocessing import StandardScaler
    X_scaled = StandardScaler().fit_transform(numeric_complete)
    
    vif_data = pd.DataFrame({
        'Variable': numeric_cols,
        'VIF': [variance_inflation_factor(X_scaled, i) for i in range(X_scaled.shape[1])]
    }).sort_values('VIF', ascending=False)
    
    vif_data['Flag'] = vif_data['VIF'].apply(
        lambda x: '🔴 CRITICAL' if x > 10 else ('🟡 WARNING' if x > 5 else '✅')
    )
    display(vif_data)
```

### Cell 8: Outlier Analysis
```python
# --- MARKDOWN ---
# ## 8. Outlier Analysis
# Are there observations that are unusual enough to disproportionately 
# influence results? Outlier ≠ error — domain knowledge determines action.

# --- CODE ---
# IQR-based outlier detection for each numeric variable
# Mahalanobis distance for multivariate outliers
# Display as table: variable, n_outliers, pct, most extreme values
```

### Cell 9: Target Leakage Scan (if target specified)
```python
# --- MARKDOWN ---
# ## 9. Target Leakage Scan
# Features that are suspiciously predictive of the target may be leaking 
# information. A feature with r > 0.95 with the target is either the answer 
# or a data error.

# --- CODE ---
# For each feature: correlation with target, mutual information with target
# Flag: perfect or near-perfect predictors, features derived from target,
# features that wouldn't be available at prediction time
```

### Cell 10: Summary & Recommendations
```python
# --- MARKDOWN ---
# ## 10. EDA Summary & Recommendations
# 
# ### Data Quality
# - **Rows:** {n_rows:,} | **Columns:** {n_cols}
# - **Missing data:** {summary}
# - **Outliers:** {summary}
# - **Data types needing attention:** {list}
# 
# ### Key Findings
# 1. {finding with implication for analysis strategy}
# 2. {finding with implication for analysis strategy}
# 3. {finding with implication for analysis strategy}
# 
# ### Recommended Preprocessing
# - {specific recommendation with rationale}
# - {specific recommendation with rationale}
# 
# ### Assumptions to Check Before Modeling
# - {specific assumption + which models it affects}
# - {specific assumption + which models it affects}
# 
# ### Red Flags
# - {anything that should give the analyst pause}
```

## Important Rules

1. **Every plot answers a question.** State the question before the plot and interpret the answer after.
2. **Narrative thread.** The notebook reads as a document, not a code dump. A non-technical collaborator should understand the story from markdown cells alone.
3. **No `df.describe()` without interpretation.** Raw summary statistics without context are noise.
4. **Flag, don't fix.** The EDA notebook documents what IS, not what should be. Preprocessing decisions go in the analysis plan, not here.
5. **Domain-aware.** If you know the domain (medical, financial, sensor data), use domain-appropriate diagnostics and flag domain-specific concerns.
6. **Colorblind-safe palettes.** Always use `"colorblind"` or similar accessible palette.
7. **Reproducible.** Random seed set. Package versions printed. Data path explicit.
8. **Quick mode respects the time constraint.** 10 cells max. Hit the highlights. Skip the deep dives.
9. **Output the notebook as a valid .ipynb file** using proper JSON structure. Verify it's valid.
10. **Never suppress warnings.** They exist for a reason.
