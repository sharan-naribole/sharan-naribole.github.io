---
title: 'Deep Q-Network for Stock Trading (Part II): Data Engineering Pipeline'
date: 2026-01-07 12:00:00 -0800
permalink: /posts/2026/01/07/dqn-trading-part-II
tags:
  - reinforcement learning
  - data engineering
  - feature engineering
  - algorithmic trading
redirect_from:
  - /2026/01/07/dqn-trading-part-II.html
toc: true
---

*This post is part of a series on building a Deep Q-Network (DQN) based trading system for SPY (S&P 500 ETF).*

- [Part I: Problem Statement & RL Motivation][dqn-part-I]
- [Part II: Data Engineering Pipeline][dqn-part-II]
- [Part III: Learning Environment Design][dqn-part-III]
- [Part IV: DQN Architecture Deep Dive][dqn-part-IV]
- [Part V: Software Architecture & Results][dqn-part-V]

---

**← Previous:** [Part I: Problem Statement & RL Motivation](/posts/2026/01/01/dqn-trading-part-I)

**Next Post →** [Part III: Learning Environment Design](/posts/2026/01/14/dqn-trading-part-III)

---

## ⚠️ Disclaimer

**This blog series is for educational and research purposes only.** The content should not be considered financial advice, investment advice, or trading advice. Trading stocks and financial instruments involves substantial risk of loss and is not suitable for every investor. Past performance does not guarantee future results. Always consult with a qualified financial advisor before making investment decisions.

---

## Introduction

In [Part I](/posts/2026/01/01/dqn-trading-part-I), we established why Reinforcement Learning and DQN make sense for trading. Now, we dive into the **data engineering pipeline**—the foundation that ensures our agent learns from clean, properly prepared data.

Bad data leads to bad models. In this part, we'll cover:
1. **Data Collection** with intelligent caching
2. **Feature Engineering** using 25+ technical indicators
3. **Data Splitting** strategy to prevent lookahead bias
4. **Rolling Normalization** for stationarity

