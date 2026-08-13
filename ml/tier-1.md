# ML Tier -1 — Data Science Toolkit (Staff-Level Depth)
*NumPy · Pandas · Matplotlib · Seaborn · Scikit-Learn*
*Every method, pattern, gotcha, and real-world workflow*

---

# Chapter 1: NumPy — The Foundation of All ML

## Why NumPy Exists

Python lists are slow because each element is a full Python object stored at a random memory location. NumPy arrays are fast because elements are stored as raw numbers in a contiguous block — operations run in compiled C, not interpreted Python.

```python
import numpy as np

# Speed comparison
python_list = list(range(1_000_000))
numpy_array = np.arange(1_000_000)

# Python: ~200ms to square all elements
result = [x**2 for x in python_list]

# NumPy: ~2ms (100x faster)
result = numpy_array ** 2
```

## Creating Arrays

```python
# From lists
a = np.array([1, 2, 3, 4, 5])                        # 1D, shape (5,)
m = np.array([[1, 2, 3], [4, 5, 6]])                  # 2D, shape (2, 3)

# Zeros, ones, filled
np.zeros((3, 4))                                       # 3×4 of 0.0
np.ones((2, 3))                                        # 2×3 of 1.0
np.full((3, 3), 7)                                     # 3×3 filled with 7
np.eye(4)                                              # 4×4 identity matrix
np.empty((2, 2))                                       # 2×2 uninitialized (fast, garbage values)

# Sequences
np.arange(0, 10, 2)                                    # [0, 2, 4, 6, 8]
np.linspace(0, 1, 5)                                   # [0, 0.25, 0.5, 0.75, 1.0]
np.logspace(-3, 3, 7)                                  # [0.001, 0.01, 0.1, 1, 10, 100, 1000]

# Random
np.random.seed(42)                                     # reproducibility
np.random.rand(3, 4)                                   # uniform [0, 1)
np.random.randn(3, 4)                                  # standard normal (mean=0, std=1)
np.random.randint(0, 100, size=(3, 4))                 # integers [0, 100)
np.random.choice(["a", "b", "c"], size=5, replace=True, p=[0.5, 0.3, 0.2])
np.random.permutation(10)                              # random permutation of 0-9
rng = np.random.default_rng(42)                        # modern API (preferred)
rng.standard_normal((3, 4))

# Data types
a = np.array([1, 2, 3], dtype=np.float32)             # 32-bit float
a = np.array([1.5, 2.7], dtype=np.int32)              # truncates! [1, 2]
a.astype(np.float64)                                    # convert type
```

## Shape Manipulation

```python
a = np.arange(12)         # [0, 1, ..., 11], shape (12,)

a.reshape(3, 4)            # 3×4, must total 12
a.reshape(2, -1)           # 2×6, -1 = auto-calculate
a.reshape(-1, 1)           # 12×1, column vector (critical for sklearn!)
a.reshape(1, -1)           # 1×12, row vector

# GOTCHA: reshape returns a VIEW (shares memory, changes propagate)
b = a.reshape(3, 4)
b[0, 0] = 99
print(a[0])                # 99! (a was modified too)

# Use .copy() for independent copy
b = a.reshape(3, 4).copy()

# Add/remove dimensions
a = np.array([1, 2, 3])   # shape (3,)
a[np.newaxis, :]           # shape (1, 3) — sklearn needs 2D for single samples
a[:, np.newaxis]           # shape (3, 1)
np.expand_dims(a, axis=0)  # shape (1, 3)
np.squeeze(a.reshape(1,3,1)) # removes all size-1 dimensions → shape (3,)

# Flatten
a.ravel()                  # 1D VIEW (shares memory)
a.flatten()                # 1D COPY (independent)

# Transpose
m = np.array([[1,2,3],[4,5,6]])  # (2,3)
m.T                        # (3,2)

# Stack/Concatenate
np.vstack([a, b])          # stack vertically (add rows)
np.hstack([a, b])          # stack horizontally (add columns)
np.concatenate([a, b], axis=0)  # along rows
np.concatenate([a, b], axis=1)  # along columns
np.column_stack([col1, col2])   # stack 1D arrays as columns
```

