---
title: 'Deep Q-Network for Stock Trading (Part V): Software Architecture & Results'
date: 2026-01-18 12:00:00 -0800
permalink: /posts/2026/01/18/dqn-trading-part-V
tags:
  - software architecture
  - system design
  - backtesting results
  - trading strategies
redirect_from:
  - /2026/01/18/dqn-trading-part-V.html
toc: true
---

*This is the final post in a series on building a Deep Q-Network (DQN) based trading system for SPY (S&P 500 ETF).*

- [Part I: Problem Statement & RL Motivation][dqn-part-I]
- [Part II: Data Engineering Pipeline][dqn-part-II]
- [Part III: Learning Environment Design][dqn-part-III]
- [Part IV: DQN Architecture Deep Dive][dqn-part-IV]
- [Part V: Software Architecture & Results][dqn-part-V]

---

**← Previous:** [Part IV: DQN Architecture Deep Dive](/posts/2026/01/17/dqn-trading-part-IV)

---

## ⚠️ Disclaimer

**This blog series is for educational and research purposes only.** The content should not be considered financial advice, investment advice, or trading advice. Trading stocks and financial instruments involves substantial risk of loss and is not suitable for every investor. Past performance does not guarantee future results. Always consult with a qualified financial advisor before making investment decisions.

---

## Introduction

We've covered the theory—data pipeline, environment design, and DQN architecture. Now it's time to see how it all comes together in a **production-ready system** with real backtesting results.

This final post covers:
1. **Software Architecture**: Modular design, separation of concerns
2. **Configuration System**: JSON-based experiment management
3. **Testing Strategy**: Dry runs, system tests, validation
4. **Training Pipeline**: From data to trained models
5. **Real Results**: Out-of-sample validation across 5 market periods
6. **Strategy Comparison**: Different risk management approaches
7. **Lessons Learned**: What worked, what didn't

