---
title: 'Deep Q-Network for Stock Trading (Part I): Problem Statement & RL Motivation'
date: 2026-01-01 12:00:00 -0800
permalink: /posts/2026/01/01/dqn-trading-part-I
tags:
  - reinforcement learning
  - deep q-network
  - algorithmic trading
  - machine learning
redirect_from:
  - /2026/01/01/dqn-trading-part-I.html
toc: true
---

*This post is part of a series on building a Deep Q-Network (DQN) based trading system for SPY (S&P 500 ETF).*

- [Part I: Problem Statement & RL Motivation][dqn-part-I]
- [Part II: Data Engineering Pipeline][dqn-part-II]
- [Part III: Learning Environment Design][dqn-part-III]
- [Part IV: DQN Architecture Deep Dive][dqn-part-IV]
- [Part V: Software Architecture & Results][dqn-part-V]

---

**Next Post →** [Part II: Data Engineering Pipeline](/posts/2026/01/07/dqn-trading-part-II)

---

## ⚠️ Disclaimer

**This blog series is for educational and research purposes only.** The content should not be considered financial advice, investment advice, or trading advice. Trading stocks and financial instruments involves substantial risk of loss and is not suitable for every investor. Past performance does not guarantee future results. Always consult with a qualified financial advisor before making investment decisions.

---

## Introduction

