---
title: 'Stock Return Classifier (Part IV): Test Evaluation & Portfolio Backtesting'
date: 2026-02-17 12:00:00 -0800
permalink: /posts/2026/02/17/stock-return-classifier-part-IV
tags:
  - machine learning
  - backtesting
  - portfolio simulation
  - algorithmic trading
redirect_from:
  - /2026/02/17/stock-return-classifier-part-IV.html
toc: true
---

*This post is part of a series on building a supervised ML pipeline to classify SPY daily returns.*

- [Part I: Problem Statement & Technical Indicators][src-part-I]
- [Part II: EDA, Feature Selection & Feature Engineering][src-part-II]
- [Part III: Baseline Models, ML Models & Hyperparameter Tuning][src-part-III]
- [Part IV: Test Evaluation & Portfolio Backtesting][src-part-IV]

---

**← Previous:** [Part III: Baseline Models, ML Models & Hyperparameter Tuning](/posts/2026/02/17/stock-return-classifier-part-III)

---

## ⚠️ Disclaimer

**This blog series is for educational and research purposes only.** The content should not be considered financial advice, investment advice, or trading advice. Trading stocks and financial instruments involves substantial risk of loss and is not suitable for every investor. Past performance does not guarantee future results. Always consult with a qualified financial advisor before making investment decisions.

---

## Introduction

In [Part III](/posts/2026/02/17/stock-return-classifier-part-III), we selected Random Forest as the best model based on validation F1. Now we evaluate it on the **held-out test set**—251 trading days from February 2025 to February 2026—data the model has never seen.

This is the moment of truth: validation scores are honest, but they reflect the past. Test evaluation tells us whether the patterns learned generalise to the most recent year of market data.

## Test Set: 251 Trading Days

The test period (Feb 2025 – Feb 2026) represents a challenging evaluation environment:

- SPY experienced multiple significant drawdowns and recoveries
- VIX spiked several times above 20 during tariff/geopolitical uncertainty
- The final year buy & hold return was +12.97%—a positive but not outlier year

All test predictions use the model trained on the full 2006–Feb 2025 training set with the best hyperparameters found in Part III.

## Classification Metrics

### Signal Distribution

On the 251 test days:
- **Total predictions**: 251
- **Predicted 1 (buy signal)**: 89 days (35.5%)
- **Actual 1 (≥1% up day)**: 25 days (10%)

The model fires more aggressively than the true positive rate—a direct consequence of using balanced class weights and optimising recall during training. This is a deliberate design choice: in the trading context, missing a big up day (false negative) is costly, so we accept some extra false alarms.

### Confusion Matrix

![Confusion matrix](/images/stock-return-classifier/08_confusion_matrix.png)

*Left: raw counts. Right: row-normalised percentages. The model correctly identifies 84% of actual up days (high recall) but also flags many flat/down days as up — yielding 23.6% precision. The normalised view makes class-level error rates directly comparable despite the imbalance.*

| Metric | Value |
|--------|-------|
| Accuracy | 0.713 |
| Precision | 0.236 |
| Recall | **0.840** |
| F1 | 0.368 |
| ROC-AUC | **0.810** |

**Recall of 0.84** means the model catches 84% of the 25 actual ≥1% up days in the test period. Out of 25 actual positive days, it misses only 4.

**Precision of 0.236** means that of the 89 buy signals generated, only 21 correspond to actual ≥1% up days. The other 68 signals trade on days that don't hit the 1% threshold—though many of these are profitable at smaller gains (addressed in the portfolio section below).

**AUC-ROC of 0.810** is the headline metric: the model correctly ranks a randomly chosen positive day above a randomly chosen negative day 81% of the time. This is strong ranking performance for daily return prediction.

### ROC Curve

![ROC curve](/images/stock-return-classifier/09_roc_curve.png)

*AUC of 0.810 on the held-out test set (vs. 0.726 at validation) indicates the model generalises well—and actually improves on test, suggesting the most recent year of data may be better represented by the patterns learned across the full 2006–2025 training history.*

The validation AUC (0.726) vs. test AUC (0.810) gap is worth noting. While this direction (test > val) can sometimes indicate lucky test period selection, it also reflects the fact that expanding window validation uses earlier historical folds that may be less representative of current market dynamics. The test period's market conditions appear well-captured by the full training history.

### Precision vs. Win Rate

An important distinction in the portfolio context:

- **Precision (23.6%)**: Among all 89 buy signals, 21 hit the ≥1% threshold—the specific target the classifier was trained on.
- **Win rate (61.4%)**: Among all trades executed, 61.4% close at a *higher* price than entry (next-day close > entry close).

