# ML Tier 2 — Classical ML Algorithms (Staff-Level Depth)
*Intuition · Math · Code · Tuning · Edge Cases · Interview Q&A*

---

# 1. Linear Regression

## Intuition

You have data points scattered on a graph. Linear regression draws the BEST straight line through them — "best" meaning the line that minimizes the total distance between the line and all points.

```
  Price ↑
  800K │                          * 
  600K │               *    *  
  400K │          *  *
  200K │    *  *
       └─────────────────────────────▶ Square Feet
        500  1000  1500  2000  2500

  The line: price = w × sqft + b
  w (slope): each extra sqft adds $w to the price
  b (intercept): base price when sqft = 0
```

## The Math

```
Model:    ŷ = w₁x₁ + w₂x₂ + ... + wₙxₙ + b   (or in matrix form: ŷ = Xw + b)

Loss function (MSE):
  L = (1/2n) × Σ (ŷᵢ - yᵢ)²
  "Average of squared differences between prediction and truth"

Gradient (how to update weights):
  ∂L/∂w = (1/n) × Xᵀ(Xw - y)
  ∂L/∂b = (1/n) × Σ(ŷᵢ - yᵢ)

Closed-form solution (no gradient descent needed):
  w = (XᵀX)⁻¹ Xᵀy     ← "Normal Equation"
  Works instantly for small datasets.
  Fails for large datasets (matrix inversion is O(n³)).

Assumptions:
  1. Linear relationship between features and target
  2. Features are not highly correlated (no multicollinearity)
  3. Errors are normally distributed with constant variance (homoscedasticity)
  4. Observations are independent
```

## Code (sklearn)

```python
import numpy as np
import pandas as pd
from sklearn.linear_model import LinearRegression, Ridge, Lasso, ElasticNet
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.preprocessing import StandardScaler, PolynomialFeatures
from sklearn.metrics import mean_squared_error, mean_absolute_error, r2_score
from sklearn.pipeline import Pipeline
import matplotlib.pyplot as plt

# ── Prepare Data ──
df = pd.read_csv("houses.csv")
X = df[["sqft", "bedrooms", "bathrooms", "age"]]
y = df["price"]

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# ── Basic Linear Regression ──
model = LinearRegression()
model.fit(X_train, y_train)

print(f"Coefficients: {dict(zip(X.columns, model.coef_))}")
# {'sqft': 210.5, 'bedrooms': 45000, 'bathrooms': 12000, 'age': -1500}
# Interpretation: each extra sqft adds $210.50, each extra year REDUCES price by $1500

print(f"Intercept: {model.intercept_:.2f}")    # base price
print(f"R² (train): {model.score(X_train, y_train):.4f}")
print(f"R² (test):  {model.score(X_test, y_test):.4f}")

y_pred = model.predict(X_test)
print(f"MAE:  ${mean_absolute_error(y_test, y_pred):,.0f}")
print(f"RMSE: ${np.sqrt(mean_squared_error(y_test, y_pred)):,.0f}")

# ── Regularized Variants ──
# Ridge (L2): penalizes large weights → prevents overfitting
ridge = Ridge(alpha=1.0)    # alpha = regularization strength
ridge.fit(X_train, y_train)

# Lasso (L1): pushes some weights to EXACTLY 0 → feature selection
lasso = Lasso(alpha=0.1)
lasso.fit(X_train, y_train)
print(f"Lasso non-zero features: {np.sum(lasso.coef_ != 0)} out of {len(lasso.coef_)}")

# ElasticNet: mix of Ridge + Lasso
elastic = ElasticNet(alpha=0.1, l1_ratio=0.5)  # l1_ratio: 0=Ridge, 1=Lasso, 0.5=both

# ── Polynomial Regression (for curves) ──
# If relationship is non-linear: sqft² might matter
poly_pipeline = Pipeline([
    ("poly", PolynomialFeatures(degree=2, include_bias=False)),
    ("scaler", StandardScaler()),
    ("model", Ridge(alpha=1.0)),
])
poly_pipeline.fit(X_train, y_train)
print(f"Poly R² (test): {poly_pipeline.score(X_test, y_test):.4f}")

# ── Cross-Validation (more reliable than single split) ──
cv_scores = cross_val_score(model, X, y, cv=5, scoring="r2")
print(f"CV R²: {cv_scores.mean():.4f} ± {cv_scores.std():.4f}")

# ── Residual Plot (check assumptions) ──
residuals = y_test - y_pred
plt.scatter(y_pred, residuals, alpha=0.5)
plt.axhline(y=0, color='red', linestyle='--')
plt.xlabel("Predicted")
plt.ylabel("Residual")
plt.title("Residual Plot — should be random scatter around 0")
plt.show()
# Pattern in residuals → linear model is WRONG (need polynomial/tree model)
```