## Indexing, Slicing, and Boolean Masking

```python
a = np.array([10, 20, 30, 40, 50, 60])

# Basic
a[0], a[-1]                # 10, 60
a[1:4]                     # [20, 30, 40]
a[::2]                     # [10, 30, 50] (every 2nd)
a[::-1]                    # reversed

# 2D
m = np.arange(12).reshape(3, 4)
# [[ 0  1  2  3],
#  [ 4  5  6  7],
#  [ 8  9 10 11]]

m[1, 2]                    # 6 (row 1, col 2)
m[0:2, 1:3]                # [[1,2],[5,6]] (submatrix)
m[:, 0]                    # [0, 4, 8] (first column)
m[-1]                      # [8, 9, 10, 11] (last row)

# Boolean indexing (THE most important operation for data filtering)
a = np.array([10, -5, 30, -15, 50])
mask = a > 0                # [True, False, True, False, True]
a[mask]                     # [10, 30, 50]
a[a > 0]                    # same, shorthand
a[(a > 0) & (a < 40)]       # [10, 30] — use & not 'and', wrap in parens
a[~(a > 0)]                 # [-5, -15] — ~ is NOT

# Fancy indexing
a[[0, 2, 4]]                # [10, 30, 50] — pick specific indices

# np.where (conditional)
np.where(a > 0, a, 0)       # [10, 0, 30, 0, 50] — replace negatives with 0
np.where(a > 0)              # (array([0, 2, 4]),) — INDICES where condition is True

# GOTCHA: Boolean indexing returns a COPY, basic indexing returns a VIEW
b = a[a > 0]
b[0] = 999                  # does NOT modify a (it's a copy)

c = a[0:3]
c[0] = 999                  # DOES modify a (it's a view!)
```

## Math Operations

```python
a = np.array([1.0, 4.0, 9.0, 16.0])

# Element-wise (all ops work element-by-element)
a + 10                # [11, 14, 19, 26]
a * 2                 # [2, 8, 18, 32]
a ** 0.5              # [1, 2, 3, 4] — square root
np.sqrt(a)            # same
np.exp(a)             # [e¹, e⁴, e⁹, e¹⁶]
np.log(a)             # [0, 1.39, 2.20, 2.77] — natural log
np.log2(a)            # [0, 2, 3.17, 4]
np.log1p(a)           # log(1+a) — safe for a near 0
np.abs(np.array([-1, -2, 3]))  # [1, 2, 3]
np.clip(a, 2, 10)     # [2, 4, 9, 10] — clip to range
np.round(np.array([1.45, 2.55]), 1)  # [1.4, 2.6]

# Aggregations
a.sum()               # 30
a.mean()              # 7.5
a.std()               # standard deviation
a.var()               # variance
a.min(), a.max()      # 1.0, 16.0
a.argmin(), a.argmax() # 0, 3 (INDEX of min/max)
np.median(a)          # 6.5
np.percentile(a, 75)  # 75th percentile

# Axis parameter (CRITICAL for 2D)
m = np.array([[1, 2], [3, 4], [5, 6]])  # (3, 2)
m.sum(axis=0)         # [9, 12] — sum each COLUMN (collapse rows)
m.sum(axis=1)         # [3, 7, 11] — sum each ROW (collapse columns)
m.mean(axis=0)        # [3, 4] — mean of each column

# GOTCHA: axis=0 means "along rows" = collapse rows = PER COLUMN result
# Think: axis=0 operates DOWN, axis=1 operates ACROSS
```

## Linear Algebra