These measure different things. Precision reflects the classification objective. Win rate reflects raw profitability at any positive return level. The gap between 23.6% and 61.4% means many signals—while not meeting the ≥1% threshold—still produce profitable trades.

## Calibration

A well-calibrated classifier's predicted probability of class 1 should match the actual fraction of positives at each probability level. If the model says 60% probability, approximately 60% of those days should be genuine ≥1% up days.

The calibration curve shows moderate miscalibration at the extremes—the model slightly overestimates probabilities in the mid-range. This matters for Strategy 2 and Strategy 3, which use probability thresholds to control position sizing. Well-calibrated probabilities ensure the thresholds have their intended meaning.

## Portfolio Backtesting

Classification metrics tell us how good the predictions are. Portfolio simulation translates predictions into economic outcomes.

### Setup

- **Initial balance**: $100,000
- **Hold period**: 1 day (buy at close, sell at next close)
- **Position sizing**: deploy maximum affordable shares on each signal
- **No transaction costs** (SPY has near-zero bid-ask spread for large accounts)

The 1-day hold period matches the prediction horizon exactly: we predicted whether *tomorrow* will be a ≥1% day, so we hold for exactly 1 day.

### Three Strategies + Buy & Hold

**Strategy 1**: Buy max affordable shares whenever `prob > 0.5`. Most aggressive—fires on any net-positive signal.

**Strategy 2**: Buy max affordable shares whenever `prob ≥ 0.6`. Higher confidence threshold—fires less often but with more conviction.

**Strategy 3**: Variable position sizing by probability bin:

| Probability | Position Size |
|-------------|--------------|
| ≥ 0.875 | 100% of max affordable |
| 0.75 – 0.875 | 75% |
| 0.625 – 0.75 | 50% |
| 0.50 – 0.625 | 25% |

The intent of Strategy 3 is to scale exposure proportionally with the model's conviction. In practice, with `max_shares=1000` and a $100K balance, all bins exceed the available capital—so all signals clamp to the same maximum affordable position. Strategy 3 becomes numerically identical to Strategy 1 in this run. The variable sizing design is preserved for configurations where max_shares is tighter relative to the balance.

**Buy & Hold**: Buy maximum shares at the start of the test period (day 1 close), hold through the entire test year.

### Portfolio Results

![Portfolio values over time](/images/stock-return-classifier/10_portfolio_performance.png)

*All four strategies start at $100,000 and are tracked daily across the 251-day test period. Strategy 1 and Buy & Hold are competitive throughout, with Strategy 1 finishing slightly ahead. Strategy 2 lags significantly due to insufficient trade frequency at the 0.6 threshold.*

| Strategy | Return | Final Value | Sharpe | Max Drawdown | Trades | Win Rate |
|----------|--------|-------------|--------|--------------|--------|----------|
| Strategy 1 (prob > 0.5) | **+13.72%** | $113,719 | **0.707** | -16.93% | 88 | 61.4% |
| Strategy 2 (prob ≥ 0.6) | +3.35% | $103,355 | 0.161 | -13.42% | 49 | 61.2% |
| Strategy 3 (variable sizing) | +13.72% | $113,719 | 0.707 | -16.93% | 88 | 61.4% |
| Buy & Hold | +12.97% | $112,972 | 0.628 | -18.65% | — | — |

### Analysis

**Strategy 1 vs. Buy & Hold**:

Strategy 1 outperforms buy & hold on all three risk-adjusted metrics:
- Return: +13.72% vs. +12.97% (+75 basis points)
- Sharpe ratio: 0.707 vs. 0.628 (+12.5% improvement)
- Max drawdown: -16.93% vs. -18.65% (172 bps lower)

The Sharpe improvement is the most meaningful result. It shows the model adds value not just by capturing more returns, but by doing so with lower volatility per unit of return—a sign that the buy signals are directionally informative, not just lucky.

**Why does Strategy 1 beat Buy & Hold?**

Buy & Hold is permanently invested—it participates in every down day and every flat day during the year. Strategy 1 holds for only 1 day at a time and is in cash the rest of the time. When in cash, it avoids prolonged drawdown periods. This produces a lower maximum drawdown despite similar total return.

**Strategy 2 underperforms**:

At the 0.6 threshold, Strategy 2 generates only 49 signals vs. 88 for Strategy 1. The reduced trade frequency cuts the number of winning trades proportionally, leading to much lower total return (+3.35%). The 0.6 threshold is too restrictive given the model's calibration: most of the model's useful signals fall in the 0.5–0.6 probability range.

The optimal threshold depends on how finely the model's probabilities are calibrated—a topic worth exploring further.