## Hyperparameter Tuning

```
LinearRegression:     no hyperparameters (that's the beauty and limitation)
Ridge:                alpha (0.001 to 100) — higher = more regularization
Lasso:                alpha (0.0001 to 10) — higher = more features pushed to 0
ElasticNet:           alpha + l1_ratio (0 to 1)
PolynomialFeatures:   degree (2 or 3 — higher overfits fast)

from sklearn.model_selection import GridSearchCV

param_grid = {"alpha": [0.001, 0.01, 0.1, 1, 10, 100]}
grid = GridSearchCV(Ridge(), param_grid, cv=5, scoring="r2")
grid.fit(X_train, y_train)
print(f"Best alpha: {grid.best_params_['alpha']}, R²: {grid.best_score_:.4f}")
```

## When to Use / When NOT to Use

```
USE when:
  ✓ Relationship is roughly linear
  ✓ You need interpretable coefficients ("each bedroom adds $45K")
  ✓ Quick baseline before trying complex models
  ✓ Few features relative to samples
  ✓ Need to understand which features matter most

DON'T USE when:
  ✗ Relationship is clearly non-linear (use trees or polynomial features)
  ✗ Features are highly correlated (multicollinearity → unstable coefficients, use Ridge)
  ✗ Many irrelevant features (use Lasso for automatic feature selection)
  ✗ Outliers dominate the data (use robust regression or tree models)
  ✗ You need the best possible accuracy (use XGBoost/neural nets)
```

## Edge Cases and Gotchas

```
1. MULTICOLLINEARITY: if sqft and total_rooms are highly correlated,
   coefficients become unstable (wild swings with small data changes).
   Fix: drop one feature, or use Ridge (L2 stabilizes coefficients).
   Detect: compute VIF (Variance Inflation Factor > 5 = problem).

2. OUTLIERS: one mansion priced at $50M skews the entire line.
   Fix: use Huber regression (robust to outliers), or log-transform the target.
   
3. FEATURE SCALING: linear regression COEFFICIENTS change with scale
   but PREDICTIONS don't. However, for regularized versions (Ridge/Lasso),
   you MUST scale features first (StandardScaler), otherwise the penalty
   is unfair — features with large scales get penalized more.

4. TARGET DISTRIBUTION: if prices are right-skewed (many cheap houses, few mansions),
   predicting log(price) and then exponentiating often works better.
   y_transformed = np.log1p(y)    # log(1 + y) to handle zeros

5. EXTRAPOLATION: the model has NO IDEA what happens outside training range.
   If all houses are 500-3000 sqft, don't trust predictions for 10,000 sqft.
```

---

# 2. Logistic Regression

## Intuition

Linear regression predicts a number. But what if you need a yes/no answer — "will this customer churn?" You need the output to be a probability between 0 and 1.

Logistic regression does: linear regression → squeeze through sigmoid → probability.

```
  z = w₁x₁ + w₂x₂ + ... + b        (linear, can be -∞ to +∞)
  P = σ(z) = 1 / (1 + e⁻ᶻ)          (sigmoid squashes to 0-1)

  If P > 0.5 → predict class 1 (churn)
  If P ≤ 0.5 → predict class 0 (no churn)

  σ(z):
  1.0 │              ─────────
      │           ╱
  0.5 │─ ─ ─ ─╱─ ─ ─ ─ ─ ─ ─    ← decision boundary
      │      ╱
  0.0 │─────╱
      └────────────────────────▶ z
     -6   -3    0    3    6
```