```python
a = np.array([[1, 2], [3, 4]])
b = np.array([[5, 6], [7, 8]])

# Matrix multiplication
a @ b                       # [[19, 22], [43, 50]]
np.matmul(a, b)             # same
np.dot(a, b)                # same for 2D

# GOTCHA: * is ELEMENT-WISE, not matrix multiplication!
a * b                       # [[5, 12], [21, 32]] — NOT what you want for linear algebra

# Dot product (vectors)
v1, v2 = np.array([1, 2, 3]), np.array([4, 5, 6])
np.dot(v1, v2)              # 32

# Norms
np.linalg.norm(v1)          # L2 norm: √14 ≈ 3.74
np.linalg.norm(v1, ord=1)   # L1 norm: 6

# Cosine similarity (the RAG metric)
cos_sim = np.dot(v1, v2) / (np.linalg.norm(v1) * np.linalg.norm(v2))

# Matrix operations
np.linalg.inv(a)            # inverse
np.linalg.det(a)            # determinant (-2)
vals, vecs = np.linalg.eig(a)  # eigenvalues, eigenvectors
U, S, Vt = np.linalg.svd(a)    # SVD
np.linalg.solve(a, np.array([1, 2]))  # solve Ax = b
np.trace(a)                 # 5 (sum of diagonal)
```

## Broadcasting (How Mismatched Shapes Work)

```python
# The rule: dimensions are compared RIGHT to LEFT
# They match if: equal, or one is 1

m = np.ones((3, 4))          # (3, 4)
v = np.array([1, 2, 3, 4])   # (4,) → treated as (1, 4)
m + v                         # (3, 4) — v is broadcast to each row

col = np.array([[10], [20], [30]])  # (3, 1)
m + col                       # (3, 4) — col is broadcast to each column

# THIS is what happens in neural networks:
# z = X @ W + b
# X@W shape: (batch, outputs), b shape: (outputs,)
# b broadcasts across all batch rows automatically

# GOTCHA: shapes that DON'T broadcast
# (3, 4) + (3,)   → ERROR! 4 ≠ 3
# Fix: (3, 4) + (3, 1) → works (broadcast column)
```

## Common Patterns in ML

```python
# One-hot encoding
labels = np.array([0, 2, 1, 0, 3])
one_hot = np.eye(4)[labels]
# [[1,0,0,0], [0,0,1,0], [0,1,0,0], [1,0,0,0], [0,0,0,1]]

# Softmax
def softmax(x):
    e_x = np.exp(x - np.max(x))   # subtract max for numerical stability
    return e_x / e_x.sum()

softmax(np.array([2.0, 1.0, 0.1]))  # [0.659, 0.242, 0.099]

# Feature normalization
X_normalized = (X - X.mean(axis=0)) / X.std(axis=0)  # z-score normalization

# Cosine similarity matrix (for embedding comparison)
def cosine_similarity_matrix(X):
    norms = np.linalg.norm(X, axis=1, keepdims=True)
    X_normalized = X / norms
    return X_normalized @ X_normalized.T

# Train/test split (manual)
n = len(X)
indices = np.random.permutation(n)
train_idx = indices[:int(0.8 * n)]
test_idx = indices[int(0.8 * n):]
X_train, X_test = X[train_idx], X[test_idx]
```

---

# Chapter 2: Pandas — Data Manipulation

## Core Objects

```python
import pandas as pd

# Series: 1D labeled array (one column)
s = pd.Series([10, 20, 30], index=["a", "b", "c"], name="values")
s["a"]           # 10
s.values         # numpy array: [10, 20, 30]
s.index          # Index(['a', 'b', 'c'])

# DataFrame: 2D labeled table
df = pd.DataFrame({
    "name": ["Alice", "Bob", "Charlie", "Diana", "Eve"],
    "age": [30, 25, 35, 28, 22],
    "salary": [70000, 55000, 90000, 65000, 48000],
    "dept": ["Eng", "Mkt", "Eng", "Mkt", "Eng"],
    "joined": pd.to_datetime(["2020-01-15", "2021-06-01", "2019-03-20", 
                               "2022-09-10", "2023-02-28"]),
})
```

## Reading and Writing Data

