---
title: 'Stock Return Classifier (Part III): Baseline Models, ML Models & Hyperparameter Tuning'
date: 2026-02-17 12:00:00 -0800
permalink: /posts/2026/02/17/stock-return-classifier-part-III
tags:
  - machine learning
  - cross-validation
  - hyperparameter tuning
  - algorithmic trading
redirect_from:
  - /2026/02/17/stock-return-classifier-part-III.html
toc: true
---

*This post is part of a series on building a supervised ML pipeline to classify SPY daily returns.*

- [Part I: Problem Statement & Technical Indicators][src-part-I]
- [Part II: EDA, Feature Selection & Feature Engineering][src-part-II]
- [Part III: Baseline Models, ML Models & Hyperparameter Tuning][src-part-III]
- [Part IV: Test Evaluation & Portfolio Backtesting][src-part-IV]

---

**← Previous:** [Part II: EDA, Feature Selection & Feature Engineering](/posts/2026/02/17/stock-return-classifier-part-II)

**Next Post →** [Part IV: Test Evaluation & Portfolio Backtesting](/posts/2026/02/17/stock-return-classifier-part-IV)

---

## ⚠️ Disclaimer

**This blog series is for educational and research purposes only.** The content should not be considered financial advice, investment advice, or trading advice. Trading stocks and financial instruments involves substantial risk of loss and is not suitable for every investor. Past performance does not guarantee future results. Always consult with a qualified financial advisor before making investment decisions.

---

## Introduction

In [Part II](/posts/2026/02/17/stock-return-classifier-part-II), we analysed the data and engineered the features. Now the modelling work begins.

Before training complex models, we need baselines—simple benchmarks that any ML model must beat to be worth deploying. Then, we train three classifiers with temporal cross-validation and grid-search hyperparameter tuning. Finally, learning curves reveal whether the models are overfitting, underfitting, or data-hungry.

## Temporal Cross-Validation: The Critical Foundation

Financial time series have a property that makes standard cross-validation dangerous: **temporal ordering matters**. Training on future data to predict the past would produce wildly optimistic validation scores that don't reflect real-world performance.

### Expanding Window CV

We use **expanding window cross-validation** with 5 folds:

```
Data timeline: [============================================]
                2006                                  Feb 2025

Fold 1: Train [====]          | Val [==]
Fold 2: Train [======]        | Val [==]
Fold 3: Train [=========]     | Val [==]
Fold 4: Train [============]  | Val [==]
Fold 5: Train [==============]| Val [==]
                                        | Test [===] → Feb 2026
```

Each fold's training set grows by one block—simulating a model that retrains on all available history. Validation always falls *after* training chronologically. The test set (last year) is never touched during this entire stage.

### Why Not Standard K-Fold?

Standard K-fold randomly shuffles data before splitting. For a time series, this means:

- Training on June 2020 data while evaluating on March 2020 data
- The model sees future information during training
- Validation scores become unrealistically inflated

Expanding window CV avoids this by maintaining strict temporal order.

## Baseline Models

Baselines serve as the **minimum bar** any ML model must clear. They also reveal the practical difficulty of the task—if a simple rule achieves near-ML performance, the marginal value of complex models is limited.

We evaluate two baselines on all 5 validation folds:

### 1. Majority-Class Classifier

Always predicts class 0 (flat/down day). Achieves ~87% accuracy but F1 = 0 because it never predicts a buy signal.

```python
y_pred = np.zeros(len(y_val))  # always predict 0
```

This demonstrates the **accuracy paradox**: a high-accuracy model can be worthless. It also establishes why we use F1 as the optimisation metric.

### 2. MACD Momentum Rule

Predicts 1 (buy) when the MACD histogram is positive (short-term momentum bullish):

```python
y_pred = (X_val["MACD_hist"] > 0).astype(int)
```

This is a classic momentum strategy. It achieves moderate recall (it fires on many positive-momentum days) but poor precision (~11%)—it doesn't discriminate well between genuine ≥1% days and ordinary positive-momentum days.

### Baseline Results

![Baseline model comparison](/images/stock-return-classifier/05_baseline_comparison.png)

*Two baselines across validation folds: (1) majority-class — ~87% accuracy, F1=0, useless as a buy signal; (2) MACD momentum rule — moderate recall but low precision (~0.11). These set the minimum bar that ML models must clear to be considered useful.*