## The Math

```
Model:    P(y=1|x) = σ(wᵀx + b) = 1 / (1 + e^(-(wᵀx + b)))

Loss function (Binary Cross-Entropy):
  L = -(1/n) × Σ [yᵢ × log(P̂ᵢ) + (1-yᵢ) × log(1-P̂ᵢ)]

  If y=1 and P̂=0.95: loss = -log(0.95) = 0.05     (small, good)
  If y=1 and P̂=0.05: loss = -log(0.05) = 3.0      (large, bad — model was wrong)

  No closed-form solution. Solved with gradient descent.

Gradient:
  ∂L/∂w = (1/n) × Xᵀ(σ(Xw) - y)    (same form as linear regression!)

Multi-class: use softmax instead of sigmoid, cross-entropy loss.
  One-vs-Rest (OvR): train K binary classifiers, pick highest confidence.
  Multinomial: one model with softmax output (sklearn default with solver='lbfgs').
```

## Code (sklearn)

```python
from sklearn.linear_model import LogisticRegression
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score, f1_score,
    classification_report, confusion_matrix, roc_auc_score, roc_curve,
    ConfusionMatrixDisplay
)
from sklearn.pipeline import Pipeline

# ── Prepare ──
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, 
                                                      random_state=42, stratify=y)

# ── Train (always scale for logistic regression!) ──
pipe = Pipeline([
    ("scaler", StandardScaler()),
    ("model", LogisticRegression(
        C=1.0,                  # inverse regularization (higher C = less regularization)
        penalty="l2",           # "l1", "l2", "elasticnet", or None
        solver="lbfgs",         # "lbfgs" (default), "liblinear" (small data), "saga" (large data)
        max_iter=1000,          # increase if it doesn't converge
        class_weight="balanced", # auto-adjust for imbalanced classes
        random_state=42,
    )),
])
pipe.fit(X_train, y_train)

# ── Evaluate ──
y_pred = pipe.predict(X_test)
y_proba = pipe.predict_proba(X_test)[:, 1]   # probability of class 1

print(classification_report(y_test, y_pred))
#               precision    recall  f1-score   support
# 0 (no churn)      0.92      0.95      0.93       400
# 1 (churn)         0.78      0.70      0.74       100
# accuracy                              0.90       500

# ── Confusion Matrix ──
cm = confusion_matrix(y_test, y_pred)
ConfusionMatrixDisplay(cm, display_labels=["No Churn", "Churn"]).plot(cmap="Blues")
plt.title("Confusion Matrix")
plt.show()

# ── ROC Curve ──
fpr, tpr, thresholds = roc_curve(y_test, y_proba)
auc = roc_auc_score(y_test, y_proba)
plt.plot(fpr, tpr, label=f"AUC = {auc:.3f}")
plt.plot([0, 1], [0, 1], "k--", label="Random (AUC = 0.5)")
plt.xlabel("False Positive Rate")
plt.ylabel("True Positive Rate (Recall)")
plt.title("ROC Curve")
plt.legend()
plt.show()

# ── Interpret Coefficients ──
model = pipe.named_steps["model"]
coefs = pd.Series(model.coef_[0], index=X.columns).sort_values()
print("Feature importance (log-odds):")
print(coefs)
# Positive coefficient → increases probability of class 1
# Negative coefficient → decreases probability

# ── Adjusting Threshold (not always 0.5!) ──
# For cancer detection: lower threshold to catch more cases (higher recall)
threshold = 0.3
y_pred_custom = (y_proba >= threshold).astype(int)
print(f"Recall at threshold {threshold}: {recall_score(y_test, y_pred_custom):.2f}")
print(f"Precision at threshold {threshold}: {precision_score(y_test, y_pred_custom):.2f}")
```

## Edge Cases and Gotchas

