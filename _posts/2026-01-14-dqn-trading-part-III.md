---
title: 'Deep Q-Network for Stock Trading (Part III): Learning Environment Design'
date: 2026-01-14 12:00:00 -0800
permalink: /posts/2026/01/14/dqn-trading-part-III
tags:
  - reinforcement learning
  - trading environment
  - markov decision process
  - reward engineering
redirect_from:
  - /2026/01/14/dqn-trading-part-III.html
toc: true
---

*This post is part of a series on building a Deep Q-Network (DQN) based trading system for SPY (S&P 500 ETF).*

- [Part I: Problem Statement & RL Motivation][dqn-part-I]
- [Part II: Data Engineering Pipeline][dqn-part-II]
- [Part III: Learning Environment Design][dqn-part-III]
- [Part IV: DQN Architecture Deep Dive][dqn-part-IV]
- [Part V: Software Architecture & Results][dqn-part-V]

---

**← Previous:** [Part II: Data Engineering Pipeline](/posts/2026/01/07/dqn-trading-part-II)

**Next Post →** [Part IV: DQN Architecture Deep Dive](/posts/2026/01/17/dqn-trading-part-IV)

---

## ⚠️ Disclaimer

**This blog series is for educational and research purposes only.** The content should not be considered financial advice, investment advice, or trading advice. Trading stocks and financial instruments involves substantial risk of loss and is not suitable for every investor. Past performance does not guarantee future results. Always consult with a qualified financial advisor before making investment decisions.

---

## Introduction

In [Part II](/posts/2026/01/07/dqn-trading-part-II), we built the data pipeline with 25 technical indicators. Now comes the exciting part: designing the **learning environment** where our DQN agent will interact with the market.

A well-designed environment is critical for successful RL. We need to formulate trading as a **Markov Decision Process (MDP)** with:
- **State Space**: What does the agent observe?
- **Action Space**: What can the agent do?
- **Reward Function**: How do we measure success?
- **Transition Dynamics**: How does the environment evolve?

This part covers our sophisticated environment featuring:
- Multi-buy position accumulation
- Partial sells with FIFO lot tracking
- Action masking for safety
- Risk management guardrails

