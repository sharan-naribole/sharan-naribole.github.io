---
title: 'Deep Q-Network for Stock Trading (Part IV): DQN Architecture Deep Dive'
date: 2026-01-17 12:00:00 -0800
permalink: /posts/2026/01/17/dqn-trading-part-IV
tags:
  - deep learning
  - double dqn
  - dueling dqn
  - neural networks
  - tensorflow
redirect_from:
  - /2026/01/17/dqn-trading-part-IV.html
toc: true
---

*This post is part of a series on building a Deep Q-Network (DQN) based trading system for SPY (S&P 500 ETF).*

- [Part I: Problem Statement & RL Motivation][dqn-part-I]
- [Part II: Data Engineering Pipeline][dqn-part-II]
- [Part III: Learning Environment Design][dqn-part-III]
- [Part IV: DQN Architecture Deep Dive][dqn-part-IV]
- [Part V: Software Architecture & Results][dqn-part-V]

---

**← Previous:** [Part III: Learning Environment Design](/posts/2026/01/14/dqn-trading-part-III)

**Next Post →** [Part V: Software Architecture & Results](/posts/2026/01/18/dqn-trading-part-V)

---

## ⚠️ Disclaimer

**This blog series is for educational and research purposes only.** The content should not be considered financial advice, investment advice, or trading advice. Trading stocks and financial instruments involves substantial risk of loss and is not suitable for every investor. Past performance does not guarantee future results. Always consult with a qualified financial advisor before making investment decisions.

---

## Introduction

In [Part III](/posts/2026/01/14/dqn-trading-part-III), we designed the trading environment with multi-buy accumulation and FIFO sells. Now it's time to build the **brain** of our system: the Deep Q-Network (DQN).

This post dives into:
1. **Double DQN**: Preventing Q-value overestimation
2. **Dueling Architecture**: Separating value and advantage
3. **Experience Replay**: Breaking temporal correlation
4. **Target Networks**: Stabilizing training
5. **Network Architecture**: Layers, activations, and hyperparameters