```python
# Reading (with common options)
df = pd.read_csv("data.csv",
    sep=",",                    # delimiter
    header=0,                   # row number for column names (None if no header)
    names=["col1", "col2"],     # custom column names
    usecols=["name", "age"],    # only read these columns (memory efficient)
    dtype={"age": int},         # force data types
    parse_dates=["joined"],     # parse as datetime
    na_values=["N/A", "missing", ""],  # treat these as NaN
    nrows=1000,                 # only read first 1000 rows (for testing)
    encoding="utf-8",
)

# Reading large files in chunks
for chunk in pd.read_csv("huge_file.csv", chunksize=10000):
    process(chunk)    # process 10K rows at a time

# Writing
df.to_csv("output.csv", index=False)      # index=False to avoid saving row numbers
df.to_parquet("output.parquet")            # 10x smaller, 10x faster than CSV
df.to_excel("output.xlsx", index=False)
df.to_json("output.json", orient="records")
```

## Exploring Data (First Thing You Do)

```python
df.shape                    # (5, 5) — rows, columns
df.head(3)                  # first 3 rows
df.tail(2)                  # last 2 rows
df.sample(3)                # 3 random rows
df.info()                   # dtypes, non-null counts, memory
df.describe()               # statistics for numeric columns
df.describe(include="object")  # for string columns (count, unique, top, freq)
df.dtypes                   # type per column
df.columns.tolist()         # column names as list
df.nunique()                # unique values per column
df["dept"].value_counts()   # count per category
df["dept"].value_counts(normalize=True)  # as percentages
df.isnull().sum()           # NaN count per column
df.duplicated().sum()       # duplicate row count
df.memory_usage(deep=True)  # memory per column in bytes
df.corr(numeric_only=True)  # correlation matrix
```

## Selecting and Filtering

```python
# Columns
df["name"]                           # Series
df[["name", "age"]]                  # DataFrame (multiple columns)
df.select_dtypes(include="number")   # only numeric columns
df.select_dtypes(exclude="object")   # exclude strings

# Rows by position
df.iloc[0]                           # first row as Series
df.iloc[0:3]                         # rows 0-2
df.iloc[[0, 3, 4], [0, 2]]          # specific rows and columns by position

# Rows by label
df.loc[0]                            # row with index label 0
df.loc[0:3, ["name", "salary"]]     # rows 0-3, specific columns

# Filtering (boolean indexing)
df[df["age"] > 28]
df[df["dept"] == "Eng"]
df[(df["age"] > 25) & (df["salary"] > 60000)]    # AND: use &, wrap in ()
df[(df["dept"] == "Eng") | (df["dept"] == "Mkt")] # OR: use |
df[df["name"].isin(["Alice", "Bob"])]
df[df["name"].str.contains("li", case=False)]
df[df["salary"].between(50000, 80000)]
df[~df["dept"].isin(["Mkt"])]                     # NOT
df[df["salary"].notna()]                           # not null

# Query syntax (cleaner for complex filters)
df.query("age > 25 and dept == 'Eng'")
df.query("salary > @threshold")                   # Python variable with @

# Chained conditions with .loc
df.loc[(df["age"] > 25) & (df["dept"] == "Eng"), ["name", "salary"]]
```

## Modifying Data