The complete code is in the `src/data/` and `src/features/` modules on [GitHub](https://github.com/sharan-naribole/dqn-trading).

## Data Collection: Smart Caching with yfinance

### The Challenge

Training a DQN agent requires thousands of episodes over historical data. Naively downloading data from Yahoo Finance for every run would be:
- **Slow**: API calls take 1-2 seconds each
- **Wasteful**: Downloading the same data repeatedly
- **Risky**: May hit rate limits

### Solution: Intelligent Caching

The `DataCollector` class implements smart caching:

```python
class DataCollector:
    """Collect market data from Yahoo Finance with intelligent caching."""

    def __init__(self, config: Dict):
        self.ticker = config['ticker']  # e.g., "SPY"
        self.start_date = pd.to_datetime(config['start_date'])
        self.end_date = pd.to_datetime(config['end_date'])
        self.data_dir = config.get('output', {}).get('data_dir', 'data')

        # Buffer for technical indicators (need historical context)
        self.buffer_days = 250  # ~1 year of trading days
```

**Key Design Decision: Buffer Days**

Technical indicators like 200-day SMA require historical data. If we request data from `2023-01-01` but calculate a 200-day SMA, we need data back to ~`2022-04-26`. The `buffer_days` automatically handles this.

### Caching Strategy

```python
def _get_ticker_data(self, ticker: str, force_download: bool) -> pd.DataFrame:
    """Get data with caching logic."""

    # Calculate actual range needed (with buffer)
    start_with_buffer = self.start_date - timedelta(days=self.buffer_days)

    if not force_download:
        # Try to find cached file that contains our date range
        cached_file = self._find_cached_file(ticker, start_with_buffer, self.end_date)

        if cached_file:
            print(f"Using cached data from {cached_file}")
            data = pd.read_csv(cached_file, index_col='Date', parse_dates=True)
            return data

    # Download new data
    print(f"Downloading {ticker} data from {start_with_buffer} to {self.end_date}")
    data = yf.download(ticker, start=start_with_buffer, end=self.end_date)

    # Save with date range in filename for easy identification
    filename = f"{ticker}_{start_with_buffer.strftime('%Y%m%d')}_{self.end_date.strftime('%Y%m%d')}.csv"
    filepath = os.path.join(self.data_dir, filename)
    data.to_csv(filepath)

    return data
```

**File Naming Convention:**
```
data/
├── SPY_20220426_20251231.csv
└── VIX_20220426_20251231.csv
```

The date range in the filename allows quick identification of cached data coverage.

### VIX Data

We also collect **VIX (Volatility Index)** data:

```python
def collect_data(self) -> Tuple[pd.DataFrame, pd.DataFrame]:
    """Collect ticker and VIX data."""
    ticker_data = self._get_ticker_data(self.ticker, force_download=False)
    vix_data = self._get_ticker_data("^VIX", force_download=False)
    return ticker_data, vix_data
```

**Why VIX?**

VIX measures market fear/uncertainty. Including VIX helps the agent understand:
- High VIX (>30): Market stress, increase risk management
- Low VIX (<15): Market calm, potentially more aggressive
- VIX spikes: Often precede market sell-offs

## Feature Engineering: 25+ Technical Indicators

### Feature Categories

The `FeatureEngineer` class creates 4 categories of features:

1. **Price Features** (7 features)
2. **Technical Indicators** (14 features)
3. **Volume Features** (3 features)
4. **VIX Features** (1 feature)

### 1. Price Features

```python
def _add_price_features(self, data: pd.DataFrame) -> pd.DataFrame:
    """Add price-based features."""
    close_col = f'{self.ticker}_Close'
    high_col = f'{self.ticker}_High'
    low_col = f'{self.ticker}_Low'

    # Price changes (multiple timeframes)
    data['Price_Change'] = data[close_col].pct_change()
    data['Price_Change_5d'] = data[close_col].pct_change(5)
    data['Price_Change_20d'] = data[close_col].pct_change(20)

    # Normalized intraday range
    data['HL_Spread'] = (data[high_col] - data[low_col]) / data[close_col]

    # Close position within daily range (derived feature)
    data['Close_Position'] = (data[close_col] - data[low_col]) / \
                              (data[high_col] - data[low_col] + 1e-10)

    # Rolling volatility
    data['Volatility_5d'] = data['Price_Change'].rolling(5).std()
    data['Volatility_20d'] = data['Price_Change'].rolling(20).std()

    return data
```

**Price Features Explained:**

- **Price_Change (1d, 5d, 20d)**: Multi-timeframe momentum
- **HL_Spread**: Intraday volatility (normalized by price)
- **Close_Position**: Where close sits in daily range (0=at low, 1=at high)
  - Derived feature similar to BB_Position but for daily range
- **Volatility (5d, 20d)**: Recent price stability/instability

### 2. Technical Indicators

#### Bollinger Bands

```python
def _add_bollinger_bands(self, data: pd.DataFrame) -> pd.DataFrame:
    """Add Bollinger Bands using TA library."""
    period = self.indicators_config['bollinger_period']  # 20
    std = self.indicators_config['bollinger_std']  # 2

    indicator = ta.volatility.BollingerBands(
        close=data[f'{self.ticker}_Close'],
        window=period,
        window_dev=std
    )

    # Calculate raw bands
    data['BB_High'] = indicator.bollinger_hband()
    data['BB_Low'] = indicator.bollinger_lband()
    data['BB_Middle'] = indicator.bollinger_mavg()

    # Derived features (these are used in the model, not raw bands)
    data['BB_Width'] = data['BB_High'] - data['BB_Low']
    data['BB_Position'] = (
        (data[f'{self.ticker}_Close'] - data['BB_Low']) /
        (data['BB_Width'] + 1e-10)  # Avoid division by zero
    )

    return data
```

**Why Derived Features?**

Instead of using raw band values (BB_High, BB_Low), we use two derived features:

**BB_Position** (Normalized position in bands):
- Range: 0.0 to 1.0
- `0.0`: Price at lower band (oversold)
- `0.5`: Price at middle band (neutral)
- `1.0`: Price at upper band (overbought)
- Scale-invariant: Works regardless of stock price level
- Known as **%B indicator** in technical analysis

**BB_Width** (Volatility measure):
- Absolute width between bands
- High value = High volatility (wide bands)
- Low value = Low volatility (bands squeezed)
- Captures Bollinger Band "squeeze" patterns directly

#### Moving Averages

```python
def _add_moving_averages(self, data: pd.DataFrame) -> pd.DataFrame:
    """Add EMA and SMA indicators."""
    close_col = f'{self.ticker}_Close'

    # Exponential Moving Averages (faster reaction)
    data['EMA_8'] = ta.trend.ema_indicator(data[close_col], window=8)
    data['EMA_21'] = ta.trend.ema_indicator(data[close_col], window=21)

    # Simple Moving Averages (smoother)
    data['SMA_50'] = ta.trend.sma_indicator(data[close_col], window=50)
    data['SMA_200'] = ta.trend.sma_indicator(data[close_col], window=200)

    # Binary signal features
    data['EMA_Signal'] = (data['EMA_8'] > data['EMA_21']).astype(int)
    data['Price_Above_SMA50'] = (data[close_col] > data['SMA_50']).astype(int)
    data['Price_Above_SMA200'] = (data[close_col] > data['SMA_200']).astype(int)

    return data
```

**Trading Signals (Binary Indicators):**

The model uses binary (0/1) indicators rather than continuous differences:
- **EMA_Signal**: 1 if EMA_8 > EMA_21 (short-term bullish), 0 otherwise
- **Price_Above_SMA50**: 1 if price above 50-day SMA (intermediate trend)
- **Price_Above_SMA200**: 1 if price above 200-day SMA (long-term trend)

This approach simplifies the signal for the neural network while capturing trend direction.

#### RSI (Relative Strength Index)

```python
def _add_rsi(self, data: pd.DataFrame) -> pd.DataFrame:
    """Add RSI indicator."""
    period = self.indicators_config['rsi_period']  # 14

    data['RSI'] = ta.momentum.RSIIndicator(
        close=data[f'{self.ticker}_Close'],
        window=period
    ).rsi()

    # Binary zone indicators
    data['RSI_Oversold'] = (data['RSI'] < 30).astype(int)
    data['RSI_Overbought'] = (data['RSI'] > 70).astype(int)

    return data
```

**RSI Features:**
- **RSI** (continuous 0-100): Raw momentum strength
- **RSI_Oversold** (binary): 1 if RSI < 30 (potential buy zone)
- **RSI_Overbought** (binary): 1 if RSI > 70 (potential sell zone)

The binary indicators help the model recognize extreme conditions explicitly.

#### ADX (Average Directional Index)

```python
def _add_adx(self, data: pd.DataFrame) -> pd.DataFrame:
    """Add ADX for trend strength."""
    period = self.indicators_config['adx_period']  # 14

    adx = ta.trend.ADXIndicator(
        high=data[f'{self.ticker}_High'],
        low=data[f'{self.ticker}_Low'],
        close=data[f'{self.ticker}_Close'],
        window=period
    )

    data['ADX'] = adx.adx()
    data['ADX_Pos'] = adx.adx_pos()   # Directional components (created but not used)
    data['ADX_Neg'] = adx.adx_neg()
    data['ADX_Strong'] = (data['ADX'] > 25).astype(int)  # Binary trend strength

    return data
```

**ADX Features:**
- **ADX** (continuous 0-100): Trend strength magnitude
- **ADX_Strong** (binary): 1 if ADX > 25 (strong trend present)

ADX > 25 indicates trending market conditions, where momentum strategies may be more effective than mean reversion.

### 3. Volume Features

```python
def _add_volume_features(self, data: pd.DataFrame) -> pd.DataFrame:
    """Add volume-based features."""
    close_col = f'{self.ticker}_Close'
    volume_col = f'{self.ticker}_Volume'

    # Volume moving averages
    data['Volume_MA_10'] = data[volume_col].rolling(10).mean()
    data['Volume_MA_20'] = data[volume_col].rolling(20).mean()

    # Relative volume (derived feature)
    data['Volume_Ratio'] = data[volume_col] / (data['Volume_MA_20'] + 1e-10)

    # On-Balance Volume (cumulative volume-weighted indicator)
    data['OBV'] = ta.volume.OnBalanceVolumeIndicator(
        close=data[close_col],
        volume=data[volume_col]
    ).on_balance_volume()

    return data
```

**Volume Features Explained:**

- **Volume_Ratio**: Current volume relative to 20-day average
  - Derived feature capturing unusual volume activity
  - > 1.0 indicates above-average volume
- **OBV (On-Balance Volume)**: Cumulative indicator linking volume to price direction
  - Rising OBV + rising price = Strong uptrend
  - Falling OBV + falling price = Strong downtrend
  - Divergences signal potential reversals

### Summary: 25 Features Used in Model

| Category | Count | Features | Purpose |
|----------|-------|----------|---------|
| **Price** | 5 | SPY_Close, Price_Change, Price_Change_5d, Close_Position, HL_Spread | Price levels and momentum |
| **Bollinger Bands** | 2 | BB_Position, BB_Width | Normalized volatility indicators |
| **Moving Averages** | 7 | EMA_8, EMA_21, SMA_50, SMA_200, EMA_Signal, Price_Above_SMA50, Price_Above_SMA200 | Trend identification with binary signals |
| **RSI** | 3 | RSI, RSI_Oversold, RSI_Overbought | Momentum with extreme zone detection |
| **ADX** | 2 | ADX, ADX_Strong | Trend strength measurement |
| **Volume** | 3 | SPY_Volume, Volume_Ratio, OBV | Volume analysis and confirmation |
| **Volatility** | 2 | Volatility_5d, Volatility_20d | Recent price variance |
| **VIX** | 1 | VIX_Close | Market-wide sentiment |

**Total: 25 features** passed to the DQN model

**Key Design Decisions:**
- **Derived features** (BB_Position, Volume_Ratio, Close_Position) provide scale-invariant, normalized inputs
- **Binary indicators** (EMA_Signal, RSI_Oversold, ADX_Strong) simplify signal interpretation for the neural network
- **Multi-timeframe** momentum (1d, 5d, 20d) captures different trend horizons
- Raw price bands and directional components (BB_High/BB_Low, ADX_Pos/ADX_Neg) are created but not used in the model

## Data Splitting: Avoiding Lookahead Bias

### The Lookahead Bias Problem

**Incorrect Approach:**
```python
# ❌ WRONG: Normalize entire dataset first
normalized_data = (data - data.mean()) / data.std()

# Then split
train = normalized_data[:1000]
test = normalized_data[1000:]
```

**Why it's wrong:** Test data statistics "leaked" into training normalization!

### Our Solution: Split First, Normalize Later

```python
class DataSplitter:
    """Split data into train, validation, and test sets."""

    def split_data(self, data: pd.DataFrame) -> Dict[str, pd.DataFrame]:
        """
        Split strategy:
        1. Extract test period (last N months/years)
        2. Extract validation periods (random non-overlapping periods)
        3. Remaining data → training
        """
        data = data.sort_index()

        # 1. Test period (most recent data)
        test_data, remaining_data = self._extract_test_period(data)

        # 2. Validation periods (random from remaining)
        validation_data, train_periods = self._extract_validation_periods(remaining_data)

        # 3. Training data (everything else)
        train_data = self._combine_train_periods(train_periods)

        return {
            'train': train_data,
            'validation': validation_data,  # List of periods
            'test': test_data
        }
```

### Test Period Extraction

```python
def _extract_test_period(self, data: pd.DataFrame) -> Tuple:
    """Extract most recent period for testing."""
    end_date = data.index.max()

    if self.test_unit == 'year':
        start_date = end_date - timedelta(days=365 * self.test_duration)
    elif self.test_unit == 'month':
        start_date = end_date - timedelta(days=30 * self.test_duration)

    test_data = data[data.index >= start_date]
    remaining_data = data[data.index < start_date]

    return test_data, remaining_data
```

### Validation Periods (Random Non-Overlapping)

```python
def _extract_validation_periods(self, data: pd.DataFrame) -> Tuple:
    """Extract N random non-overlapping periods for validation."""

    # Create candidate periods
    periods = self._create_period_buckets(data)

    # Randomly sample N non-overlapping periods
    random.shuffle(periods)
    validation_periods = []
    train_periods = []

    for period in periods:
        if len(validation_periods) < self.n_validation_periods:
            # Check for overlap with existing validation periods
            if not self._overlaps_with_existing(period, validation_periods):
                validation_periods.append(period)
        else:
            train_periods.append(period)

    return validation_periods, train_periods
```

**Example Split:**
```
Data: 2005-2025 (20 years)

Test: 2025 (last year)
Validation: 5 random years from 2005-2024
Training: Remaining ~14 years
```

This creates **diverse validation** across different market conditions (bull, bear, crisis).

## Rolling Normalization: Maintaining Stationarity

### Why Normalize?

Neural networks learn better with normalized inputs:
- Faster convergence
- Prevents features with large scales from dominating
- Stabilizes training

### Rolling Z-Score Normalization

```python
class Normalizer:
    """Apply rolling Z-score normalization to maintain stationarity."""

    def __init__(self, window: int = 30):
        """
        Args:
            window: Rolling window for mean/std calculation (default: 30 days)
        """
        self.window = window

    def normalize(self, data: pd.DataFrame) -> pd.DataFrame:
        """
        Apply rolling Z-score: (x - rolling_mean) / rolling_std
        """
        normalized = data.copy()

        for col in data.columns:
            if col.endswith('_orig'):  # Skip original price columns
                continue

            # Calculate rolling statistics
            rolling_mean = data[col].rolling(window=self.window, min_periods=1).mean()
            rolling_std = data[col].rolling(window=self.window, min_periods=1).std()

            # Z-score normalization
            normalized[col] = (data[col] - rolling_mean) / (rolling_std + 1e-10)

        return normalized
```

**Key Point: Continuous Timeline**

We normalize the **entire chronological dataset** before splitting:
```
Timeline: [────────────────────────]
          2005                  2025

1. Normalize entire timeline (rolling window)
2. THEN split into train/val/test

This preserves temporal relationships while preventing lookahead.
```

**Why This Works:**
- Rolling window only looks backward (no future data leakage)
- Each point normalized using only past 30 days
- Maintains time-series stationarity

### Example: SPY Close Price

```python
# Before normalization (actual prices)
Date         SPY_Close
2025-01-01   450.23
2025-01-02   448.67
2025-01-03   455.12

# After rolling Z-score (normalized)
Date         SPY_Close_normalized
2025-01-01    0.45
2025-01-02   -0.23
2025-01-03    1.12
```

## Putting It All Together: The Pipeline

```python
# 1. Data Collection
config_loader = ConfigLoader('config/my_project')
data_config = config_loader.load_data_config()

collector = DataCollector(data_config)
spy_data, vix_data = collector.collect_data()

# 2. Feature Engineering
engineer = FeatureEngineer(data_config)
featured_data = engineer.create_features(spy_data, vix_data)
# Result: 25 features

# 3. Data Splitting (BEFORE normalization)
splitter = DataSplitter(data_config)
splits = splitter.split_data(featured_data)
# Result: train, validation (5 periods), test

# 4. Normalization (continuous timeline)
normalizer = Normalizer(window=30)
normalized_timeline = normalizer.normalize_continuous_timeline(
    train=splits['train'],
    validation=splits['validation'],
    test=splits['test']
)

# Ready for training!
```

## Key Takeaways

1. **Smart Caching**: Saves time and API calls with intelligent file naming
2. **Buffer Days**: Automatically handles historical data needed for long-period indicators
3. **Feature Engineering**: 25 features across 4 categories capture price, momentum, trend, and sentiment
4. **Split First**: Prevents lookahead bias by splitting before normalization
5. **Rolling Normalization**: Maintains stationarity while respecting temporal ordering
6. **Diverse Validation**: Random periods test generalization across market conditions

## What's Next?

In **Part III**, we'll design the **Learning Environment**:
- State space representation (what does the agent observe?)
- Action space design (multi-buy, partial sells with FIFO)
- Reward function (how do we incentivize profitable behavior?)
- Risk management guardrails (stop-loss, take-profit)

Stay tuned!

---

**← Previous:** [Part I: Problem Statement & RL Motivation](/posts/2026/01/01/dqn-trading-part-I)

**Next Post →** [Part III: Learning Environment Design](/posts/2026/01/14/dqn-trading-part-III)

---

## References

- [yfinance Documentation](https://pypi.org/project/yfinance/) - Yahoo Finance API wrapper
- [TA-Lib Technical Indicators](https://technical-analysis-library-in-python.readthedocs.io/) - Technical analysis library
- [Bollinger Bands Explained](https://www.investopedia.com/terms/b/bollingerbands.asp) - Investopedia
- [RSI Indicator Guide](https://www.investopedia.com/terms/r/rsi.asp) - Investopedia
- [ADX Indicator Explained](https://www.investopedia.com/terms/a/adx.asp) - Investopedia
- [Avoiding Lookahead Bias](https://www.quantstart.com/articles/Avoiding-Look-Ahead-Bias-in-Time-Series-Modelling/) - QuantStart

[dqn-part-I]: /posts/2026/01/01/dqn-trading-part-I
[dqn-part-II]: /posts/2026/01/07/dqn-trading-part-II
[dqn-part-III]: /posts/2026/01/14/dqn-trading-part-III
[dqn-part-IV]: /posts/2026/01/17/dqn-trading-part-IV
[dqn-part-V]: /posts/2026/01/18/dqn-trading-part-V