| Baseline | Avg Precision | Avg Recall | Avg F1 |
|----------|--------------|------------|--------|
| Majority Class | 0.000 | 0.000 | 0.000 |
| MACD Momentum | ~0.11 | ~0.55 | ~0.17 |

Any ML model that can't beat F1 ≈ 0.17 with better precision is not adding meaningful value over a simple rule.

## ML Models

Three models are trained with `class_weight="balanced"` (Logistic Regression, Random Forest) or `scale_pos_weight` (XGBoost) to handle the 6.7:1 class imbalance:

### Logistic Regression

A linear classifier with L2 regularisation. Simple, fast, and interpretable—the coefficients directly show which features push the probability toward class 1.

**Tuned parameter:**
- `C` (inverse regularisation strength): grid `[0.01, 0.1, 1, 10, 100]`

Lower C → stronger regularisation → simpler model. With 15+ features and ~4,800 training rows, moderate regularisation (C around 1–10) typically works well.

### Random Forest

An ensemble of decision trees where each tree is trained on a bootstrap sample with a random feature subset at each split. Inherently handles non-linear interactions between features.

**Tuned parameters:**

```json
{
  "n_estimators": [50, 100, 200],
  "max_depth": [3, 5, 10, null],
  "min_samples_split": [2, 5]
}
```

`max_depth=null` allows fully grown trees (high variance, low bias). `min_samples_split=5` requires at least 5 samples per leaf—a soft depth limiter that reduces overfitting.

### XGBoost

Gradient boosted trees—sequential ensemble where each tree corrects the errors of the previous. Generally the strongest performer on tabular data but requires more careful tuning to avoid overfitting.

**Tuned parameters:**

```json
{
  "n_estimators": [50, 100, 200],
  "max_depth": [3, 5, 7],
  "learning_rate": [0.01, 0.1, 0.3]
}
```

Low learning rate with more trees tends to generalise better than high learning rate with fewer trees.

## Hyperparameter Tuning

Grid search over all hyperparameter combinations, averaged across all 5 validation folds, optimising F1.

### Why F1 as the HPT Metric?

- **Accuracy** is misleading at 87/13 class split
- **Precision** alone optimises for avoiding false positives—but may produce a model that rarely fires
- **Recall** alone optimises for catching all up days—but floods the signal with false positives
- **F1** (harmonic mean of precision and recall) penalises both extremes and rewards balance

For a buy signal, we want *some* precision (not trading on every possible signal) and *some* recall (not missing all the big up days). F1 reflects this balance.

### HPT Implementation

```python
from itertools import product

best_score = -np.inf
for params in product(*param_grid.values()):
    param_dict = dict(zip(param_grid.keys(), params))
    model.set_params(**param_dict)

    fold_scores = []
    for train_fold, val_fold in expanding_window_folds:
        model.fit(X_train_fold, y_train_fold)
        y_pred = model.predict(X_val_fold)
        fold_scores.append(f1_score(y_val_fold, y_pred))

    avg_score = np.mean(fold_scores)
    if avg_score > best_score:
        best_score = avg_score
        best_params = param_dict
```

The best hyperparameters are then used to re-fit the model on the **full training set** (all 5 folds combined) before saving.

## Model Comparison

![Model comparison](/images/stock-return-classifier/06_model_comparison.png)

*All three models evaluated on honest cross-validation. Random Forest achieves the highest average validation F1, followed by Logistic Regression. XGBoost trails in recall—its conservative predictions miss too many up days at the fold level.*

| Model | Avg Val Precision | Avg Val Recall | Avg Val F1 |
|-------|------------------|---------------|-----------|
| Logistic Regression | ~0.22 | ~0.62 | ~0.32 |
| Random Forest | ~0.24 | ~0.67 | **~0.36** |
| XGBoost | ~0.26 | ~0.48 | ~0.32 |

**Random Forest** is selected as the best model. Its F1 advantage comes from better recall—it finds more of the actual up days without sacrificing too much precision.

XGBoost, despite its reputation for tabular data dominance, underperforms here in recall. The boosting iterations tend toward conservative predictions on the minority class, even with `scale_pos_weight`. Random Forest's bagging approach with balanced class weights proves more effective at this imbalance ratio.

## Learning Curves

Learning curves show how model performance evolves as training data grows—from 10% to 100% of the available training set. They answer two questions:

1. **Is the model overfitting?** (large gap between train and val scores)
2. **Would more data help?** (val score still rising at 100%)

![Learning curves](/images/stock-return-classifier/07_learning_curves.png)

*Train and validation F1 as training set size increases from 10% to 100%. The last validation fold is held out as a fixed evaluation set at all training sizes—preventing the evaluation set from shifting as the training pool grows.*

### No-Leakage Learning Curve Design

A subtle bug in naive learning curve implementations: if you use all 5 folds for evaluation, the validation set *changes* as training size grows (earlier training points borrow from folds that later become evaluation data). This inflates scores at small training sizes.

Our fix: **hold out the last fold as a fixed evaluation set** at all training sizes. The training pool grows from `folds 1–4` progressively, while `fold 5` always serves as the evaluation target.

```python
# Fixed eval set: last fold
X_eval, y_eval = val_folds[-1]

# Training pool: all folds except last
train_pool = pd.concat([f[0] for f in val_folds[:-1]])

for size in train_sizes:
    n = int(len(train_pool) * size)
    X_sub, y_sub = train_pool.iloc[:n], y_pool.iloc[:n]
    model.fit(X_sub, y_sub)

    train_score = f1_score(y_sub, model.predict(X_sub))
    val_score = f1_score(y_eval, model.predict(X_eval))
```

### What the Curves Show

- **Random Forest** shows the characteristic tree-ensemble pattern: high train F1 at small sizes (memorisation) that converges downward as more samples constrain the trees. The train-val gap closes as training data grows.
- **Logistic Regression** shows a smaller but persistent gap—lower variance but also lower capacity for the non-linear structure in the data.
- **Validation scores** plateau around 60–70% training size for all models—suggesting the current feature set is the bottleneck, not the volume of training data.

## Best Model: Random Forest

Random Forest is saved as `models/{project}/best_model.pkl` after re-fitting on the full training dataset (all validation folds included):

```python
best_rf = RandomForestClassifier(
    n_estimators=200,
    max_depth=10,
    min_samples_split=5,
    class_weight="balanced",
    random_state=42,
)
best_rf.fit(X_train_full, y_train_full)
joblib.dump(best_rf, "models/default_run/best_model.pkl")
```

Re-fitting on the full training set (not just the last fold) ensures maximum data utilisation before test evaluation.

## Key Takeaways

1. **Expanding window CV is non-negotiable** for financial time series. Standard K-fold would inflate scores and lead to overconfident model selection.
2. **F1 is the right HPT metric** at this imbalance ratio—optimising accuracy or precision alone leads to degenerate solutions.
3. **Random Forest outperforms XGBoost** here due to better recall on the minority class. Class imbalance handling via bootstrap bagging + balanced weights is effective.
4. **Learning curves plateau early**—additional training data contributes marginal improvement. Feature quality matters more than dataset size for this task.
5. **Learning curve leakage is subtle**: holding out a fixed evaluation fold prevents the evaluation set from shifting as training size grows.

## What's Next?

In **Part IV**, we evaluate the best model on the held-out test year—never-seen data—and walk through three portfolio strategies to quantify the economic value of the predictions.

---

**← Previous:** [Part II: EDA, Feature Selection & Feature Engineering](/posts/2026/02/17/stock-return-classifier-part-II)

**Next Post →** [Part IV: Test Evaluation & Portfolio Backtesting](/posts/2026/02/17/stock-return-classifier-part-IV)

---

## References

- [Scikit-learn Cross-Validation](https://scikit-learn.org/stable/modules/cross_validation.html) - scikit-learn docs
- [Random Forest Classifier](https://scikit-learn.org/stable/modules/ensemble.html#random-forests) - scikit-learn docs
- [XGBoost Documentation](https://xgboost.readthedocs.io/) - XGBoost
- [F1 Score and Imbalanced Classification](https://machinelearningmastery.com/tour-of-evaluation-metrics-for-imbalanced-classification/) - Machine Learning Mastery
- [Time Series Cross-Validation](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.TimeSeriesSplit.html) - scikit-learn docs

[src-part-I]: /posts/2026/02/17/stock-return-classifier-part-I
[src-part-II]: /posts/2026/02/17/stock-return-classifier-part-II
[src-part-III]: /posts/2026/02/17/stock-return-classifier-part-III
[src-part-IV]: /posts/2026/02/17/stock-return-classifier-part-IV