The complete code is in `src/trading/environment.py` on [GitHub](https://github.com/sharan-naribole/dqn-trading).

## State Space: What Does the Agent See?

### State Composition

The agent observes a **window** of recent market data plus its current portfolio state:

```
State = [Market Features (window_size × n_features) + Portfolio State]
```

**Example with window_size=5, 25 features:**
```
Shape: (5, 27)  # 25 market features + 2 portfolio features

Market Features (past 5 days):
- SPY_Close, SPY_Return, SPY_High_Low_Pct, ...
- BB_Position, BB_Width, ...
- EMA_8, EMA_21, SMA_50, SMA_200, ...
- RSI, ADX, DI_Plus, DI_Minus
- Volume_Ratio, Volume_Trend
- VIX_Close

Portfolio State (current):
- Position Ratio: shares_held / max_affordable_shares  (dynamically normalized)
- Cash Ratio: balance / starting_balance   (0.0 to ~2.0+)
```

### Implementation

```python
def _get_state(self) -> np.ndarray:
    """
    Get current state observation.

    Returns:
        np.ndarray: State array of shape (window_size, n_features + 2)
    """
    # Extract window of market features
    start_idx = self.current_step - self.window_size
    end_idx = self.current_step

    market_state = self.data[self.feature_columns].iloc[start_idx:end_idx].values

    # Add portfolio state to each time step
    # Normalize position by max affordable shares (price-adaptive)
    current_price = self._get_current_price()
    max_affordable_shares = self.starting_balance / current_price if current_price > 0 else 1

    position_ratio = self.shares_held / max_affordable_shares
    balance_ratio = self.balance / self.starting_balance

    portfolio_state = np.array([position_ratio, balance_ratio])

    # Broadcast portfolio state across time window
    portfolio_broadcast = np.tile(portfolio_state, (self.window_size, 1))

    # Concatenate: (window_size, n_market_features + 2)
    state = np.concatenate([market_state, portfolio_broadcast], axis=1)

    return state.astype(np.float32)
```

### Why This Design?

**1. Temporal Context (window_size)**

Using a lookback window (e.g., 5 days) allows the agent to:
- Detect short-term trends (price rising/falling over past few days)
- Identify patterns (e.g., Bollinger Band squeeze followed by breakout)
- Learn temporal dependencies

**2. Portfolio Awareness**

The agent needs to know its current state:
- **Position Ratio**: "Am I fully invested, partially invested, or in cash?"
  - `0.0`: No position (can only buy)
  - `0.5`: Half-invested (can buy more or start selling)
  - `~1.0`: Fully invested (used all starting capital at current price)

- **Cash Ratio**: "How much buying power do I have?"
  - `< 1.0`: Lost money (unrealized or realized losses)
  - `= 1.0`: Break-even
  - `> 1.0`: Profitable (gains available for reinvestment)

**3. Normalization**

All features are normalized (see Part II), ensuring:
- Stable neural network training
- Features contribute equally regardless of scale
- Generalization across different price regimes

## Action Space: Multi-Buy and Partial Sells

### The Traditional Approach (Naive)

Simple implementations often use:
```
Action 0: HOLD
Action 1: BUY (fixed quantity)
Action 2: SELL (sell entire position)
```

**Limitations:**
- Can't scale into positions gradually
- Can't take partial profits
- All-or-nothing exits (no risk management flexibility)

### Our Sophisticated Approach

We use **share_increments** and **enable_buy_max** to define flexible position sizing:

```python
# Configuration
share_increments = [10, 50, 100, 200]
enable_buy_max = True  # Enable BUY_MAX action

# Resulting Action Space (11 actions with BUY_MAX enabled):
Action 0: HOLD
Action 1: BUY 10 shares
Action 2: BUY 50 shares
Action 3: BUY 100 shares
Action 4: BUY 200 shares
Action 5: BUY_MAX (buy as many shares as balance allows)
Action 6: SELL 10 shares
Action 7: SELL 50 shares
Action 8: SELL 100 shares
Action 9: SELL 200 shares
Action 10: SELL ALL (regardless of shares held)
```

**Formula (with BUY_MAX enabled):**
```
n_actions = 1 + N + 1 + N + 1
where N = len(share_increments)

= 1 (HOLD) + N (BUY actions) + 1 (BUY_MAX) + N (SELL actions) + 1 (SELL_ALL)

Example: [10, 50, 100, 200] → 1 + 4 + 1 + 4 + 1 = 11 actions
```

**Formula (with BUY_MAX disabled):**
```
n_actions = 1 + N + N + 1
= 1 (HOLD) + N (BUY actions) + N (SELL actions) + 1 (SELL_ALL)

Example: [10, 50, 100, 200] → 1 + 4 + 4 + 1 = 10 actions
```

**Why BUY_MAX?**

The BUY_MAX action enables price-adaptive position sizing:
- **Dynamic allocation**: Buys `int(balance / current_price)` shares
- **No arbitrary limits**: Agent can go "all-in" when confident
- **Price-adaptive**: At $400 vs $500, BUY_MAX buys different quantities
- **Symmetric to SELL_ALL**: Intuitive action pairing

### ActionMasker Implementation

The `ActionMasker` class handles action space creation and validation:

```python
class ActionMasker:
    """Manages action space with share increments and BUY_MAX."""

    def __init__(self, share_increments: List[int], enable_buy_max: bool = True):
        """
        Args:
            share_increments: List of share quantities (e.g., [10, 50, 100, 200])
            enable_buy_max: Whether to enable BUY_MAX action (default: True)
        """
        self.share_increments = sorted(share_increments)
        self.enable_buy_max = enable_buy_max

        n_increments = len(share_increments)

        # Calculate action space size
        if enable_buy_max:
            self.n_actions = 1 + n_increments + 1 + n_increments + 1
        else:
            self.n_actions = 1 + n_increments + n_increments + 1

        # Define action indices
        self.HOLD = 0
        self.BUY_ACTIONS = list(range(1, 1 + n_increments))

        if enable_buy_max:
            self.BUY_MAX = 1 + n_increments
            self.SELL_ACTIONS = list(range(self.BUY_MAX + 1, self.BUY_MAX + 1 + n_increments))
            self.SELL_ALL = self.BUY_MAX + 1 + n_increments
        else:
            self.BUY_MAX = None
            self.SELL_ACTIONS = list(range(1 + n_increments, 1 + 2*n_increments))
            self.SELL_ALL = 1 + 2*n_increments
        self.SELL_ALL = 1 + 2*n_increments

    def action_to_shares(self, action: int) -> int:
        """Convert action to number of shares."""
        if action == self.HOLD:
            return 0
        elif action in self.BUY_ACTIONS:
            idx = action - 1
            return self.share_increments[idx]
        elif action in self.SELL_ACTIONS:
            idx = action - (1 + len(self.share_increments))
            return -self.share_increments[idx]  # Negative for sell
        else:  # SELL_ALL
            return 0  # Handled separately

    def get_action_mask(
        self,
        current_position: int,
        current_balance: float,
        current_price: float
    ) -> np.ndarray:
        """
        Get mask of valid actions (1 = valid, 0 = invalid).

        Args:
            current_position: Shares currently held
            current_balance: Cash balance
            current_price: Current stock price

        Returns:
            Boolean array of shape (n_actions,)
        """
        mask = np.zeros(self.n_actions, dtype=bool)

        # HOLD always valid
        mask[self.HOLD] = True

        # BUY actions valid if have sufficient balance
        for action in self.BUY_ACTIONS:
            shares = self.action_to_shares(action)
            cost = shares * current_price
            if current_balance >= cost:
                mask[action] = True

        # BUY_MAX valid if can afford at least 1 share
        if self.enable_buy_max and current_balance >= current_price:
            mask[self.BUY_MAX] = True

        # SELL actions valid if:
        # Have enough shares to sell
        for action in self.SELL_ACTIONS:
            shares = abs(self.action_to_shares(action))
            if current_position >= shares:
                mask[action] = True

        # SELL_ALL valid if holding any shares
        if current_position > 0:
            mask[self.SELL_ALL] = True

        return mask
```

### Action Masking: Preventing Invalid Moves

**Why Action Masking?**

Without masking, the agent might:
- Attempt to sell 50 shares when holding only 10
- Try to buy when balance is insufficient
- Execute BUY_MAX with zero balance

**Why Necessary During Training?**

During training, the agent uses **epsilon-greedy exploration**, taking random actions with probability ε (e.g., 20%). Without action masking:

1. **Wasted Exploration**: Random exploration would frequently attempt invalid actions, providing no learning signal
2. **Environment Errors**: Invalid actions could crash the environment or return meaningless rewards
3. **Slower Convergence**: The agent would waste valuable episodes learning basic constraints instead of learning the actual trading task

Action masking **constrains the search space to valid actions**, allowing the agent to focus exploration on the interesting part of the problem.

**Implementation During Training:**
```python
# Get valid action mask
action_mask = env.get_action_mask()

# Agent outputs Q-values for all actions
q_values = agent.predict(state)

# Mask invalid actions with large negative value
masked_q_values = np.where(action_mask, q_values, -np.inf)

# Select best valid action
action = np.argmax(masked_q_values)
```

## Multi-Buy Position Accumulation

### The Trading Scenario

Imagine this sequence:
1. **Day 1**: Buy 10 shares @ $450 (entry price = $450)
2. **Day 5**: Buy 50 shares @ $460 (market went up)
3. **Day 10**: Buy 10 shares @ $455 (small dip)

**Question**: What's the agent's average entry price?

**Answer**: Weighted average!

```
Total Cost = (10 × $450) + (50 × $460) + (10 × $455)
           = $4,500 + $23,000 + $4,550
           = $32,050

Total Shares = 10 + 50 + 10 = 70

Weighted Avg Entry = $32,050 / 70 = $457.86
```

### Lot Tracking Implementation

```python
def _add_lot(self, shares: int, price: float):
    """
    Add a new lot to position.

    Args:
        shares: Number of shares bought
        price: Purchase price
    """
    self.lots.append({
        'shares': shares,
        'entry_price': price
    })

    self.shares_held += shares
    self.entry_price = self._calculate_weighted_avg_entry()

def _calculate_weighted_avg_entry(self) -> Optional[float]:
    """Calculate weighted average entry price from all lots."""
    if not self.lots or self.shares_held == 0:
        return None

    total_cost = sum(lot['shares'] * lot['entry_price'] for lot in self.lots)
    return total_cost / self.shares_held
```

**After 3 buys, lots look like:**
```python
self.lots = [
    {'shares': 10, 'entry_price': 450.0},
    {'shares': 50, 'entry_price': 460.0},
    {'shares': 10, 'entry_price': 455.0}
]
self.shares_held = 70
self.entry_price = 457.86  # Weighted average
```

## Partial Sells with FIFO Tracking

### Why FIFO (First-In-First-Out)?

When selling portions of a position, we need to track which lots are sold for accurate profit calculation. FIFO matches tax accounting rules and provides consistent profit attribution.

### FIFO Example

**Position:**
```
Lot 1: 10 shares @ $450
Lot 2: 50 shares @ $460
Lot 3: 10 shares @ $455
```

**Agent decides: SELL 55 shares @ $470**

**FIFO Logic:**
1. Sell entire Lot 1: 10 shares @ $450 → Profit = 10 × ($470 - $450) = $200
2. Sell 45 shares from Lot 2: 45 shares @ $460 → Profit = 45 × ($470 - $460) = $450
3. Remaining Lot 2: 5 shares @ $460 (stays in position)
4. Lot 3 unchanged: 10 shares @ $455

**Total Profit**: $200 + $450 = $650

**Remaining Position**:
```
Lot 2 (partial): 5 shares @ $460
Lot 3: 10 shares @ $455
New weighted avg entry = (5×460 + 10×455) / 15 = $456.67
```

### Implementation

```python
def _remove_shares_fifo(
    self,
    shares_to_sell: int,
    sell_price: float
) -> Tuple[float, float]:
    """
    Remove shares using FIFO and calculate realized profit.

    Args:
        shares_to_sell: Number of shares to sell
        sell_price: Current market price

    Returns:
        Tuple of (realized_profit, avg_entry_sold)
    """
    remaining_to_sell = shares_to_sell
    total_cost_basis = 0
    shares_sold = 0

    while remaining_to_sell > 0 and self.lots:
        lot = self.lots[0]  # Always take from first lot (FIFO)

        if lot['shares'] <= remaining_to_sell:
            # Sell entire lot
            shares_from_lot = lot['shares']
            total_cost_basis += shares_from_lot * lot['entry_price']
            shares_sold += shares_from_lot
            remaining_to_sell -= shares_from_lot
            self.lots.pop(0)  # Remove depleted lot
        else:
            # Partial lot sale
            shares_from_lot = remaining_to_sell
            total_cost_basis += shares_from_lot * lot['entry_price']
            shares_sold += shares_from_lot
            lot['shares'] -= shares_from_lot  # Reduce lot size
            remaining_to_sell = 0

    # Calculate metrics
    avg_entry_sold = total_cost_basis / shares_sold if shares_sold > 0 else 0
    realized_profit = (sell_price - avg_entry_sold) * shares_sold

    # Update state
    self.shares_held -= shares_sold
    self.entry_price = self._calculate_weighted_avg_entry()

    return realized_profit, avg_entry_sold
```

### SELL_ALL Action

The `SELL_ALL` action is special—it always sells 100% of the position:

```python
if action == self.action_masker.SELL_ALL:
    shares_to_sell = self.shares_held  # Sell everything
```

This is useful when:
- Stop-loss triggers (get out completely)
- Market conditions change drastically
- Agent wants to reset to cash

## Reward Function: Incentivizing Profitability

### Design Principles

A good reward function should:
1. **Incentivize profitable trades** (positive reward for gains)
2. **Penalize losses** (negative reward for losing trades)
3. **Discourage excessive trading** (transaction costs)
4. **Encourage action** (small penalty for inactivity)

### Our Reward Structure

**1. HOLD Action**
```python
if action == HOLD:
    reward = idle_reward  # Default: -0.001
```

Small negative reward encourages the agent to trade when opportunities arise, rather than staying idle.

**2. BUY Action**
```python
if is_buy_action(action):
    shares_bought = action_to_shares(action)

    # Buy reward component (default: 0.0)
    buy_reward = buy_reward_per_share * shares_bought

    # Transaction cost penalty
    transaction_penalty = -buy_transaction_cost_per_share * shares_bought

    reward = buy_reward + transaction_penalty
```

**Buy Transaction Cost**: Represents:
- Broker commissions
- Bid-ask spread slippage
- Market impact

Typical value: `$0.01` per share

**3. SELL Action (The Main Reward)**
```python
if is_sell_action(action):
    # Execute FIFO sell
    realized_profit, avg_entry_sold = _remove_shares_fifo(shares_to_sell, sell_price)

    # Net profit after ALL transaction costs
    buy_cost = buy_transaction_cost_per_share * shares_to_sell
    sell_cost = sell_transaction_cost_per_share * shares_to_sell
    total_transaction_cost = buy_cost + sell_cost

    net_profit = realized_profit - total_transaction_cost

    # Reward using log return (time-additive, symmetric, outlier-resistant)
    position_value_sold = avg_entry_sold * shares_to_sell
    simple_return = net_profit / position_value_sold
    reward = np.log(1 + simple_return) * 100
```

**Why Log Returns Instead of Simple Percentage?**

We use **logarithmic returns** instead of simple percentage returns for several critical reasons:

1. **Time-Additivity (Compound Growth)**:
   - Simple: 10% + 10% = 20% ❌ (incorrect for compounding)
   - Log: log(1.1) + log(1.1) = log(1.21) ≈ 19.06% ✓ (correct!)

2. **Symmetry of Gains/Losses**:
   - Simple: -50% loss requires +100% gain to break even (asymmetric)
   - Log: -69.3 vs +69.3 (more balanced)

3. **Outlier Compression**:
   - Reduces extreme reward spikes that destabilize RL training
   - 200% gain isn't treated as 4x more valuable than 50% gain

4. **Better Statistical Properties**:
   - Closer to normal distribution → better neural network gradients
   - Standard practice in quantitative finance

**Comparison Table:**

| Trade Return | Simple % | Log Reward | Difference |
|--------------|----------|------------|------------|
| +10% | +10.00 | +9.53 | -4.7% |
| +50% | +50.00 | +40.55 | -18.9% |
| +100% | +100.00 | +69.31 | -30.7% |
| -50% | -50.00 | -69.31 | +38.6% |

### Reward Example

**Scenario:**
- Buy 50 shares @ $450 (weighted avg entry)
- Sell 50 shares @ $460
- Transaction costs: $0.01 per share (buy) + $0.01 (sell)

**Calculation:**
```python
realized_profit = 50 × ($460 - $450) = $500

total_transaction_cost = 50 × ($0.01 + $0.01) = $1.00

net_profit = $500 - $1.00 = $499

position_value_sold = 50 × $450 = $22,500

simple_return = $499 / $22,500 = 0.0222 (2.22%)

reward = log(1 + 0.0222) × 100 = log(1.0222) × 100 ≈ 2.19
```

**Interpretation**: 2.19 log return units (≈2.22% simple return) on capital deployed

## Risk Management Guardrails

### Stop-Loss and Take-Profit

While we want the agent to learn optimal exit strategies, we also implement **guardrails** to prevent catastrophic losses and stabilize training.

**Why Necessary During Training?**

Early in training, the agent's policy is essentially random. Without stop-loss/take-profit:

1. **Catastrophic Losses**: Random exploration could hold losing positions indefinitely, accumulating -50%, -80% unrealized losses
2. **Noisy Learning Signals**: Extreme losses create unstable, unhelpful gradients that hinder learning
3. **Reward Space Instability**: A few disastrous episodes can dominate the replay buffer, biasing the agent away from taking any positions

**With guardrails**, even during random exploration:
- Maximum loss per position is bounded (e.g., -20% stop-loss)
- Maximum gain is capped (e.g., +20% take-profit, preventing greed during lucky streaks)
- The agent learns in a **stable, bounded reward space**
- Learning signals are cleaner and more actionable

Think of it like training wheels on a bicycle—they prevent catastrophic failures during the learning phase, allowing the agent to focus on learning optimal entry/exit timing rather than basic risk management.

```python
class TradingGuardrails:
    """Risk management rules."""

    def __init__(
        self,
        stop_loss_pct: float = 20,
        take_profit_pct: float = 20
    ):
        """
        Args:
            stop_loss_pct: Sell all if position down X% from weighted avg entry
            take_profit_pct: Sell all if position up X% from weighted avg entry
        """
        self.stop_loss_pct = stop_loss_pct
        self.take_profit_pct = take_profit_pct

    def check_guardrails(
        self,
        current_price: float,
        entry_price: Optional[float],
        shares_held: int
    ) -> Optional[str]:
        """
        Check if guardrails triggered.

        Returns:
            'stop_loss', 'take_profit', or None
        """
        if shares_held == 0 or entry_price is None:
            return None

        # Calculate position P&L percentage
        pnl_pct = ((current_price - entry_price) / entry_price) * 100

        if pnl_pct <= -self.stop_loss_pct:
            return 'stop_loss'
        elif pnl_pct >= self.take_profit_pct:
            return 'take_profit'

        return None
```

### Guardrail Trigger Example

**Position:**
- Holding 70 shares
- Weighted avg entry: $457.86
- Stop-loss: 20%
- Take-profit: 20%

**Trigger Thresholds:**
```python
stop_loss_price = $457.86 × (1 - 0.20) = $366.29
take_profit_price = $457.86 × (1.20) = $549.43
```

**If current price drops to $365**:
```
PnL% = ($365 - $457.86) / $457.86 × 100 = -20.3%
→ Stop-loss TRIGGERED → Force SELL_ALL
```

### Disabling Guardrails

For experiments where we want pure RL control:
```json
{
  "trading": {
    "stop_loss_pct": 100,     // Effectively disabled
    "take_profit_pct": 10000  // Effectively disabled
  }
}
```

This allows comparing:
- **Baseline**: 20% stop-loss/take-profit
- **Aggressive**: 10% stop-loss/take-profit
- **Pure DQN**: No guardrails (100%/10000%)

## The Complete MDP

### Putting It All Together

Our trading environment implements this MDP:

**State Space**:
- Market features: (window_size, 25 features)
- Portfolio state: (position_ratio, cash_ratio)

**Action Space** (10 actions with share_increments=[10, 50, 100, 200], enable_buy_max=false):
```
{HOLD, BUY_10, BUY_50, BUY_100, BUY_200, SELL_10, SELL_50, SELL_100, SELL_200, SELL_ALL}
```

**Reward Function**:
- HOLD: -0.001 (idle penalty)
- BUY: -transaction_cost
- SELL: log(1 + net_profit / position_value) × 100 (log return)

**Transition Dynamics**:
- Deterministic price progression (historical data)
- Stochastic agent behavior (epsilon-greedy exploration)

**Termination**:
- Episode ends when reaching end of data

### Environment Step

```python
def step(self, action: int) -> Tuple[np.ndarray, float, bool, Dict]:
    """
    Execute one timestep.

    Args:
        action: Action index to execute

    Returns:
        Tuple of (next_state, reward, done, info)
    """
    current_price = self._get_current_price()

    # Check guardrails (stop-loss / take-profit)
    guardrail_trigger = self.guardrails.check_guardrails(
        current_price, self.entry_price, self.shares_held
    )

    if guardrail_trigger:
        # Force exit entire position
        action = self.action_masker.SELL_ALL

    # Execute action and get reward
    reward = self._execute_action(action, current_price)

    # Advance time
    self.current_step += 1

    # Get next state
    next_state = self._get_state()
    done = self.current_step >= len(self.data)

    info = self._get_info()

    return next_state, reward, done, info
```

## Key Takeaways

1. **State Design**: Combines market features (technical indicators) with portfolio state (position, cash)

2. **Action Space**: Flexible position sizing with share_increments and BUY_MAX enables gradual accumulation, full allocation, and partial exits

3. **Action Masking**: Prevents wasting training on impossible actions (insufficient balance, invalid sells)

4. **Multi-Buy**: Weighted average entry price calculated from multiple buy lots

5. **FIFO Sells**: First-In-First-Out lot tracking for accurate profit calculation

6. **Reward Engineering**: Percentage-based returns incentivize risk-adjusted profitability

7. **Guardrails**: Stop-loss and take-profit provide safety net while agent learns

## What's Next?

In **Part IV**, we'll dive into the **DQN Architecture**:
- Double DQN to prevent overestimation
- Dueling architecture (value & advantage streams)
- Experience replay buffer
- Target network stabilization
- Network layers and hyperparameters

The environment is ready—now let's build the brain!

---

**← Previous:** [Part II: Data Engineering Pipeline](/posts/2026/01/07/dqn-trading-part-II)

**Next Post →** [Part IV: DQN Architecture Deep Dive](/posts/2026/01/17/dqn-trading-part-IV)

---

## References

- [Human-level control through deep reinforcement learning](https://www.nature.com/articles/nature14236) - DeepMind DQN paper
- [Markov Decision Processes](http://incompleteideas.net/book/ebook/node31.html) - Sutton & Barto Chapter 3
- [Reward Shaping for Reinforcement Learning](https://people.eecs.berkeley.edu/~pabbeel/cs287-fa09/readings/NgHaradaRussell-shaping-ICML1999.pdf)
- [FIFO Accounting Method](https://www.investopedia.com/terms/f/fifo.asp) - Investopedia
- [Action Masking in RL](https://arxiv.org/abs/2006.14171) - Invalid Action Masking paper

[dqn-part-I]: /posts/2026/01/01/dqn-trading-part-I
[dqn-part-II]: /posts/2026/01/07/dqn-trading-part-II
[dqn-part-III]: /posts/2026/01/14/dqn-trading-part-III
[dqn-part-IV]: /posts/2026/01/17/dqn-trading-part-IV
[dqn-part-V]: /posts/2026/01/18/dqn-trading-part-V