```
1. IMBALANCED CLASSES: 95% no-churn, 5% churn. Model predicts "no churn" for everyone → 95% accuracy!
   Fix: class_weight="balanced", or use SMOTE oversampling, or evaluate with F1/AUC instead of accuracy.

2. FEATURE SCALING IS MANDATORY: sigmoid is sensitive to input magnitude.
   Without scaling, features with large values dominate. Always StandardScaler.

3. CONVERGENCE WARNING: "ConvergenceWarning: lbfgs failed to converge"
   Fix: increase max_iter=5000, or scale features, or reduce C (more regularization).

4. PERFECT SEPARATION: if one feature perfectly separates classes (e.g., age > 60 → always churn),
   coefficients blow up to infinity. Fix: add regularization (C < 1.0).

5. THRESHOLD TUNING: default 0.5 isn't always best.
   Use precision-recall curve to find the right threshold for your use case.
   from sklearn.metrics import precision_recall_curve
   precisions, recalls, thresholds = precision_recall_curve(y_test, y_proba)
```

---

# 3. Decision Trees

## Intuition

A flowchart of yes/no questions, learned automatically from data.

```
"Should we approve this loan?"

                    Income > $50K?
                   /              \
                 Yes               No
                /                    \
      Credit > 700?             Employed?
       /        \                /       \
     Yes        No             Yes       No
      |          |              |         |
   APPROVE    REVIEW          REVIEW    REJECT
```

## The Math: How Splits Are Chosen

```
At each node, try EVERY feature and EVERY possible threshold.
Pick the split that maximally REDUCES impurity.

GINI IMPURITY (sklearn default):
  Gini(node) = 1 - Σ pᵢ²    where pᵢ = proportion of class i in the node

  Pure node (all same class):   Gini = 1 - 1² = 0
  Worst node (50/50 binary):    Gini = 1 - 0.5² - 0.5² = 0.5

INFORMATION GAIN (entropy-based):
  Entropy(node) = -Σ pᵢ × log₂(pᵢ)
  
  Pure node:   Entropy = 0
  50/50:       Entropy = 1.0

  Information Gain = Entropy(parent) - weighted_avg(Entropy(children))
  Pick the split with HIGHEST information gain.

Example:
  Parent: 100 samples (60 approved, 40 rejected)
  Gini = 1 - (0.6)² - (0.4)² = 1 - 0.36 - 0.16 = 0.48

  Split on "Income > 50K":
    Left child:  70 samples (55 approved, 15 rejected)  Gini = 0.337
    Right child: 30 samples (5 approved, 25 rejected)   Gini = 0.278
    Weighted Gini = (70/100 × 0.337) + (30/100 × 0.278) = 0.319

  Split on "Credit > 700":
    Left child:  50 samples (48 approved, 2 rejected)   Gini = 0.077
    Right child: 50 samples (12 approved, 38 rejected)  Gini = 0.365
    Weighted Gini = (50/100 × 0.077) + (50/100 × 0.365) = 0.221

  Credit > 700 has LOWER Gini (more pure) → better split → chosen!
```

## Code (sklearn)