```python
# New columns
df["bonus"] = df["salary"] * 0.1
df["age_group"] = pd.cut(df["age"], bins=[0, 25, 35, 100], labels=["junior", "mid", "senior"])
df["tenure_years"] = (pd.Timestamp.now() - df["joined"]).dt.days / 365.25
df["salary_rank"] = df["salary"].rank(ascending=False)

# Conditional columns
df["level"] = np.where(df["salary"] > 70000, "senior", "junior")

# Multiple conditions
df["band"] = np.select(
    [df["salary"] > 80000, df["salary"] > 60000, df["salary"] > 0],
    ["A", "B", "C"],
    default="D"
)

# Apply (custom function to each row or column)
df["name_length"] = df["name"].apply(len)
df["description"] = df.apply(lambda r: f"{r['name']} ({r['dept']})", axis=1)

# Replace values
df["dept"] = df["dept"].replace({"Eng": "Engineering", "Mkt": "Marketing"})

# Rename
df = df.rename(columns={"dept": "department", "salary": "base_salary"})

# Drop
df = df.drop(columns=["bonus", "band"])
df = df.drop(index=[0, 4])

# Sort
df = df.sort_values("salary", ascending=False)
df = df.sort_values(["dept", "salary"], ascending=[True, False])

# Reset index
df = df.reset_index(drop=True)

# Type conversion
df["age"] = df["age"].astype(float)
df["joined"] = pd.to_datetime(df["joined"])
df["salary"] = pd.to_numeric(df["salary"], errors="coerce")  # invalid → NaN
```

## Missing Data

```python
# Detect
df.isnull().sum()                              # count per column
df[df["salary"].isnull()]                      # rows with missing salary
df.isnull().mean()                             # percentage missing per column

# Fill
df["salary"] = df["salary"].fillna(df["salary"].median())     # fill with median
df["dept"] = df["dept"].fillna("Unknown")                      # fill categorical
df = df.fillna({"salary": 0, "dept": "Unknown", "age": df["age"].mean()})  # per-column
df["salary"] = df["salary"].ffill()                            # forward fill (time series)
df["salary"] = df["salary"].interpolate()                      # linear interpolation

# Drop
df = df.dropna()                               # drop any row with ANY NaN
df = df.dropna(subset=["salary", "dept"])       # drop if these specific cols are NaN
df = df.dropna(thresh=3)                        # keep rows with ≥3 non-NaN values

# GOTCHA: NEVER fill test data with test statistics
# Always fit imputer on TRAIN only, then transform both:
# imputer.fit(X_train) → imputer.transform(X_train) and imputer.transform(X_test)
```

## GroupBy and Aggregation

```python
# Basic groupby
df.groupby("dept")["salary"].mean()
df.groupby("dept")["salary"].agg(["mean", "median", "std", "count"])

# Multiple columns
df.groupby("dept").agg(
    avg_salary=("salary", "mean"),
    max_salary=("salary", "max"),
    headcount=("name", "count"),
    avg_age=("age", "mean"),
)

# Group by multiple columns
df.groupby(["dept", "age_group"])["salary"].mean()

# Transform (returns same-shape as input — for adding group stats as new columns)
df["dept_avg_salary"] = df.groupby("dept")["salary"].transform("mean")
df["salary_vs_dept_avg"] = df["salary"] - df["dept_avg_salary"]

# Filter groups
df.groupby("dept").filter(lambda g: g["salary"].mean() > 60000)

# Pivot table
df.pivot_table(values="salary", index="dept", columns="age_group", aggfunc="mean")

# Crosstab
pd.crosstab(df["dept"], df["age_group"], margins=True)
```

## Merging and Joining

```python
# Merge (SQL JOIN)
result = pd.merge(df1, df2, on="user_id", how="left")
result = pd.merge(df1, df2, left_on="id", right_on="user_id", how="inner")

# how: "inner" (default), "left", "right", "outer"
# Inner: only matching rows
# Left: all rows from left, matching from right (NaN if no match)

# Concat (stack DataFrames)
combined = pd.concat([df1, df2], axis=0, ignore_index=True)  # vertical stack
combined = pd.concat([df1, df2], axis=1)                      # horizontal stack

# GOTCHA: merge on columns with different types → unexpected results
# df1["id"] is int, df2["id"] is string → NO matches found!
# Fix: df2["id"] = df2["id"].astype(int)
```

## String Operations

```python
df["name"].str.lower()
df["name"].str.upper()
df["name"].str.strip()
df["name"].str.len()
df["name"].str.contains("ali", case=False, na=False)  # na=False: treat NaN as False
df["name"].str.startswith("A")
df["name"].str.replace("Alice", "Alicia", regex=False)
df["name"].str.split(" ", expand=True)
df["email"].str.extract(r"@(\w+)\.")                  # regex capture group
df["name"].str.cat(df["dept"], sep=" - ")              # concatenate two columns
df["name"].str.zfill(10)                               # pad with zeros
```