The complete code is in `src/models/dqn.py` on [GitHub](https://github.com/sharan-naribole/dqn-trading).

## The Q-Learning Foundation

### What is Q-Learning?

Q-learning learns a **Q-function** that estimates the expected cumulative reward:

```
Q(s, a) = Expected total reward from state s, taking action a
```

**Bellman Equation:**
```
Q(s, a) = r + γ × max_a' Q(s', a')
```

Where:
- `r`: Immediate reward
- `γ` (gamma): Discount factor (0.99)
- `s'`: Next state
- `max_a' Q(s', a')`: Best Q-value in next state

**Policy:** Choose action with highest Q-value:
```
π(s) = argmax_a Q(s, a)
```

### From Tabular to Deep Q-Learning

**Tabular Q-Learning** stores Q(s,a) in a table:
```
State    Action    Q-value
-----    ------    -------
s1       a1        0.5
s1       a2        0.3
s2       a1        0.8
...
```

**Problem:** Trading states are **continuous** (prices, indicators, etc.). Infinite table size!

**Solution:** Use a **neural network** to approximate Q(s,a):
```
Q(s, a) ≈ NN(s)[a]
```

Input: State → Output: Q-values for all actions

## Vanilla DQN vs. Our Improvements

### Vanilla DQN (2013)

DeepMind's original DQN used:
- Single network predicting Q-values
- Experience replay
- Target network

**Problems:**
1. **Overestimation bias**: max operator causes Q-values to be too optimistic
2. **No value/advantage separation**: Conflates state value with action advantages

### Our Enhanced DQN

We implement **two key improvements**:
1. **Double DQN** (2015): Fixes overestimation
2. **Dueling Architecture** (2016): Separates value and advantage

Let's explore each!

## Double DQN: Fixing Overestimation

### The Overestimation Problem

Vanilla DQN computes targets as:
```
Target = r + γ × max_a Q_target(s', a)
```

**Issue:** The same network selects AND evaluates the action:
```
max_a Q(s', a) = Q(s', argmax_a Q(s', a))
```

If Q-values have noise (they do!), max will pick **overestimated** values:

**Example:**
```
True Q-values:  [1.0, 1.2, 1.1]
Noisy Q-values: [1.5, 0.9, 1.3]  ← Noise added

max(true) = 1.2
max(noisy) = 1.5  ← Overestimate!
```

Over many updates, this compounds, causing Q-values to diverge.

### Double DQN Solution

**Key Idea:** Decouple selection from evaluation:

1. **Behavior Network** selects action:
   ```
   a* = argmax_a Q_behavior(s', a)
   ```

2. **Target Network** evaluates that action:
   ```
   Target = r + γ × Q_target(s', a*)
   ```

**Why This Works:**

If behavior network overestimates action a*, target network is unlikely to also overestimate it (independent noise). This reduces systematic overestimation.

### Implementation

```python
@tf.function
def train_step(self, states, actions, rewards, next_states, dones, gamma=0.99):
    """Double DQN training step."""

    # Current Q-values from behavior network
    current_q_values = self.q_network(states, training=True)
    actions_one_hot = tf.one_hot(actions, self.n_actions)
    current_q_values = tf.reduce_sum(current_q_values * actions_one_hot, axis=1)

    # Double DQN: Select action with behavior network
    next_q_values_behavior = self.q_network(next_states, training=False)
    next_actions = tf.argmax(next_q_values_behavior, axis=1)  # ← Selection

    # Evaluate selected action with target network
    next_q_values_target = self.target_network(next_states, training=False)
    next_actions_one_hot = tf.one_hot(next_actions, self.n_actions)
    next_q_values = tf.reduce_sum(next_q_values_target * next_actions_one_hot, axis=1)  # ← Evaluation

    # Compute targets
    targets = rewards + gamma * next_q_values * (1 - dones)

    # MSE loss
    loss = tf.keras.losses.MSE(targets, current_q_values)

    # Update behavior network only
    gradients = tape.gradient(loss, self.q_network.trainable_variables)
    self.optimizer.apply_gradients(zip(gradients, self.q_network.trainable_variables))

    return loss
```

**Key Points:**
- Behavior network updated every step
- Target network copied periodically (e.g., every 1-10 episodes)
- Decoupling reduces overestimation

## Dueling Architecture: Value vs. Advantage

### The Motivation

Some states are intrinsically valuable regardless of action:

**Example:**
```
State: Bull market, RSI=50, trending up, cash available

All actions have similar Q-values:
Q(s, HOLD) = 2.5
Q(s, BUY_10) = 2.7
Q(s, BUY_50) = 2.6
Q(s, BUY_100) = 2.8
```

**Observation:** The state itself is valuable (2.5 baseline). Actions provide small advantages (+0.0 to +0.3).

**Dueling Architecture Idea:**

Separate the **state value** from **action advantages**:

```
Q(s, a) = V(s) + A(s, a)
```

Where:
- `V(s)`: Value of being in state s (independent of action)
- `A(s, a)`: Advantage of action a in state s (relative to average)

### Dueling Formula

To ensure identifiability, we center advantages:

```
Q(s, a) = V(s) + (A(s, a) - mean_a A(s, a))
```

**Why subtract mean?**

Without centering, the network could output arbitrary V and A values that sum to the same Q. Centering forces:
- `V(s)`: Represents baseline state value
- `A(s, a) - mean(A)`: Relative advantage of action a

### Architecture Diagram

```
Input State (window_size, n_features)
         ↓
     [Flatten]
         ↓
  [Shared Layers]
    256 → 128
         ↓
    ┌─────┴─────┐
    ↓           ↓
[Value Stream] [Advantage Stream]
   128           128
    ↓             ↓
[V(s): 1]    [A(s,a): n_actions]
    └──────┬──────┘
           ↓
     Q(s,a) = V + A - mean(A)
```

### Implementation

```python
class DQNNetwork(Model):
    """Dueling DQN network."""

    def __init__(self, state_shape, n_actions, config):
        super().__init__()

        # Shared layers
        self.flatten = layers.Flatten()
        self.shared_1 = layers.Dense(256, activation='relu', name='shared_1')
        self.shared_2 = layers.Dense(128, activation='relu', name='shared_2')

        # Value stream
        self.value_hidden = layers.Dense(128, activation='relu', name='value_hidden')
        self.value_output = layers.Dense(1, name='value_output')  # Single value

        # Advantage stream
        self.advantage_hidden = layers.Dense(128, activation='relu', name='advantage_hidden')
        self.advantage_output = layers.Dense(n_actions, name='advantage_output')  # One per action

    def call(self, inputs, training=False):
        """Forward pass."""
        # Shared processing
        x = self.flatten(inputs)
        x = self.shared_1(x)
        x = self.shared_2(x)

        # Value stream
        value = self.value_hidden(x)
        value = self.value_output(value)  # Shape: (batch, 1)

        # Advantage stream
        advantage = self.advantage_hidden(x)
        advantage = self.advantage_output(advantage)  # Shape: (batch, n_actions)

        # Combine with centering
        q_values = value + (advantage - tf.reduce_mean(advantage, axis=1, keepdims=True))

        return q_values
```

**Benefits:**
1. **Faster Learning**: Network learns state values even when actions don't matter much
2. **Better Generalization**: Decoupling helps in states with clear best actions
3. **Robustness**: More stable Q-value estimates

## Experience Replay: Breaking Correlation

### The Problem with Online Learning

If we train on consecutive experiences:
```
Episode:
Step 1: (s1, a1, r1, s2)  → Train
Step 2: (s2, a2, r2, s3)  → Train
Step 3: (s3, a3, r3, s4)  → Train
...
```

**Issues:**
1. **Temporal Correlation**: Consecutive states are highly correlated
2. **Inefficient**: Each experience used once, then discarded
3. **Catastrophic Forgetting**: New experiences overwrite old learnings

### Experience Replay Solution

**Key Idea:** Store experiences in a buffer, sample random mini-batches for training.

```python
class ReplayBuffer:
    """Fixed-size buffer to store experience tuples."""

    def __init__(self, buffer_size: int = 10000):
        """
        Args:
            buffer_size: Maximum buffer size
        """
        self.buffer = []
        self.buffer_size = buffer_size
        self.position = 0

    def add(self, state, action, reward, next_state, done, next_valid_actions):
        """Add experience to buffer."""
        experience = (state, action, reward, next_state, done, next_valid_actions)

        if len(self.buffer) < self.buffer_size:
            self.buffer.append(experience)
        else:
            # Circular buffer: overwrite oldest
            self.buffer[self.position] = experience
            self.position = (self.position + 1) % self.buffer_size

    def sample(self, batch_size: int = 64):
        """Sample random mini-batch."""
        indices = np.random.choice(len(self.buffer), batch_size, replace=False)

        states = []
        actions = []
        rewards = []
        next_states = []
        dones = []
        next_valid_actions = []

        for idx in indices:
            s, a, r, s_next, d, nva = self.buffer[idx]
            states.append(s)
            actions.append(a)
            rewards.append(r)
            next_states.append(s_next)
            dones.append(d)
            next_valid_actions.append(nva)

        return (
            np.array(states),
            np.array(actions),
            np.array(rewards),
            np.array(next_states),
            np.array(dones),
            np.array(next_valid_actions)
        )
```

### Training Loop with Replay

```python
# Initialize
replay_buffer = ReplayBuffer(buffer_size=10000)
agent = DoubleDQN(state_shape, n_actions, config)

for episode in range(num_episodes):
    state = env.reset()

    for step in range(max_steps):
        # Select action
        valid_actions = env.get_action_mask()
        action = agent.get_action(state, valid_actions, epsilon)

        # Execute action
        next_state, reward, done, info = env.step(action)

        # Store experience
        next_valid_actions = env.get_action_mask()
        replay_buffer.add(state, action, reward, next_state, done, next_valid_actions)

        # Train from random mini-batch
        if len(replay_buffer) >= batch_size:
            batch = replay_buffer.sample(batch_size)
            loss = agent.train_step(*batch, gamma=0.99)

        state = next_state
        if done:
            break
```

**Benefits:**
1. **Breaks Correlation**: Random sampling decorrelates training samples
2. **Data Efficiency**: Each experience used multiple times
3. **Stable Learning**: Smooth out variance in updates

## Target Network: Stabilizing the Target

### The Moving Target Problem

Q-learning updates look like:
```
Q(s, a) ← Q(s, a) + α[r + γ max_a' Q(s', a') - Q(s, a)]
                          ↑
                    This is the target
```

**Problem:** The target depends on Q itself! As we update Q, the target moves.

**Analogy:** Imagine learning to shoot basketballs, but the hoop moves every time you shoot. Impossible to converge!

### Target Network Solution

**Key Idea:** Use a **frozen copy** of the network for computing targets:

1. **Behavior Network (Q)**: Updated every training step
2. **Target Network (Q')**: Frozen for N steps, then copied from Q

```python
class DoubleDQN:
    def __init__(self, state_shape, n_actions, config):
        # Create two identical networks
        self.q_network = DQNNetwork(state_shape, n_actions, config, 'behavior')
        self.target_network = DQNNetwork(state_shape, n_actions, config, 'target')

        # Initialize target with behavior weights
        self.update_target_network()

    def update_target_network(self):
        """Copy weights from behavior to target."""
        self.target_network.set_weights(self.q_network.get_weights())

    def train(self, states, actions, rewards, next_states, dones):
        # Compute targets using FROZEN target network
        next_q_values = self.target_network(next_states)
        targets = rewards + gamma * tf.reduce_max(next_q_values, axis=1)

        # Update behavior network toward these fixed targets
        with tf.GradientTape() as tape:
            current_q_values = self.q_network(states)
            loss = MSE(targets, current_q_values)

        gradients = tape.gradient(loss, self.q_network.trainable_variables)
        self.optimizer.apply_gradients(zip(gradients, self.q_network.trainable_variables))

# Training loop
for episode in range(num_episodes):
    # ... collect experience and train ...

    # Update target network every N episodes
    if episode % target_update_freq == 0:
        agent.update_target_network()
```

**Analogy:** Now the basketball hoop stays still for 10 shots, then moves to a new position. Much easier to learn!

**Hyperparameter:** `target_update_freq`
- Low (e.g., 1): Frequent updates, less stable
- High (e.g., 10): More stable, but slower to adapt
- Our default: 1 episode (works well with multiple training steps per episode)

## Network Architecture Configuration

### Configurable Design

Our system allows flexible network configuration via JSON:

```json
{
  "network": {
    "architecture": "dueling",
    "shared_layers": [256, 128],
    "value_layers": [128],
    "advantage_layers": [128],
    "activation": "relu",
    "dropout_rate": 0.0,
    "batch_norm": false
  }
}
```

### Layer Sizes

**Shared Layers:** `[256, 128]`
- First layer: 256 units (large capacity for complex patterns)
- Second layer: 128 units (dimensionality reduction)

**Value/Advantage Streams:** `[128]`
- Single hidden layer with 128 units each
- Keeps streams symmetric and manageable

**Total Parameters:**

For state_shape=(5, 27) and n_actions=8:

```
Input: 5 × 27 = 135 features

Shared:
  Layer 1: 135 × 256 + 256 bias = 34,816
  Layer 2: 256 × 128 + 128 bias = 32,896

Value Stream:
  Hidden: 128 × 128 + 128 bias = 16,512
  Output: 128 × 1 + 1 bias = 129

Advantage Stream:
  Hidden: 128 × 128 + 128 bias = 16,512
  Output: 128 × 8 + 8 bias = 1,032

Total: ~102K parameters
```

### Activation Functions

**ReLU (Rectified Linear Unit):**
```
f(x) = max(0, x)
```

**Pros:**
- Fast to compute
- No vanishing gradient
- Sparse activation (some neurons off)

**Cons:**
- "Dying ReLU" problem (neurons stuck at 0)

**Alternatives:**
- **ELU (Exponential Linear Unit)**: Smoother, no dying ReLU
- **tanh**: Outputs in [-1, 1], useful for normalized data

Our default: **ReLU** (standard, reliable)

### Regularization

**Dropout (optional):**
```json
"dropout_rate": 0.2
```

Randomly drops 20% of neurons during training. Prevents overfitting.

**Batch Normalization (optional):**
```json
"batch_norm": true
```

Normalizes layer inputs. Can speed up training but adds complexity.

**Our default:** No dropout or batch norm (unnecessary for our data size and simplicity)

## Hyperparameters Summary

| Hyperparameter | Value | Purpose |
|----------------|-------|---------|
| **Learning Rate** | 0.001 | Step size for gradient descent |
| **Gamma (γ)** | 0.99 | Discount factor (long-term focus) |
| **Epsilon Start** | 1.0 | Initial exploration rate |
| **Epsilon End** | 0.01 | Minimum exploration rate |
| **Epsilon Decay** | 0.995 | Decay per episode |
| **Replay Buffer Size** | 10,000 | Experiences stored |
| **Batch Size** | 64 | Mini-batch size for training |
| **Target Update Freq** | 1 episode | How often to sync target network |
| **Optimizer** | Adam | Adaptive learning rate optimizer |

### Epsilon-Greedy Exploration

Balances exploration vs. exploitation:

```
if random() < epsilon:
    action = random_valid_action()  # Explore
else:
    action = argmax Q(s, a)          # Exploit
```

**Epsilon Schedule:**
```python
epsilon = max(epsilon_end, epsilon * epsilon_decay)
```

**Example:**
```
Episode 1:   ε = 1.0   (100% random)
Episode 10:  ε = 0.95  (95% random)
Episode 50:  ε = 0.78  (78% random)
Episode 100: ε = 0.60  (60% random)
Episode 500: ε = 0.08  (8% random)
Episode 1000: ε = 0.01 (1% random)
```

Early episodes explore broadly; later episodes exploit learned policy.

## Putting It All Together

### Complete Training Algorithm

```
1. Initialize:
   - Behavior network Q
   - Target network Q' (copy of Q)
   - Replay buffer D
   - Epsilon = 1.0

2. For each episode:
   a. Reset environment → state s
   b. For each step:
      i.   Select action a using epsilon-greedy with action masking
      ii.  Execute a → reward r, next state s', done
      iii. Store (s, a, r, s', done) in D
      iv.  If |D| >= batch_size:
           - Sample mini-batch from D
           - Compute Double DQN targets using Q'
           - Train Q on mini-batch
      v.   s ← s'
      vi.  If done, break

   c. Decay epsilon: ε ← ε × 0.995
   d. Update target: Q' ← Q (every N episodes)

3. Save best model based on validation performance
```

### Key Innovations Recap

1. **Double DQN**: Decouples action selection from evaluation → reduces overestimation
2. **Dueling Architecture**: Separates V(s) and A(s,a) → faster learning
3. **Experience Replay**: Random mini-batches → stable training
4. **Target Network**: Frozen targets → convergence
5. **Action Masking**: Invalid actions filtered → efficiency

## What's Next?

In **Part V (Final)**, we'll cover:
- **Software Architecture**: Modular design, configuration system
- **Training Pipeline**: Dry run, full training, monitoring
- **Results Analysis**: Out-of-sample validation across 5 market periods
- **Strategy Comparison**: Baseline (20% SL/TP) vs. Aggressive (10%) vs. No Guardrails
- **Performance Metrics**: Total return, Sharpe ratio, max drawdown, win rate
- **Key Insights**: What worked, what didn't, lessons learned

The DQN brain is complete—now let's see how it performs in the real market!

---

**← Previous:** [Part III: Learning Environment Design](/posts/2026/01/14/dqn-trading-part-III)

**Next Post →** [Part V: Software Architecture & Results](/posts/2026/01/18/dqn-trading-part-V)

---

## References

- [Human-level control through deep reinforcement learning](https://www.nature.com/articles/nature14236) - Original DQN paper (2015)
- [Deep Reinforcement Learning with Double Q-learning](https://arxiv.org/abs/1509.06461) - Double DQN paper (2015)
- [Dueling Network Architectures for Deep Reinforcement Learning](https://arxiv.org/abs/1511.06581) - Dueling DQN paper (2016)
- [Prioritized Experience Replay](https://arxiv.org/abs/1511.05952) - Advanced replay technique
- [Rainbow: Combining Improvements in Deep Reinforcement Learning](https://arxiv.org/abs/1710.02298) - Combines 6 DQN improvements
- [TensorFlow Documentation](https://www.tensorflow.org/api_docs) - TensorFlow 2.x API

[dqn-part-I]: /posts/2026/01/01/dqn-trading-part-I
[dqn-part-II]: /posts/2026/01/07/dqn-trading-part-II
[dqn-part-III]: /posts/2026/01/14/dqn-trading-part-III
[dqn-part-IV]: /posts/2026/01/17/dqn-trading-part-IV
[dqn-part-V]: /posts/2026/01/18/dqn-trading-part-V