```python
from sklearn.tree import DecisionTreeClassifier, DecisionTreeRegressor, plot_tree
from sklearn.model_selection import GridSearchCV

# ── Train ──
tree = DecisionTreeClassifier(
    max_depth=5,               # limit depth to prevent overfitting
    min_samples_split=20,      # minimum samples to split a node
    min_samples_leaf=10,       # minimum samples in a leaf node
    criterion="gini",          # "gini" or "entropy"
    class_weight="balanced",   # handle imbalanced classes
    random_state=42,
)
tree.fit(X_train, y_train)

# ── Evaluate ──
print(f"Train accuracy: {tree.score(X_train, y_train):.4f}")
print(f"Test accuracy:  {tree.score(X_test, y_test):.4f}")
# If train >> test → overfitting! Reduce max_depth or increase min_samples_split.

# ── Visualize the tree ──
plt.figure(figsize=(20, 10))
plot_tree(tree, feature_names=X.columns, class_names=["No", "Yes"],
          filled=True, rounded=True, max_depth=3, fontsize=10)
plt.title("Decision Tree (first 3 levels)")
plt.tight_layout()
plt.show()

# ── Feature Importance ──
importances = pd.Series(tree.feature_importances_, index=X.columns)
importances.sort_values(ascending=True).plot.barh()
plt.title("Feature Importance")
plt.xlabel("Importance (Gini reduction)")
plt.show()

# ── Hyperparameter Tuning ──
param_grid = {
    "max_depth": [3, 5, 7, 10, None],
    "min_samples_split": [2, 5, 10, 20, 50],
    "min_samples_leaf": [1, 5, 10, 20],
    "criterion": ["gini", "entropy"],
}

grid = GridSearchCV(DecisionTreeClassifier(random_state=42), param_grid,
                    cv=5, scoring="f1", n_jobs=-1)
grid.fit(X_train, y_train)
print(f"Best params: {grid.best_params_}")
print(f"Best F1: {grid.best_score_:.4f}")

# ── Regression Tree (predicting numbers) ──
reg_tree = DecisionTreeRegressor(max_depth=5)
reg_tree.fit(X_train, y_train)
```

## Edge Cases

```
1. OVERFITTING: an unrestricted tree memorizes training data perfectly (train acc=100%).
   Fix: max_depth, min_samples_split, min_samples_leaf, max_leaf_nodes, ccp_alpha (pruning).

2. INSTABILITY: small data changes → completely different tree. Sensitive to noise.
   Fix: use Random Forest (ensemble of trees, much more stable).

3. BIASED TOWARD FEATURES WITH MANY LEVELS: a feature with 100 unique values
   has more potential split points → more likely to be chosen. Not because it's BETTER.
   Fix: careful feature engineering, or use information gain ratio (C4.5 algorithm).

4. CAN'T EXTRAPOLATE: decision trees partition the feature space into rectangles.
   They can't predict values OUTSIDE the training range (they'll just predict the
   nearest leaf's average). For extrapolation, use linear models.

5. NO FEATURE SCALING NEEDED: trees split on thresholds, not distances. 
   A feature in [0,1] and one in [0,1000000] are treated the same.
```

---

# 4. Random Forest

## Code (sklearn)

```python
from sklearn.ensemble import RandomForestClassifier, RandomForestRegressor

rf = RandomForestClassifier(
    n_estimators=200,            # number of trees (more = better, diminishing returns after ~200)
    max_depth=10,                # per-tree max depth (None = unlimited)
    min_samples_split=10,        # min samples to split
    min_samples_leaf=5,          # min samples per leaf
    max_features="sqrt",         # features per split: "sqrt" (classification) or "log2"
    class_weight="balanced",     # handle imbalance
    n_jobs=-1,                   # use all CPU cores
    random_state=42,
    oob_score=True,              # out-of-bag score (free validation!)
)
rf.fit(X_train, y_train)

print(f"OOB Score: {rf.oob_score_:.4f}")   # validation without a separate val set!
print(f"Test Accuracy: {rf.score(X_test, y_test):.4f}")

# Feature importance
importances = pd.Series(rf.feature_importances_, index=X.columns)
importances.sort_values(ascending=True).tail(10).plot.barh()
plt.title("Top 10 Features (Random Forest)")
plt.show()

# Hyperparameter tuning
from sklearn.model_selection import RandomizedSearchCV

param_dist = {
    "n_estimators": [100, 200, 300, 500],
    "max_depth": [5, 10, 15, 20, None],
    "min_samples_split": [2, 5, 10, 20],
    "min_samples_leaf": [1, 2, 5, 10],
    "max_features": ["sqrt", "log2", 0.3, 0.5],
}

search = RandomizedSearchCV(RandomForestClassifier(random_state=42),
                            param_dist, n_iter=50, cv=5, scoring="f1",
                            n_jobs=-1, random_state=42)
search.fit(X_train, y_train)
print(f"Best F1: {search.best_score_:.4f}")
print(f"Best params: {search.best_params_}")
```