## Datetime Operations

```python
df["joined"] = pd.to_datetime(df["joined"])

df["year"] = df["joined"].dt.year
df["month"] = df["joined"].dt.month
df["day_of_week"] = df["joined"].dt.day_name()
df["quarter"] = df["joined"].dt.quarter
df["is_weekend"] = df["joined"].dt.dayofweek >= 5

# Time differences
df["tenure"] = pd.Timestamp.now() - df["joined"]
df["tenure_days"] = df["tenure"].dt.days

# Resample (time series)
ts = df.set_index("joined")
ts.resample("M")["salary"].mean()   # monthly average
ts.resample("Q")["salary"].sum()    # quarterly sum

# Date ranges
pd.date_range("2024-01-01", periods=12, freq="M")
pd.bdate_range("2024-01-01", "2024-12-31")  # business days only
```

---

# Chapter 3: Matplotlib & Seaborn

```python
import matplotlib.pyplot as plt
import seaborn as sns
sns.set_theme(style="whitegrid")

# ── Line Plot (training curves) ──
fig, ax = plt.subplots(figsize=(10, 6))
ax.plot(epochs, train_loss, label="Train", color="#2196F3", linewidth=2)
ax.plot(epochs, val_loss, label="Validation", color="#FF5722", linewidth=2, linestyle="--")
ax.set_xlabel("Epoch", fontsize=12)
ax.set_ylabel("Loss", fontsize=12)
ax.set_title("Training Progress", fontsize=14)
ax.legend(fontsize=11)
ax.grid(True, alpha=0.3)
plt.tight_layout()
plt.savefig("training_curves.png", dpi=150, bbox_inches="tight")

# ── Histogram + KDE (feature distribution) ──
fig, ax = plt.subplots(figsize=(8, 5))
sns.histplot(df["salary"], bins=30, kde=True, color="#4CAF50", ax=ax)
ax.axvline(df["salary"].median(), color="red", linestyle="--", label="Median")
ax.legend()
ax.set_title("Salary Distribution")

# ── Box Plot (compare groups) ──
sns.boxplot(x="dept", y="salary", data=df, palette="Set2")

# ── Scatter with regression line ──
sns.regplot(x="age", y="salary", data=df, scatter_kws={"alpha": 0.5})

# ── Correlation Heatmap ──
plt.figure(figsize=(10, 8))
corr = df.select_dtypes(include=np.number).corr()
mask = np.triu(np.ones_like(corr, dtype=bool))  # mask upper triangle
sns.heatmap(corr, mask=mask, annot=True, cmap="coolwarm", center=0, 
            fmt=".2f", square=True, linewidths=0.5)
plt.title("Feature Correlations")

# ── Pair Plot (all pairwise) ──
sns.pairplot(df[["age", "salary", "tenure_years", "dept"]], hue="dept",
             diag_kind="kde", plot_kws={"alpha": 0.6})

# ── Confusion Matrix ──
from sklearn.metrics import confusion_matrix, ConfusionMatrixDisplay
cm = confusion_matrix(y_test, y_pred)
disp = ConfusionMatrixDisplay(cm, display_labels=model.classes_)
disp.plot(cmap="Blues", values_format="d")
plt.title("Confusion Matrix")

# ── ROC Curve ──
from sklearn.metrics import roc_curve, roc_auc_score
fpr, tpr, _ = roc_curve(y_test, y_proba)
plt.plot(fpr, tpr, label=f"Model (AUC={roc_auc_score(y_test, y_proba):.3f})")
plt.plot([0,1], [0,1], "k--", label="Random")
plt.xlabel("FPR"); plt.ylabel("TPR"); plt.legend(); plt.title("ROC Curve")

# ── Feature Importance ──
importances = pd.Series(model.feature_importances_, index=X.columns)
importances.sort_values().tail(15).plot.barh(color="#2196F3")
plt.xlabel("Importance"); plt.title("Top 15 Features")
```