After completing the [AI Trading Strategies Nanodegree Reinforcement Learning course](https://www.udacity.com/course/ai-trading-strategies--nd881), I wanted to apply reinforcement learning (RL) to a real-world trading problem. While many algorithmic trading approaches rely on supervised learning to predict price direction, I was intrigued by the potential of RL to learn **trading policies** directly—deciding when to buy, sell, or hold based on maximizing long-term profitability.

This blog series documents my journey building a Deep Q-Network (DQN) trading system for SPY, the most liquid ETF tracking the S&P 500 index. The goal is not just to build a profitable model, but to create a **production-ready system** with proper data engineering, backtesting, and risk management.

The complete code for this project is available on [GitHub](https://github.com/sharan-naribole/dqn-trading).

## Problem Statement

### The Trading Challenge

**Objective**: Build an intelligent agent that learns to trade SPY shares to maximize total returns while managing risk.

Unlike traditional approaches that predict whether prices will go up or down, we want our agent to learn:
- **When** to enter a position (and with how many shares)
- **When** to exit partially or completely
- **When** to stay in cash and wait

The agent should adapt to different market conditions:
- **Trending markets**: Ride the momentum, hold winners
- **Mean-reverting markets**: Buy dips, sell rallies
- **High-volatility periods**: Manage risk, avoid large drawdowns

### Why SPY?

I chose SPY (SPDR S&P 500 ETF) for several reasons:

1. **High Liquidity**: SPY is the most traded ETF globally with tight bid-ask spreads, making it ideal for algorithmic trading
2. **Lower Complexity**: Unlike individual stocks, SPY represents a diversified basket of 500 companies, reducing company-specific risk
3. **Historical Data**: Decades of reliable historical data available
4. **Real-world Relevance**: Many retail and institutional traders focus on SPY for both short-term and long-term strategies

### Financial Data Primer

For readers new to financial data, let's clarify some key concepts:

**Price Data:**
- **Open**: Price at market open
- **High/Low**: Highest and lowest prices during the trading day
- **Close**: Price at market close (used for most analysis)
- **Volume**: Number of shares traded

**Technical Indicators:**

Technical analysis uses historical price patterns to predict future movements. Common indicators include:

- **Moving Averages** (SMA, EMA): Smooth out price action to identify trends
- **Bollinger Bands**: Measure volatility using standard deviation bands around moving averages
- **RSI (Relative Strength Index)**: Momentum indicator measuring overbought/oversold conditions (0-100 scale)
- **ADX (Average Directional Index)**: Measures trend strength

**VIX (Volatility Index):**

The VIX measures market fear and uncertainty by tracking S&P 500 options prices. High VIX means high expected volatility, often associated with market downturns. Including VIX in our features helps the agent understand broader market sentiment.

## Why Reinforcement Learning?

Before diving into RL, let's understand why traditional supervised learning approaches fall short for trading:

### Limitations of Supervised Learning

**1. Sequential Decision Making**

Supervised learning predicts price direction at a single time step:
- "Will SPY go up or down tomorrow?"
- But trading involves **sequences of decisions** over time
- Each decision affects future states (cash balance, position size, entry price)

**2. Reward is Delayed**

- In trading, a buy decision only shows profit/loss after the eventual sell
- Supervised models don't naturally handle this **temporal credit assignment** problem
- RL explicitly models delayed rewards through discounting

**3. No Action Feedback Loop**

- Supervised learning: `State → Prediction`
- Trading reality: `State → Action → New State → Reward`
- The agent's actions **change the environment** (portfolio state)

**4. Static Policy vs. Adaptive Strategy**

- Supervised models learn a fixed mapping: features → signal
- RL learns a **policy** that adapts based on current portfolio state
- Example: The optimal action when holding 100 shares at a loss is different from holding 0 shares

### Reinforcement Learning Advantages

RL naturally addresses these challenges:

**1. Sequential Decision Process**

RL models trading as a **Markov Decision Process (MDP)**:
- **State (S)**: Market features + portfolio state (balance, shares held, entry price)
- **Action (A)**: Buy X shares, Sell Y shares, or Hold
- **Reward (R)**: Profit/loss from trading decisions
- **Transition**: `(State, Action) → New State`

The agent learns a **policy π(a|s)** that maximizes cumulative long-term reward:

```
Total Return = Σ γ^t * R_t
```

where γ (gamma) is the discount factor (0.99 in our case), emphasizing recent rewards more than distant ones.

**2. Exploration vs. Exploitation**

RL agents balance:
- **Exploration**: Try new strategies (e.g., buying during a crash)
- **Exploitation**: Use known profitable strategies (e.g., selling at resistance levels)

This is critical in trading where market dynamics evolve over time.

**3. Risk Management Through Rewards**

We can shape agent behavior through **reward engineering**:
- Positive rewards for profitable trades
- Negative rewards (penalties) for:
  - Excessive trading (transaction costs)
  - Holding idle cash too long (opportunity cost)
  - Large drawdowns (risk penalty)

**4. Direct Policy Learning**

Instead of predicting prices (which is notoriously difficult), we directly learn:
```
Optimal Action = argmax_a Q(state, action)
```

where Q(state, action) estimates the expected future return of taking action `a` in state `s`.

## Why Deep Q-Network (DQN)?

Among RL algorithms, I chose **DQN** for several reasons:

### 1. Discrete Action Space

DQN works well with discrete actions. For example, with `share_increments = [10, 50, 100, 200]` and `enable_buy_max = false`:
- Action 0: HOLD
- Action 1: BUY 10 shares
- Action 2: BUY 50 shares
- Action 3: BUY 100 shares
- Action 4: BUY 200 shares
- Action 5: SELL 10 shares
- Action 6: SELL 50 shares
- Action 7: SELL 100 shares
- Action 8: SELL 200 shares
- Action 9: SELL ALL

With `enable_buy_max = true`, we add BUY_MAX action (buy as many shares as balance allows), giving 11 total actions.

This discrete action space is more intuitive than policy gradient methods that handle continuous actions.

### 2. Off-Policy Learning

DQN uses **experience replay**:
- Store past experiences: `(state, action, reward, next_state)`
- Sample random mini-batches during training
- Breaks correlation between consecutive training samples
- More data-efficient than on-policy methods

### 3. Proven Track Record

DQN was famously used by DeepMind to:
- Play Atari games at superhuman level (2015)
- Defeat Go champion Lee Sedol via AlphaGo (which built on DQN concepts)

If it can master complex games, it can learn trading patterns.

### 4. Stable Training with Target Networks

DQN uses two networks:
- **Online Network**: Updated every training step
- **Target Network**: Periodically synchronized copy

This stabilizes learning by providing consistent Q-value targets during training. We'll dive deeper into this in Part IV.

## Project Scope

This project explores:

**1. Multi-Buy Position Accumulation**
- Scale into positions with multiple buys (e.g., buy 10 shares, then 50 more)
- Track weighted average entry price across lots
- Learn position sizing dynamically

**2. Partial Sells with FIFO Tracking**
- Sell portions of position: 10, 50, or all shares
- First-In-First-Out (FIFO) lot tracking for accurate profit calculation
- Exit strategies beyond binary buy/sell

**3. Risk Management Guardrails**
- Stop-loss: Automatically exit when position drops X% from weighted average entry
- Take-profit: Lock in gains at Y% profit
- Compare strategies with different risk tolerances (10%, 20%, or no guardrails)

**4. Out-of-Sample Validation**
- Train on 2022-2025 data
- Validate on 5 different market periods:
  - 2005 (pre-crisis)
  - 2008 (financial crisis)
  - 2012 (recovery)
  - 2013 (bull market)
  - 2021 (post-COVID)

**5. Production-Grade System**
- JSON-based configuration for experiments
- Modular architecture: data → features → environment → agent → results
- Comprehensive testing and dry-run mode
- GitHub-ready with documentation

## What's Next?

In **Part II**, we'll dive into the data engineering pipeline:
- Collecting SPY and VIX data from Yahoo Finance
- Feature engineering with 25 technical indicators
- Data splitting strategy to avoid lookahead bias
- Continuous rolling normalization for stationarity

Stay tuned!

---

**Next Post →** [Part II: Data Engineering Pipeline](/posts/2026/01/07/dqn-trading-part-II)

---

## References

- [Playing Atari with Deep Reinforcement Learning](https://arxiv.org/abs/1312.5602) - DeepMind's original DQN paper
- [Human-level control through deep reinforcement learning](https://www.nature.com/articles/nature14236) - DQN Nature paper
- [Reinforcement Learning: An Introduction](http://incompleteideas.net/book/the-book-2nd.html) - Sutton & Barto
- [AI Trading Strategies Nanodegree](https://www.udacity.com/course/ai-trading-strategies--nd881) - Udacity
- [SPY ETF Overview](https://www.ssga.com/us/en/individual/etfs/funds/spdr-sp-500-etf-trust-spy) - State Street Global Advisors

[dqn-part-I]: /posts/2026/01/01/dqn-trading-part-I
[dqn-part-II]: /posts/2026/01/07/dqn-trading-part-II
[dqn-part-III]: /posts/2026/01/14/dqn-trading-part-III
[dqn-part-IV]: /posts/2026/01/17/dqn-trading-part-IV
[dqn-part-V]: /posts/2026/01/18/dqn-trading-part-V