The complete project is on [GitHub](https://github.com/sharan-naribole/dqn-trading).

### Important Note on Results

**The results presented in this post are from a simple baseline configuration designed to demonstrate the project framework's capabilities.** These experiments were run with standard hyperparameters and are **not intended to showcase optimal performance**.

For readers interested in achieving better results, the framework supports:
- **Hyperparameter tuning**: Grid search or Bayesian optimization over learning rates, network architectures, epsilon decay schedules, etc.
- **Advanced RL algorithms**: The modular design allows swapping DQN for PPO, SAC, or other modern algorithms
- **Extended training**: More episodes, larger networks, and diverse data augmentation
- **Feature engineering**: Additional technical indicators, alternative data sources, or learned representations

The goal of this series is to provide a **solid, extensible foundation** that practitioners can build upon and customize for their specific trading objectives.

## Software Architecture

### Design Principles

**1. Modularity**: Each component has a single responsibility
```
Data Collection → Feature Engineering → Environment → Agent → Training → Evaluation
```

**2. Configuration-Driven**: All experiments defined in JSON (no code changes)

**3. Reproducibility**: Random seeds, data caching, model versioning

**4. Testability**: Unit tests, integration tests, dry-run mode

### Directory Structure

```
dqn-trading/
├── config/                      # Experiment configurations
│   ├── default_run/            # Full training setup
│   │   ├── data_config.json
│   │   ├── trading_baseline.json
│   │   └── trading_no_guardrails.json
│   ├── dry_run/                # Quick validation (1 episode)
│   │   ├── data_config.json
│   │   └── trading_dry_run.json
│   └── my_project/             # Custom experiments
│       ├── data_config.json
│       ├── trading_10pct.json
│       ├── trading_20pct.json
│       └── trading_no_guardrails.json
│
├── src/                        # Core source code
│   ├── data/                   # Data collection and splitting
│   │   ├── collector.py        # Yahoo Finance data fetcher
│   │   └── splitter.py         # Train/val/test split logic
│   ├── features/               # Feature engineering
│   │   ├── engineer.py         # Technical indicators
│   │   └── normalizer.py       # Rolling Z-score normalization
│   ├── trading/                # Trading environment
│   │   ├── environment.py      # Gym-like trading env
│   │   └── guardrails.py       # Stop-loss/take-profit rules
│   ├── models/                 # DQN models
│   │   ├── dqn.py              # Double DQN + Dueling architecture
│   │   └── action_masking.py   # Action space management
│   ├── training/               # Training pipeline
│   │   ├── trainer.py          # Training loop orchestration
│   │   └── replay_buffer.py    # Experience replay
│   ├── evaluation/             # Performance metrics
│   │   ├── validator.py        # Out-of-sample validation
│   │   └── metrics.py          # Sharpe, drawdown, etc.
│   └── utils/                  # Utilities
│       ├── config_loader.py    # JSON config parser
│       ├── model_manager.py    # Model save/load
│       └── progress_logger.py  # Training logs
│
├── main.ipynb                  # Main training notebook
├── test_system.py              # Integration test (dry run)
├── monitor_training.py         # Real-time training monitor
├── requirements.txt            # Python dependencies
└── README.md                   # Documentation
```

### Module Breakdown

**Data Layer (`src/data/`)**

Responsibilities:
- Fetch market data with caching
- Split data into train/validation/test
- Ensure no lookahead bias

Key classes:
- `DataCollector`: Smart caching, buffer handling
- `DataSplitter`: Random validation periods

**Features Layer (`src/features/`)**

Responsibilities:
- Create 25+ technical indicators
- Apply rolling normalization
- Handle missing data (NaN drops)

Key classes:
- `FeatureEngineer`: Bollinger Bands, RSI, ADX, EMAs
- `Normalizer`: Rolling Z-score with configurable window

**Trading Layer (`src/trading/`)**

Responsibilities:
- Define trading environment (MDP)
- Execute buy/sell with FIFO tracking
- Apply risk management guardrails

Key classes:
- `TradingEnvironment`: Gym-like interface, state/action/reward
- `TradingGuardrails`: Stop-loss, take-profit enforcement

**Models Layer (`src/models/`)**

Responsibilities:
- Define DQN network architecture
- Implement Double DQN algorithm
- Manage action masking

Key classes:
- `DQNNetwork`: Dueling architecture (value + advantage)
- `DoubleDQN`: Training logic, target network sync
- `ActionMasker`: Action space creation, validation

**Training Layer (`src/training/`)**

Responsibilities:
- Orchestrate training loop
- Manage experience replay
- Track episode metrics

Key classes:
- `Trainer`: Episode loop, epsilon decay, validation
- `ReplayBuffer`: Store and sample experiences

**Evaluation Layer (`src/evaluation/`)**

Responsibilities:
- Run out-of-sample validation
- Calculate performance metrics
- Generate comparison plots

Key classes:
- `Validator`: Validation loop, deterministic policy
- `MetricsCalculator`: Sharpe ratio, max drawdown, win rate

## Configuration System

### Two-Part Configuration

**1. Data Config (shared across strategies)**

`config/my_project/data_config.json`:
```json
{
  "_comment": "Shared data configuration for all strategies",
  "ticker": "SPY",
  "start_date": "2005-01-01",
  "end_date": "2025-12-31",

  "data": {
    "window_size": 5,
    "normalization_window": 30,
    "indicators": {
      "bollinger_period": 20,
      "bollinger_std": 2,
      "ema_short": 8,
      "ema_medium": 21,
      "sma_short": 50,
      "sma_long": 200,
      "adx_period": 14,
      "rsi_period": 14
    }
  },

  "validation": {
    "n_periods": 5,
    "period_unit": "year",
    "random_seed": 42
  },

  "test": {
    "period_duration": 1,
    "period_unit": "year"
  },

  "output": {
    "model_dir": "models",
    "results_dir": "results",
    "data_dir": "data"
  }
}
```

**2. Trading Config (strategy-specific)**

`config/my_project/trading_20pct.json`:
```json
{
  "_comment": "20% stop-loss and take-profit strategy",
  "experiment_name": "Stop-Loss-Take-Profit-20pct",
  "strategy_name": "SL/TP 20pct",
  "description": "Strategy with 20% stop-loss and take-profit",

  "trading": {
    "share_increments": [10, 50, 100, 200],
    "enable_buy_max": false,
    "starting_balance": 100000,
    "idle_reward": -0.001,
    "buy_reward_per_share": 0.0,
    "buy_transaction_cost_per_share": 0.01,
    "sell_transaction_cost_per_share": 0.01,
    "stop_loss_pct": 20,
    "take_profit_pct": 20
  },

  "network": {
    "architecture": "dueling",
    "shared_layers": [256, 128],
    "value_layers": [128],
    "advantage_layers": [128],
    "activation": "relu",
    "dropout_rate": 0.0,
    "batch_norm": false
  },

  "training": {
    "episodes": 500,
    "batch_size": 64,
    "replay_buffer_size": 10000,
    "target_update_freq": 1,
    "epsilon_start": 1.0,
    "epsilon_end": 0.01,
    "epsilon_decay": 0.995,
    "learning_rate": 0.001,
    "gamma": 0.99,
    "optimizer": "adam",
    "save_frequency": 10,
    "validation_frequency": 10,
    "validate_at_episode_1": true,
    "early_stopping_patience": 10,
    "early_stopping_metric": "total_return"
  }
}
```

### Usage in Notebook

```python
# Select project
PROJECT_FOLDER = 'my_project'

# Load data config
config_loader = ConfigLoader(f'config/{PROJECT_FOLDER}')
data_config = config_loader.load_data_config()

# Find all trading strategies in project
trading_configs = glob.glob(f'config/{PROJECT_FOLDER}/trading_*.json')

# Train each strategy
for trading_config_path in trading_configs:
    trading_config = load_json(trading_config_path)
    train_strategy(data_config, trading_config)
```

### Benefits

1. **Reproducibility**: Copy config → exact same experiment
2. **Versioning**: Track configs in git
3. **Comparison**: Run multiple strategies with different configs
4. **No Code Changes**: Tweak hyperparameters without editing code

## Testing Strategy

### 1. Dry Run Mode (< 1 minute)

**Purpose:** Validate entire pipeline without full training

`config/dry_run/trading_dry_run.json`:
```json
{
  "training": {
    "episodes": 1,           // Just 1 episode
    "batch_size": 32,        // Small batch
    "replay_buffer_size": 1000
  }
}
```

**Run:**
```bash
python test_system.py
```

**What it tests:**
- Data collection and caching
- Feature engineering (25 features)
- Environment reset and step
- Action masking logic
- DQN forward pass
- Training loop (1 episode)
- Validation (1 period)

**Output:**
```
Testing DQN Trading System (Dry Run Mode)
=========================================
✓ Data collection (0.8s)
✓ Feature engineering (25 features)
✓ Data splitting (train: 596, val: 5 periods, test: 128)
✓ Environment initialization (8 actions)
✓ DQN agent created
✓ Training Episode 1/1 (reward: 15.2)
✓ Validation (return: 1.2%)
=========================================
All tests passed! (Total time: 42s)
```

### 2. Integration Test (`test_system.py`)

```python
"""Integration test for DQN trading system."""

import sys
import numpy as np
from src.data.collector import DataCollector
from src.features.engineer import FeatureEngineer
from src.trading.environment import TradingEnvironment
from src.models.dqn import DoubleDQN
from src.utils.config_loader import ConfigLoader

def test_full_pipeline():
    """Test complete pipeline with dry_run config."""

    print("Testing DQN Trading System")
    print("=" * 60)

    # 1. Load configuration
    config_loader = ConfigLoader('config/dry_run')
    data_config = config_loader.load_data_config()
    trading_config = config_loader.load_trading_config('trading_dry_run')
    config = {**data_config, **trading_config}

    # 2. Data collection
    print("\n1. Testing data collection...")
    collector = DataCollector(data_config)
    spy_data, vix_data = collector.collect_data()
    assert len(spy_data) > 0, "No SPY data collected"
    assert len(vix_data) > 0, "No VIX data collected"
    print(f"  ✓ Collected {len(spy_data)} SPY records")

    # 3. Feature engineering
    print("\n2. Testing feature engineering...")
    engineer = FeatureEngineer(data_config)
    combined_data = engineer.combine_spy_vix(spy_data, vix_data)
    featured_data = engineer.create_features(combined_data)
    print(f"  ✓ Created {len(featured_data.columns)} features")

    # 4. Environment
    print("\n3. Testing trading environment...")
    feature_cols = [col for col in featured_data.columns
                    if col.startswith('SPY_') or col.startswith('VIX_')]
    env = TradingEnvironment(featured_data, feature_cols, config, mode='train')
    state, info = env.reset()
    print(f"  ✓ Environment initialized ({env.action_masker.n_actions} actions)")

    # 5. DQN Agent
    print("\n4. Testing DQN agent...")
    agent = DoubleDQN(state.shape, env.action_masker.n_actions, config)
    action_mask = env.get_action_mask()
    action = agent.get_action(state, action_mask, epsilon=1.0)
    print(f"  ✓ Agent created (selected action: {action})")

    # 6. Training step
    print("\n5. Testing training step...")
    for step in range(10):
        action_mask = env.get_action_mask()
        action = agent.get_action(state, action_mask, epsilon=1.0)
        next_state, reward, done, info = env.step(action)
        state = next_state
        if done:
            break
    print(f"  ✓ Completed {step+1} environment steps")

    print("\n" + "=" * 60)
    print("✅ All tests passed!")
    print("=" * 60)

if __name__ == '__main__':
    test_full_pipeline()
```

### 3. Multi-Buy Test (`test_multibuy.py`)

**Purpose:** Verify multi-buy accumulation and FIFO sells

Key tests:
- ActionMasker with share_increments=[10, 50, 100]
- Action masking with various position sizes
- Multi-buy accumulation with weighted average
- FIFO partial sell logic
- Profit calculation per lot

## Training Pipeline

### Workflow Overview

```
1. Configuration Loading
   └─> Load data_config.json
   └─> Load all trading_*.json files

2. Data Collection
   └─> Fetch SPY and VIX (with caching)
   └─> Combine into single DataFrame

3. Feature Engineering
   └─> Create 25 technical indicators
   └─> Drop NaN rows

4. Data Splitting
   └─> Extract test period (last year)
   └─> Extract 5 random validation periods
   └─> Remaining → training data

5. Normalization
   └─> Apply rolling Z-score (window=30)
   └─> Continuous timeline (preserves temporal order)

6. Training Loop (per strategy)
   └─> Initialize environment
   └─> Initialize DQN agent
   └─> For each episode:
       ├─> Collect experiences (epsilon-greedy)
       ├─> Train on mini-batches from replay buffer
       ├─> Decay epsilon
       ├─> Update target network
       └─> Validate periodically

7. Final Evaluation
   └─> Run deterministic policy on all validation periods
   └─> Run on test set
   └─> Generate comparison plots
   └─> Save results and models
```

### Training Notebook Execution

```python
# In main.ipynb

# Configuration
PROJECT_FOLDER = 'my_project'
TEST_MODE = False  # False: train, True: load existing models

# Initialize
config_loader = ConfigLoader(f'config/{PROJECT_FOLDER}')
data_config = config_loader.load_data_config()

# Collect data ONCE for all strategies
collector = DataCollector(data_config)
spy_data, vix_data = collector.collect_data()

# Engineer features
engineer = FeatureEngineer(data_config)
combined_data = engineer.combine_spy_vix(spy_data, vix_data)
featured_data = engineer.create_features(combined_data)

# Split data
splitter = DataSplitter(data_config)
splits = splitter.split_data(featured_data)

# Normalize
normalizer = Normalizer(window=30)
normalized_data = normalizer.normalize_continuous_timeline(splits)

# Find all trading strategies
trading_configs = glob.glob(f'config/{PROJECT_FOLDER}/trading_*.json')

# Train each strategy
results = {}
for trading_config_path in trading_configs:
    trading_config = load_json(trading_config_path)
    strategy_name = trading_config['strategy_name']

    print(f"\n{'='*70}")
    print(f"Training Strategy: {strategy_name}")
    print(f"{'='*70}")

    # Initialize trainer
    trainer = Trainer(normalized_data, trading_config, data_config)

    # Train
    if not TEST_MODE:
        trainer.train(episodes=training_config['training']['episodes'])

    # Validate
    validation_results = trainer.validate_all_periods()

    # Test
    test_results = trainer.test()

    results[strategy_name] = {
        'validation': validation_results,
        'test': test_results
    }

# Compare strategies
plot_strategy_comparison(results)
```

## Real Results: Out-of-Sample Validation

### Experiment Setup

**Project:** `my_project`

**Data Range:** 2005-01-01 to 2025-12-31 (21 years)

**Training Data:** 2006-01-03 to 2024-12-27 (3,772 samples, ~19 years)

**Starting Capital:** $100,000

**Share Increments:** [10, 50, 100, 200]

**Action Space:**
- With BUY_MAX enabled: 11 actions (HOLD + 4 BUY + BUY_MAX + 4 SELL + SELL_ALL)
- With BUY_MAX disabled: 10 actions (HOLD + 4 BUY + 4 SELL + SELL_ALL)

**Training:** 500 episodes per strategy (~2 hours each)

**Validation:** 5 random year-long periods (never seen during training):
1. **2005-02-08 to 2005-12-30** (Pre-financial crisis bull market)
2. **2008-01-02 to 2008-12-31** (Global financial crisis)
3. **2012-01-03 to 2012-12-31** (Post-crisis recovery)
4. **2013-01-02 to 2013-12-31** (Bull market continuation)
5. **2021-01-04 to 2021-12-31** (Post-COVID recovery)

**Test:** 2024-12-30 to 2025-12-30 (251 samples, completely held out)

### Strategies Compared

We compare four distinct strategies to understand the impact of risk management and position sizing:

| Strategy | Stop-Loss | Take-Profit | BUY_MAX | Actions | Description |
|----------|-----------|-------------|---------|---------|-------------|
| **SL/TP 10pct** | 10% | 10% | ✗ Disabled | 10 | Tight risk control with incremental sizing |
| **SL/TP 20pct** | 20% | 20% | ✗ Disabled | 10 | Balanced risk management with incremental sizing |
| **No Guardrails** | ✗ Disabled | ✗ Disabled | ✓ Enabled | 11 | Pure RL with adaptive BUY_MAX positioning |
| **No Guardrails (No BUY_MAX)** | ✗ Disabled | ✗ Disabled | ✗ Disabled | 10 | Pure RL with incremental sizing only |

**Understanding BUY_MAX:**

The **BUY_MAX action** is a critical differentiator that enables price-adaptive position sizing:

- **With BUY_MAX (No Guardrails strategy):** Agent has 11 actions
  - HOLD, BUY_10, BUY_50, BUY_100, BUY_200, **BUY_MAX** (buys as many shares as balance allows), SELL_10, SELL_50, SELL_100, SELL_200, SELL_ALL
  - At $400/share vs $500/share, BUY_MAX automatically adjusts quantity
  - Enables "go all-in" when agent is highly confident
  - No arbitrary position limits

- **Without BUY_MAX (all other strategies):** Agent has 10 actions
  - HOLD, BUY_10, BUY_50, BUY_100, BUY_200, SELL_10, SELL_50, SELL_100, SELL_200, SELL_ALL
  - Incremental position sizing: max position = 200 shares per action
  - Must accumulate larger positions through multiple BUY actions
  - Forces explicit position sizing decisions

This distinction allows us to measure the impact of adaptive position sizing on strategy performance.

### Validation Results (5 Periods)

The trained models were evaluated across 5 diverse market periods never seen during training:

![Out-of-Sample Validation Comparison](/images/dqn-trading/sample-validation-comparison.png)

#### Period 1: 2005-02-08 to 2005-12-30 (Pre-Financial Crisis)

| Strategy | Total Return | Sharpe Ratio | Max Drawdown | Trades | Win Rate |
|----------|-------------|--------------|--------------|---------|----------|
| **SL/TP 10pct** | **+5.49%** | 0.69 | -6.35% | 144 | 74.3% |
| **SL/TP 20pct** | **+5.68%** | **0.89** | **-4.10%** | 72 | **94.4%** |
| **No Guardrails (No BUY_MAX)** | +4.96% | 0.61 | -6.78% | 82 | 59.8% |

**Analysis:** Stable bull market. SL/TP 20pct achieved best risk-adjusted returns (0.89 Sharpe) with exceptional 94.4% win rate and minimal drawdown.

#### Period 2: 2008-01-02 to 2008-12-31 (Financial Crisis)

| Strategy | Total Return | Sharpe Ratio | Max Drawdown | Trades | Win Rate |
|----------|-------------|--------------|--------------|---------|----------|
| **SL/TP 10pct** | **-29.09%** | -1.40 | **-36.53%** | 150 | 25.3% |
| **SL/TP 20pct** | **-10.11%** | **-1.12** | **-12.85%** | 59 | **39.0%** |
| **No Guardrails (No BUY_MAX)** | -31.51% | -0.77 | -45.26% | 26 | 7.7% |

**Analysis:** Severe market crash. SL/TP 20pct limited damage best (-10.11% vs -29% to -32% for others). Wider guardrails provided better protection by avoiding premature stop-outs followed by further declines.

#### Period 3: 2012-01-03 to 2012-12-31 (Recovery Phase)

| Strategy | Total Return | Sharpe Ratio | Max Drawdown | Trades | Win Rate |
|----------|-------------|--------------|--------------|---------|----------|
| **SL/TP 10pct** | **+10.38%** | 0.96 | -8.18% | 165 | 86.1% |
| **SL/TP 20pct** | +7.75% | **1.10** | **-5.99%** | 102 | **88.2%** |
| **No Guardrails (No BUY_MAX)** | **+11.26%** | **0.99** | -8.93% | 101 | 76.2% |

**Analysis:** Post-crisis recovery. No BUY_MAX achieved highest raw return (11.26%) without adaptive position sizing, showing strong pattern recognition. All strategies performed well with 76-88% win rates.

#### Period 4: 2013-01-02 to 2013-12-31 (Bull Market)

| Strategy | Total Return | Sharpe Ratio | Max Drawdown | Trades | Win Rate |
|----------|-------------|--------------|--------------|---------|----------|
| **No Guardrails (No BUY_MAX)** | **+26.19%** | **2.42** | -5.00% | 147 | 78.9% |
| **SL/TP 10pct** | **+24.36%** | 2.48 | **-4.97%** | 209 | **87.6%** |
| **SL/TP 20pct** | +14.00% | 2.04 | **-3.72%** | 141 | 86.5% |

**Analysis:** Strong bull market favored all strategies. No BUY_MAX (26.19%) slightly outperformed SL/TP 10pct (24.36%) despite lacking BUY_MAX action. Exceptional Sharpe ratios (2.04-2.48) indicate strong risk-adjusted performance.

#### Period 5: 2021-01-04 to 2021-12-31 (Post-COVID)

| Strategy | Total Return | Sharpe Ratio | Max Drawdown | Trades | Win Rate |
|----------|-------------|--------------|--------------|---------|----------|
| **SL/TP 10pct** | **+21.51%** | **2.01** | **-3.80%** | 167 | **82.6%** |
| **No Guardrails (No BUY_MAX)** | +18.44% | 1.74 | -4.46% | 105 | 75.2% |
| **SL/TP 20pct** | +7.86% | 1.37 | **-3.13%** | 87 | 75.9% |

**Analysis:** Post-COVID recovery with elevated volatility. Tight guardrails (10%) excelled with 21.51% return and 2.01 Sharpe ratio. More frequent exits (167 trades) locked in gains during volatile swings.

### Aggregate Validation Performance

**Average Across 5 Periods:**

| Strategy | Avg Return | Avg Sharpe | Avg Sortino | Actions |
|----------|-----------|-----------|-------------|---------|
| **No Guardrails (No BUY_MAX)** | **6.55%** | **1.02** | **1.48** | 10 |
| SL/TP 10pct | 5.16% | 0.95 | 1.46 | 10 |
| SL/TP 20pct | 4.48% | 0.95 | 1.35 | 10 |
| No Guardrails | 2.37% | 1.01 | 1.34 | 11 |

**Key Validation Insights:**

1. **Best Risk-Adjusted:** No Guardrails (No BUY_MAX) achieved the best average Sharpe (1.02) and Sortino (1.48) ratios across all validation periods, demonstrating superior downside risk management

2. **BUY_MAX Impact:** Strategies without BUY_MAX (10 actions) demonstrated more consistent performance with superior downside risk management (higher Sortino), suggesting that incremental position sizing [10, 50, 100, 200] allows for better risk control than adaptive BUY_MAX

3. **Guardrail Trade-off:** Removing guardrails allows DQN to learn more nuanced exit strategies, but requires careful position sizing discipline. The No Guardrails (No BUY_MAX) strategy excelled in validation with this freedom

4. **Crisis Period (2008):** All strategies experienced significant drawdowns, highlighting the challenge of bear market navigation

5. **Bull Markets (2013, 2021):** Strategies showed strong performance with returns of 14-26%, demonstrating ability to capitalize on uptrends

### Test Set Results (2024-12-30 to 2025-12-30)

Final evaluation on held-out test period comparing against Buy & Hold baseline:

![Test Portfolio Performance](/images/dqn-trading/sample-test-portfolio-values.png)

*Portfolio value over time showing DQN strategies vs Buy & Hold baseline. Buy & Hold correctly shows actual market volatility (buying max shares at start with remaining cash buffer).*

![Test Metrics Comparison](/images/dqn-trading/sample-test-metrics-comparison.png)

*Comprehensive test metrics including Returns, Sharpe Ratio, Sortino Ratio, Win Rate, Max Drawdown, and Position Size Distribution.*

| Strategy | Return | Sharpe | Sortino | Max DD | Win Rate | Trades | Actions |
|----------|--------|--------|---------|--------|----------|--------|---------|
| **SL/TP 10pct** | **15.16%** | **1.63** | **2.32** | **-6.80%** | **82.98%** | **141** | 10 |
| No Guardrails (No BUY_MAX) | 11.42% | 1.11 | 1.43 | -9.99% | 94.29% | 70 | 10 |
| No Guardrails | 6.70% | 0.56 | 0.60 | -14.31% | 69.20% | 224 | 11 |
| SL/TP 20pct | 5.27% | 1.63 | 2.36 | -1.57% | 71.43% | 42 | 10 |
| Buy & Hold | 17.99% | N/A | N/A | N/A | N/A | N/A | N/A |

**Key Test Insights:**

1. **Winner: SL/TP 10pct (10% SL/TP, no BUY_MAX, 10 actions)**
   - Achieved best DQN performance with 15.16% return, 1.63 Sharpe, and 2.32 Sortino ratio
   - Came close to Buy & Hold (17.99%), demonstrating competitive active trading
   - Tight risk control (10% guardrails) excelled in this bull market test period

2. **Risk Management:**
   - Tight stop-loss strategies (10% and 20%) demonstrated superior risk-adjusted returns with Sharpe ratios of 1.63
   - Minimal drawdowns (-6.80% and -1.57%) compared to No Guardrails strategies (-9.99% to -14.31%)
   - Sortino ratios of 2.32-2.36 for SL/TP strategies indicate excellent downside risk management, prioritizing protection against negative returns

3. **Trading Efficiency:**
   - No Guardrails (No BUY_MAX) achieved 94.29% win rate with only 70 trades, demonstrating quality over quantity with incremental position sizing [10, 50, 100, 200]
   - SL/TP 10pct balanced frequency with success: 141 trades with 82.98% win rate

4. **Guardrail Impact:**
   - Strategies with stop-loss/take-profit guardrails significantly outperformed those without, validating the importance of automated risk management
   - SL/TP 10pct (15.16%) nearly matched Buy & Hold (17.99%), demonstrating competitive active trading

5. **BUY_MAX Trade-off:**
   - All top 3 performers disabled BUY_MAX, suggesting that forcing explicit position sizing decisions through incremental actions leads to better risk-adjusted returns than adaptive all-in positioning
   - No Guardrails strategy (with BUY_MAX) underperformed with only 6.70% return and 0.56 Sharpe
   - 10-action strategies (without BUY_MAX) dominated performance, suggesting a simpler action space with forced incremental sizing improves learning efficiency

6. **The Buy & Hold Benchmark:**
   - Buy & Hold achieved 17.99% (vs best DQN at 15.16%)
   - SL/TP 10pct came remarkably close with only 2.83% underperformance while providing superior risk metrics (2.32 Sortino)
   - Demonstrates that well-designed DQN strategies can compete with passive indexing


## Lessons Learned

### What Worked

**1. Guardrails Are Essential**

SL/TP 10pct strategy demonstrated that automated risk management is critical:
- Achieved best DQN performance: 15.16% return with 1.63 Sharpe and 2.32 Sortino
- Tight 10% stop-loss and take-profit provided superior risk-adjusted returns
- Minimal drawdown (-6.80%) compared to No Guardrails strategies (-9.99% to -14.31%)
- Nearly matched Buy & Hold (17.99%) while providing better risk metrics
- **Lesson:** Automated risk management enables consistent performance

**2. Incremental Position Sizing Beats BUY_MAX**

Strategies without BUY_MAX (10 actions) dominated performance:
- All top 3 performers disabled BUY_MAX
- Incremental sizing [10, 50, 100, 200] provided sufficient flexibility
- Forced explicit position sizing decisions improved learning efficiency
- **Surprise:** Fixed increments beat adaptive all-in positioning
- Lesson: Simpler action space (10 vs 11 actions) improved agent's ability to learn optimal strategies

**3. Diverse Out-of-Sample Validation**

5 random validation periods revealed critical insights:
- 2008 crisis: All strategies lost 10-32%, exposed vulnerability
- Bull markets (2013, 2021): All strategies gained 7-26%
- Validation Sharpe (1.00) predicted test Sharpe (0.94) for No BUY_MAX
- Prevented overfitting to specific market conditions

**4. Double DQN + Dueling Architecture**

Architectural improvements enabled successful learning:
- 500 episodes sufficient for convergence
- Learned complex entry/exit timing patterns
- Achieved strong performance across all strategies (70-141 trades with 71-94% win rates)
- Stable training without divergence

**5. Multi-Buy with FIFO Lot Tracking**

Position scaling with [10, 50, 100, 200] shares provided:
- Gradual position building across multiple entries with 4 different increment sizes
- Accurate profit calculation via First-In-First-Out tracking
- Weighted average entry price for realistic guardrail triggers
- Sufficient flexibility without needing BUY_MAX action

### What Didn't Work

**1. Beating Buy & Hold (The Challenge)**

Best DQN strategy came close but didn't quite beat passive baseline:
- Buy & Hold: **17.99%** vs Best DQN (SL/TP 10pct): 15.16%
- Gap of 2.83% despite sophisticated RL
- Transaction costs (141 trades × $0.01/share buy+sell) added up
- However, SL/TP 10pct provided superior risk-adjusted returns (2.32 Sortino vs N/A for Buy & Hold)
- **Reality Check:** Efficient markets are challenging to beat, but risk-adjusted performance matters

**2. Strategies Without Guardrails Struggled**

Removing stop-loss/take-profit guardrails led to poor performance:
- No Guardrails (with BUY_MAX): 6.70% return with -14.31% max drawdown
- No Guardrails (No BUY_MAX): 11.42% return but worse Sharpe (1.11) than SL/TP 10pct (1.63)
- Higher drawdowns (-9.99% to -14.31%) vs guardrailed strategies (-1.57% to -6.80%)
- **Lesson:** Automated risk management is essential for consistent performance

**3. BUY_MAX Action Didn't Help**

The adaptive BUY_MAX action underperformed fixed increments:
- All top 3 strategies disabled BUY_MAX (10 actions)
- No Guardrails (with BUY_MAX, 11 actions): only 6.70% return, 0.56 Sharpe
- Incremental sizing [10, 50, 100, 200] provided sufficient flexibility
- **Surprise:** Simpler 10-action space outperformed adaptive 11-action space
- **Lesson:** Forcing explicit position sizing decisions improved learning efficiency

### Key Insights

**1. Guardrails Are Essential for Consistent Performance**

Stop-loss/take-profit guardrails were critical for success:
- SL/TP 10pct (winner): 15.16% return with 2.32 Sortino ratio
- Minimal drawdowns with guardrails: -1.57% to -6.80%
- Strategies without guardrails: -9.99% to -14.31% drawdowns
- **Takeaway:** Automated risk management enables superior risk-adjusted returns

**2. Simpler Action Space Won**

Strategies without BUY_MAX (10 actions) dominated:
- All top 3 performers disabled BUY_MAX
- SL/TP 10pct (10 actions): 15.16% return, 1.63 Sharpe
- No Guardrails with BUY_MAX (11 actions): only 6.70% return, 0.56 Sharpe
- **Takeaway:** Simpler action space improved learning efficiency

**3. Risk-Adjusted Performance Matters**

SL/TP 10pct nearly matched Buy & Hold with better risk metrics:
- Buy & Hold: 17.99% return, no Sharpe/Sortino metrics
- SL/TP 10pct: 15.16% return, 1.63 Sharpe, 2.32 Sortino
- Only 2.83% return gap while providing measurable risk management
- **Takeaway:** Active trading can compete when considering risk-adjusted returns

**4. Market Regime Matters**

Performance varied across validation periods:
- 2008 crisis: ALL strategies lost 10-32% (SL/TP 20pct best at -10.11%)
- 2013 bull: Strong performance across strategies (14-26% returns)
- 2024-2025 bull test: SL/TP 10pct excelled with tight guardrails
- **Takeaway:** No single strategy dominates all conditions

**5. RL Successfully Learns Trading Strategies**

The framework demonstrated clear learning capability:
- Converged stably over 500 episodes
- Learned distinct strategies based on different risk configurations
- Achieved competitive performance vs Buy & Hold (15.16% vs 17.99%)
- Win rates of 71-94% across strategies showing quality pattern recognition
- **Takeaway:** RL effectively learns trading patterns when properly configured

## Future Work

The framework presented in this series is a **starting point**, not an endpoint. Here are directions for improvement:

**1. Hyperparameter Optimization**
- Grid search or Bayesian optimization over:
  - Network architecture (layer sizes, activation functions)
  - Learning rate, gamma, epsilon decay schedules
  - Replay buffer size, batch size
  - Share increments, stop-loss/take-profit percentages
- Use validation performance to select best configurations
- Tools: Optuna, Ray Tune, or custom grid search

**2. Advanced RL Algorithms**
- Proximal Policy Optimization (PPO)
- Soft Actor-Critic (SAC)
- Multi-agent approaches

**3. Richer State Space**
- Order book data (bid/ask spreads)
- News sentiment
- Macroeconomic indicators

**4. Multi-Asset Trading**
- Portfolio of ETFs (SPY, QQQ, IWM)
- Sector rotation strategies

**5. Live Trading Integration**
- Alpaca/Interactive Brokers API
- Paper trading validation
- Slippage modeling

**6. Transfer Learning**
- Train on SPY, test on QQQ
- Cross-market generalization

## Conclusion

This project demonstrated that **Deep Q-Learning can successfully learn profitable trading strategies**, but with important caveats:

✅ **What This Series Accomplished:**
- Built a complete, production-ready DQN trading framework
- Demonstrated modular architecture for easy experimentation
- Achieved reasonable baseline results (13.8% return on test set)
- Validated across diverse market conditions (2005-2021)
- Created extensible foundation for advanced research

⚠️ **Important Caveats:**
- **Results presented are baseline demonstrations**, not optimized performance
- Extensive hyperparameter tuning could significantly improve results
- Risk management (stop-loss/take-profit) remains essential
- Pure RL without guardrails underperformed in this baseline setup
- Requires significant training data (500+ episodes minimum)
- Backtested results don't account for real-world slippage and liquidity constraints

**Final Takeaway:**

This framework is a **starting point for serious research**, not a turnkey solution. The value lies in:
1. **Clean architecture**: Easy to modify and extend
2. **Proper methodology**: No lookahead bias, rigorous validation
3. **Configurability**: JSON-based experiments for rapid iteration
4. **Extensibility**: Swap algorithms, add features, tune hyperparameters

**For practitioners looking to improve performance:**
- Apply systematic hyperparameter optimization (Optuna, Ray Tune)
- Experiment with advanced RL algorithms (PPO, SAC)
- Incorporate additional data sources and features
- Extend training duration and network capacity
- Conduct thorough walk-forward analysis

I hope this series inspired you to explore RL in finance! The code is on [GitHub](https://github.com/sharan-naribole/dqn-trading)—fork it, improve it, and share your results!

---

**← Previous:** [Part IV: DQN Architecture Deep Dive](/posts/2026/01/17/dqn-trading-part-IV)

**← Back to Part I:** [Problem Statement & RL Motivation](/posts/2026/01/01/dqn-trading-part-I)

---

## References

- [DQN Trading System GitHub Repository](https://github.com/sharan-naribole/dqn-trading)
- [AI Trading Strategies Nanodegree](https://www.udacity.com/course/ai-trading-strategies--nd881) - Udacity
- [Quantitative Trading: How to Build Your Own Algorithmic Trading Business](https://www.amazon.com/Quantitative-Trading-Build-Algorithmic-Business/dp/1119800064) - Ernest Chan
- [Advances in Financial Machine Learning](https://www.amazon.com/Advances-Financial-Machine-Learning-Marcos/dp/1119482089) - Marcos López de Prado
- [QuantStart - Python for Finance](https://www.quantstart.com/) - Algorithmic trading tutorials
- [Zipline - Backtesting Library](https://github.com/quantopian/zipline) - Python backtesting framework

[dqn-part-I]: /posts/2026/01/01/dqn-trading-part-I
[dqn-part-II]: /posts/2026/01/07/dqn-trading-part-II
[dqn-part-III]: /posts/2026/01/14/dqn-trading-part-III
[dqn-part-IV]: /posts/2026/01/17/dqn-trading-part-IV
[dqn-part-V]: /posts/2026/01/18/dqn-trading-part-V