---

# Chapter 4: Scikit-Learn — Complete ML Workflow

## The Universal Pattern

```python
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline
from sklearn.metrics import classification_report

# EVERY sklearn ML project follows this:

# 1. Prepare
X = df.drop(columns=["target"])
y = df["target"]

# 2. Split (BEFORE any preprocessing!)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)

# 3. Build pipeline (preprocessing + model)
pipe = Pipeline([
    ("scaler", StandardScaler()),
    ("model", RandomForestClassifier(n_estimators=100, random_state=42)),
])

# 4. Train
pipe.fit(X_train, y_train)

# 5. Evaluate
y_pred = pipe.predict(X_test)
print(classification_report(y_test, y_pred))
```

## Preprocessing Deep Dive

```python
from sklearn.preprocessing import (
    StandardScaler,      # mean=0, std=1 (use for: logistic reg, SVM, KNN, neural nets)
    MinMaxScaler,        # range [0, 1] (use for: neural nets, algorithms sensitive to magnitude)
    RobustScaler,        # median and IQR (use for: data with outliers)
    LabelEncoder,        # text → integers (y labels only, NOT features)
    OneHotEncoder,       # categories → binary columns
    OrdinalEncoder,      # ordered categories → integers
)
from sklearn.impute import SimpleImputer, KNNImputer
from sklearn.compose import ColumnTransformer

# ── ColumnTransformer (handle different column types differently) ──
numeric_features = ["age", "salary", "experience"]
categorical_features = ["dept", "education"]

preprocessor = ColumnTransformer(
    transformers=[
        ("num", Pipeline([
            ("imputer", SimpleImputer(strategy="median")),
            ("scaler", StandardScaler()),
        ]), numeric_features),
        ("cat", Pipeline([
            ("imputer", SimpleImputer(strategy="most_frequent")),
            ("encoder", OneHotEncoder(handle_unknown="ignore", sparse_output=False)),
        ]), categorical_features),
    ],
    remainder="drop",  # drop columns not listed above
)

# Full pipeline: preprocess → model
full_pipe = Pipeline([
    ("preprocessor", preprocessor),
    ("model", RandomForestClassifier(n_estimators=100, random_state=42)),
])

full_pipe.fit(X_train, y_train)
y_pred = full_pipe.predict(X_test)

# GOTCHA: the preprocessor learns statistics from TRAIN data only.
# .fit_transform(X_train) → learns mean/std from train, transforms train
# .transform(X_test) → uses train's mean/std to transform test
# NEVER call .fit() on test data!
```

## Model Selection and Hyperparameter Tuning

```python
from sklearn.model_selection import (
    cross_val_score,
    GridSearchCV,
    RandomizedSearchCV,
    StratifiedKFold,
    learning_curve,
)

# ── Cross-Validation ──
cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
scores = cross_val_score(full_pipe, X_train, y_train, cv=cv, scoring="f1")
print(f"CV F1: {scores.mean():.4f} ± {scores.std():.4f}")

# ── Grid Search (exhaustive, good for small param spaces) ──
param_grid = {
    "model__n_estimators": [100, 200, 300],
    "model__max_depth": [5, 10, 15, None],
    "model__min_samples_split": [2, 5, 10],
}

grid = GridSearchCV(full_pipe, param_grid, cv=cv, scoring="f1", n_jobs=-1, verbose=1)
grid.fit(X_train, y_train)
print(f"Best F1: {grid.best_score_:.4f}")
print(f"Best params: {grid.best_params_}")
best_model = grid.best_estimator_

# ── Randomized Search (faster, good for large param spaces) ──
from scipy.stats import randint, uniform

param_dist = {
    "model__n_estimators": randint(50, 500),
    "model__max_depth": [3, 5, 7, 10, 15, None],
    "model__min_samples_split": randint(2, 50),
    "model__min_samples_leaf": randint(1, 20),
}

random_search = RandomizedSearchCV(full_pipe, param_dist, n_iter=100, cv=cv,
                                    scoring="f1", n_jobs=-1, random_state=42)
random_search.fit(X_train, y_train)

# ── Learning Curves (diagnose overfitting/underfitting) ──
train_sizes, train_scores, val_scores = learning_curve(
    full_pipe, X_train, y_train, cv=5, scoring="f1",
    train_sizes=np.linspace(0.1, 1.0, 10), n_jobs=-1
)

plt.plot(train_sizes, train_scores.mean(axis=1), label="Train")
plt.plot(train_sizes, val_scores.mean(axis=1), label="Validation")
plt.xlabel("Training Set Size")
plt.ylabel("F1 Score")
plt.title("Learning Curves")
plt.legend()
# Train >> Val → overfitting (need more data or regularization)
# Both low → underfitting (need more complex model)
# Both high and close → good fit
```