## When to Use / When NOT

```
USE:
  ✓ First model to try on ANY tabular data (strong baseline)
  ✓ Don't want to worry about feature scaling
  ✓ Need feature importance rankings
  ✓ Mixed feature types (numeric + categorical)
  ✓ Need out-of-bag validation (oob_score)

DON'T USE:
  ✗ Need the absolute best accuracy (XGBoost/LightGBM usually win)
  ✗ Need fast inference for millions of predictions (200 trees × each prediction is slow)
  ✗ Need interpretability (200 trees are a black box — use single tree or logistic regression)
  ✗ Image/text data (use CNNs/transformers)
  ✗ Very high-dimensional sparse data (use linear models or gradient boosting)
```

---

# 5. Gradient Boosting (XGBoost, LightGBM, CatBoost)

## Intuition

Random Forest: build trees independently, vote at the end.
Gradient Boosting: build trees SEQUENTIALLY — each tree fixes the ERRORS of all previous trees.

```
Step 1: Predict with a simple model → has errors
Step 2: Build tree 2 to predict the ERRORS of step 1 → residual errors shrink
Step 3: Build tree 3 to predict the REMAINING errors → even smaller
...
Final: sum of all trees' predictions

Like editing an essay:
  Draft 1 → fix major issues → fix grammar → fix style → polished result
  Each edit (tree) focuses on what's STILL wrong.
```

## Code

```python
# XGBoost (most popular, best for competitions)
from xgboost import XGBClassifier, XGBRegressor

xgb = XGBClassifier(
    n_estimators=200,            # number of boosting rounds
    max_depth=6,                 # per-tree depth (3-10)
    learning_rate=0.1,           # shrinkage (0.01-0.3)
    subsample=0.8,               # fraction of data per tree
    colsample_bytree=0.8,        # fraction of features per tree
    reg_alpha=0.1,               # L1 regularization
    reg_lambda=1.0,              # L2 regularization
    scale_pos_weight=5,          # for imbalanced: = count(neg)/count(pos)
    eval_metric="logloss",
    early_stopping_rounds=20,    # stop if no improvement for 20 rounds
    n_jobs=-1,
    random_state=42,
)
xgb.fit(X_train, y_train, eval_set=[(X_test, y_test)], verbose=False)
print(f"Best iteration: {xgb.best_iteration}")

# LightGBM (faster, handles large datasets)
from lightgbm import LGBMClassifier

lgbm = LGBMClassifier(
    n_estimators=200,
    max_depth=6,
    learning_rate=0.1,
    num_leaves=31,               # LightGBM uses leaf-wise growth (not depth-wise)
    subsample=0.8,
    colsample_bytree=0.8,
    reg_alpha=0.1,
    reg_lambda=1.0,
    is_unbalance=True,           # for imbalanced classes
    n_jobs=-1,
    random_state=42,
)
lgbm.fit(X_train, y_train, eval_set=[(X_test, y_test)],
         callbacks=[lightgbm.early_stopping(20), lightgbm.log_evaluation(0)])

# CatBoost (handles categorical features natively, no encoding needed!)
from catboost import CatBoostClassifier

cat_features = ["department", "city", "education"]  # specify categorical columns

catboost = CatBoostClassifier(
    iterations=200,
    depth=6,
    learning_rate=0.1,
    cat_features=cat_features,   # handles categoricals automatically!
    auto_class_weights="Balanced",
    verbose=0,
    random_state=42,
)
catboost.fit(X_train, y_train, eval_set=(X_test, y_test), early_stopping_rounds=20)
```

## Tuning Guide

```
MOST IMPACTFUL hyperparameters (tune in this order):
  1. learning_rate (0.01-0.3): lower = more trees needed but better generalization
  2. n_estimators (100-5000): use early_stopping to find the right number
  3. max_depth (3-10): deeper = more complex, higher overfitting risk
  4. subsample (0.6-1.0): fraction of data per tree, lower = less overfit
  5. colsample_bytree (0.6-1.0): fraction of features per tree

STRATEGY:
  1. Start with learning_rate=0.1, n_estimators=1000, early_stopping=50
  2. Let early stopping find optimal n_estimators
  3. Tune max_depth and min_child_weight
  4. Tune subsample and colsample_bytree
  5. Drop learning_rate to 0.01, increase n_estimators proportionally
```