**Win Rate vs. Precision revisited**:

Both Strategy 1 and Strategy 2 achieve ~61% win rate despite precision of only 23.6%. This confirms the earlier observation: many buy signals produce profitable (positive return) trades even when they don't hit the ≥1% classification target. The classifier is conservative in calling a day a "big up day" but still identifies days that tend to close higher.

## Observations Summary

1. **AUC-ROC 0.810 on unseen test data** is the headline result. The model reliably ranks positive days above negative days, which is the fundamental requirement for a useful buy signal.

2. **High recall, low precision (84% / 24%)** reflects the deliberate design: we accept false alarms to avoid missing the rare ≥1% up days. This tradeoff is appropriate for a buy-only strategy where the cost of missing a big day is high.

3. **Strategy 1 beats buy & hold on return, Sharpe, and drawdown simultaneously**. This is rare—typically, improving returns requires accepting more risk.

4. **Strategy 2's 0.6 threshold is too aggressive**. The sweet spot appears to be near 0.5, where the model's calibration produces actionable signals with reasonable precision.

5. **Strategy 3 degeneracy**: with max_shares=1000 and $100K balance, SPY at ~$600 means even the 25% bin (250 shares × $600 = $150K) exceeds the balance. All bins clamp to the same amount. A meaningful variable-sizing strategy requires setting max_shares proportionally lower (e.g., max_shares=50 for graduated allocation).

6. **Cash buffer advantage**: by being out of the market on days with no signal, Strategy 1 avoids several of the year's sharp drawdown events—producing the lower maximum drawdown vs. Buy & Hold.

## Complete Pipeline Recap

Looking back across all four parts:

| Stage | Key Decision |
|-------|-------------|
| Data | yfinance with smart caching + 60-day warm-up buffer |
| Features | 22 indicators → 15 after EDA dropping |
| Transforms | log1p for Volume/BB_Width/ATR_pct; signed log1p for MACD |
| Normalisation | Rolling 63-day Z-score on continuous timeline |
| Splits | Expanding window, 5 folds, test = last 1 year |
| Baselines | Majority class (F1=0) + MACD momentum (F1≈0.17) |
| HPT metric | F1 (harmonic mean of precision + recall) |
| Best model | Random Forest (val F1 ≈ 0.36) |
| Test AUC-ROC | 0.810 |
| Best strategy | Strategy 1 (+13.72%, Sharpe 0.707, drawdown -16.93%) |

## Final Thoughts

This project set out to answer a specific question: can a binary classifier reliably identify days when SPY will gain ≥1%? The results are cautiously positive.

The model doesn't predict the future with certainty—precision of 23.6% means three out of four buy signals don't hit the 1% target. But an AUC-ROC of 0.810 shows the probability rankings are meaningful, and the portfolio results confirm the signals translate to risk-adjusted outperformance over a passive buy & hold strategy.

The practical limits are clear: the approach is market-regime dependent, ignores transaction costs at scale, and hasn't been stress-tested across crisis periods as a live strategy. The test period (Feb 2025 – Feb 2026) is a single 1-year window—a much longer out-of-sample test would be needed before drawing strong conclusions.

Still, the pipeline demonstrates a clean end-to-end methodology: principled feature engineering, leakage-free temporal validation, calibrated probability estimation, and economic evaluation via portfolio simulation. The framework is a solid foundation for further experimentation.

---

**← Previous:** [Part III: Baseline Models, ML Models & Hyperparameter Tuning](/posts/2026/02/17/stock-return-classifier-part-III)

---

## References

- [Confusion Matrix Explained](https://scikit-learn.org/stable/modules/model_evaluation.html#confusion-matrix) - scikit-learn docs
- [ROC Curve and AUC](https://scikit-learn.org/stable/modules/model_evaluation.html#roc-metrics) - scikit-learn docs
- [Probability Calibration](https://scikit-learn.org/stable/modules/calibration.html) - scikit-learn docs
- [Sharpe Ratio](https://www.investopedia.com/terms/s/sharperatio.asp) - Investopedia
- [Maximum Drawdown](https://www.investopedia.com/terms/m/maximum-drawdown.asp) - Investopedia
- [Backtesting Pitfalls](https://www.quantstart.com/articles/Successful-Backtesting-of-Algorithmic-Trading-Strategies-Part-I/) - QuantStart

[src-part-I]: /posts/2026/02/17/stock-return-classifier-part-I
[src-part-II]: /posts/2026/02/17/stock-return-classifier-part-II
[src-part-III]: /posts/2026/02/17/stock-return-classifier-part-III
[src-part-IV]: /posts/2026/02/17/stock-return-classifier-part-IV