## Saving and Loading Models

```python
import joblib

# Save the entire pipeline (preprocessing + model)
joblib.dump(full_pipe, "model_pipeline.joblib")

# Load
loaded_pipe = joblib.load("model_pipeline.joblib")
predictions = loaded_pipe.predict(new_data)

# GOTCHA: the loaded pipeline includes the scaler's learned mean/std.
# You can predict on raw data without re-scaling. That's the power of pipelines.
```

## Real-World Workflow Template

```python
"""Complete ML project template — production-ready pattern."""
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split, cross_val_score, StratifiedKFold
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.impute import SimpleImputer
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import classification_report, roc_auc_score
from sklearn.model_selection import RandomizedSearchCV
import joblib

# 1. Load
df = pd.read_csv("data.csv")
print(f"Shape: {df.shape}")
print(f"Nulls:\n{df.isnull().sum()}")
print(f"Target distribution:\n{df['target'].value_counts(normalize=True)}")

# 2. Prepare
X = df.drop(columns=["target", "id", "timestamp"])  # drop non-feature columns
y = df["target"]

num_cols = X.select_dtypes(include="number").columns.tolist()
cat_cols = X.select_dtypes(include="object").columns.tolist()

# 3. Split
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)

# 4. Build pipeline
preprocessor = ColumnTransformer([
    ("num", Pipeline([
        ("imputer", SimpleImputer(strategy="median")),
        ("scaler", StandardScaler()),
    ]), num_cols),
    ("cat", Pipeline([
        ("imputer", SimpleImputer(strategy="most_frequent")),
        ("encoder", OneHotEncoder(handle_unknown="ignore", sparse_output=False)),
    ]), cat_cols),
])

pipe = Pipeline([
    ("preprocessor", preprocessor),
    ("model", RandomForestClassifier(random_state=42, class_weight="balanced")),
])

# 5. Quick baseline
cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
baseline_scores = cross_val_score(pipe, X_train, y_train, cv=cv, scoring="f1")
print(f"Baseline F1: {baseline_scores.mean():.4f} ± {baseline_scores.std():.4f}")

# 6. Tune
param_dist = {
    "model__n_estimators": [100, 200, 300, 500],
    "model__max_depth": [5, 10, 15, None],
    "model__min_samples_split": [2, 5, 10],
}
search = RandomizedSearchCV(pipe, param_dist, n_iter=30, cv=cv, scoring="f1", n_jobs=-1)
search.fit(X_train, y_train)
print(f"Tuned F1: {search.best_score_:.4f}")

# 7. Final evaluation on test set (ONCE!)
best_pipe = search.best_estimator_
y_pred = best_pipe.predict(X_test)
y_proba = best_pipe.predict_proba(X_test)[:, 1]
print(classification_report(y_test, y_pred))
print(f"AUC: {roc_auc_score(y_test, y_proba):.4f}")

# 8. Save
joblib.dump(best_pipe, "production_model.joblib")
```