## XGBoost vs LightGBM vs CatBoost

```
                  XGBoost            LightGBM           CatBoost
─────────         ───────            ────────            ────────
Speed             Medium             Fastest             Medium
Memory            Medium             Lowest              Highest
Categorical       Manual encoding    Manual encoding     NATIVE support
Defaults          Need tuning        Good defaults       Best defaults
GPU               Yes                Yes                 Yes
Best for          Competitions       Large datasets      Mixed data types
```

---

# 6. KNN (K-Nearest Neighbors)

## Code

```python
from sklearn.neighbors import KNeighborsClassifier
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline

# MUST scale features (distance-based algorithm)
knn_pipe = Pipeline([
    ("scaler", StandardScaler()),
    ("knn", KNeighborsClassifier(
        n_neighbors=5,           # K: odd number to avoid ties
        weights="distance",      # "uniform" or "distance" (closer neighbors count more)
        metric="euclidean",      # "euclidean", "manhattan", "minkowski"
        n_jobs=-1,
    )),
])
knn_pipe.fit(X_train, y_train)

# Finding optimal K
from sklearn.model_selection import cross_val_score

k_scores = []
for k in range(1, 31):
    pipe = Pipeline([("scaler", StandardScaler()),
                     ("knn", KNeighborsClassifier(n_neighbors=k))])
    scores = cross_val_score(pipe, X_train, y_train, cv=5, scoring="accuracy")
    k_scores.append(scores.mean())

plt.plot(range(1, 31), k_scores)
plt.xlabel("K")
plt.ylabel("CV Accuracy")
plt.title("KNN: Accuracy vs K")
plt.show()
# Pick K where accuracy plateaus (usually 5-15)
```

## Gotchas

```
1. SCALING IS MANDATORY. Feature with range [0, 100000] dominates feature [0, 1].
2. SLOW AT PREDICTION: computes distance to ALL training points. O(n) per prediction.
   Fix: use KD-tree or Ball-tree (sklearn does this automatically for < 30 dimensions).
3. CURSE OF DIMENSIONALITY: in 1000 dimensions, all points are equidistant. KNN fails.
   Fix: PCA first, or don't use KNN for high-dimensional data.
4. K=1 overfits. Large K underfits. Use cross-validation to find the sweet spot.
```

---

# 7. K-Means Clustering

## Code

```python
from sklearn.cluster import KMeans
from sklearn.preprocessing import StandardScaler
from sklearn.metrics import silhouette_score

# Scale features first
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)

# Elbow method: find optimal K
inertias = []
sil_scores = []
K_range = range(2, 11)

for k in K_range:
    km = KMeans(n_clusters=k, n_init=10, random_state=42)
    km.fit(X_scaled)
    inertias.append(km.inertia_)
    sil_scores.append(silhouette_score(X_scaled, km.labels_))

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 5))
ax1.plot(K_range, inertias, "bo-")
ax1.set_title("Elbow Method")
ax1.set_xlabel("K")
ax1.set_ylabel("Inertia")

ax2.plot(K_range, sil_scores, "ro-")
ax2.set_title("Silhouette Score")
ax2.set_xlabel("K")
ax2.set_ylabel("Score (higher = better)")
plt.tight_layout()
plt.show()

# Train with best K
kmeans = KMeans(n_clusters=4, n_init=10, random_state=42)
df["cluster"] = kmeans.fit_predict(X_scaled)

# Analyze clusters
print(df.groupby("cluster").mean(numeric_only=True))
```

---

# 8. PCA (Principal Component Analysis)

## Code

```python
from sklearn.decomposition import PCA

# Scale FIRST (PCA is sensitive to scale)
scaler = StandardScaler()
X_scaled = scaler.fit_transform(X)

# Fit PCA
pca = PCA()
X_pca = pca.fit_transform(X_scaled)

# How many components explain 95% of variance?
cumulative_variance = np.cumsum(pca.explained_variance_ratio_)
n_components_95 = np.argmax(cumulative_variance >= 0.95) + 1
print(f"Components for 95% variance: {n_components_95} (out of {X.shape[1]})")

plt.plot(cumulative_variance)
plt.axhline(y=0.95, color='r', linestyle='--', label='95% variance')
plt.xlabel("Number of Components")
plt.ylabel("Cumulative Explained Variance")
plt.title("PCA: Explained Variance")
plt.legend()
plt.show()

# Reduce dimensions
pca_final = PCA(n_components=n_components_95)
X_reduced = pca_final.fit_transform(X_scaled)

# 2D visualization
pca_2d = PCA(n_components=2)
X_2d = pca_2d.fit_transform(X_scaled)
plt.scatter(X_2d[:, 0], X_2d[:, 1], c=y, cmap="viridis", alpha=0.5)
plt.xlabel("PC1")
plt.ylabel("PC2")
plt.title(f"PCA 2D (explains {pca_2d.explained_variance_ratio_.sum():.1%} of variance)")
plt.colorbar(label="Target")
plt.show()
```

---

# Complete Model Selection Guide

```
Problem Type       Start With              Then Try
────────────       ──────────              ────────
Regression         LinearRegression        Ridge → RandomForest → XGBoost
Binary Classif.    LogisticRegression      RandomForest → XGBoost → LightGBM
Multi-class        LogisticRegression      RandomForest → XGBoost
Clustering         KMeans                  DBSCAN → Hierarchical
Dim Reduction      PCA                     t-SNE (visualization only)
Anomaly Detection  IsolationForest         One-class SVM → Autoencoder
Time Series        ARIMA                   Prophet → LSTM
Image              CNN (ResNet)            EfficientNet → ViT
Text               TF-IDF + LogReg        BERT → LLM fine-tuning

GOLDEN RULE: always start simple, add complexity only if simple fails.
LogisticRegression → RandomForest → XGBoost → Neural Network
```

---

# 🧩 Interview Q&A

**Q: Compare all the algorithms — when do you use what?**
A: Linear/logistic regression for interpretability and quick baselines. Decision trees for understanding the logic but never alone (overfit). Random Forest as the robust default for tabular data. XGBoost/LightGBM when you need maximum accuracy and can tune. KNN for small data and recommendation. SVM for small-medium data with clear margins. Naive Bayes for text classification baselines. K-Means for customer segmentation. PCA before any high-dimensional model.

**Q: How do you handle imbalanced classes?**
A: Multiple strategies in combination. (1) class_weight="balanced" in the model. (2) SMOTE oversampling of minority class. (3) Adjust classification threshold (not always 0.5). (4) Evaluate with F1/AUC, NEVER just accuracy. (5) Collect more minority class data if possible. (6) Use algorithms robust to imbalance (XGBoost with scale_pos_weight). (7) Stratified splitting to maintain ratios.

**Q: Explain the bias-variance tradeoff with a concrete algorithm example.**
A: Decision tree with max_depth=1 (stump): high bias (too simple, misses patterns), low variance (same result regardless of data sample). Decision tree with max_depth=None: low bias (captures every pattern), high variance (completely different tree with different training data, memorizes noise). Random Forest reduces variance by averaging many high-variance trees — their random errors cancel out, but each tree still has low bias. The result: low bias AND low variance.

**Q: You have a dataset with 100 features. How do you decide which ones to keep?**
A: (1) Train Random Forest or XGBoost, look at feature_importances_ — drop features with near-zero importance. (2) Use Lasso (L1) regression — it pushes irrelevant feature weights to exactly 0. (3) Correlation analysis: drop one from each pair of highly correlated features (> 0.9). (4) PCA to reduce dimensionality while keeping variance. (5) Domain knowledge — remove features that logically shouldn't affect the target. (6) Recursive Feature Elimination (RFE): iteratively remove the least important feature.
