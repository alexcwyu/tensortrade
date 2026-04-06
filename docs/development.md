# TensorTrade -- Development Guide

## Setting Up the Development Environment

### Prerequisites

- Python 3.12+
- pip or uv

### Installation

```bash
# Clone and install
git clone <repo-url>
cd tensortrade
pip install -e .

# Or with uv
uv sync --active --all-groups --all-extras

# Optional dependencies for built-in agents
pip install tensorflow   # For DQN/A2C agents
pip install torch        # For parallel DQN agent
```

## Running Tests

```bash
# Run all tests
pytest tests/

# Run specific test module
pytest tests/test_environment.py

# With coverage
pytest --cov=tensortrade tests/
```

## Building a Trading Environment

### Step 1: Prepare Data

```python
import pandas as pd

# Load OHLCV data
data = pd.read_csv("btc_usd.csv")
price_data = data["close"].values
volume_data = data["volume"].values
```

### Step 2: Create Exchange

The `Exchange` requires a name, an execution service, and price streams:

```python
from tensortrade.oms.exchanges import Exchange, ExchangeOptions
from tensortrade.oms.services.execution.simulated import execute_order
from tensortrade.feed.core import Stream

# Configure exchange options
options = ExchangeOptions(
    commission=0.003,       # 0.3% commission
    min_trade_size=1e-6,
    max_trade_size=1e6,
)

# Create exchange with price stream
exchange = Exchange("simulated", service=execute_order, options=options)(
    Stream.source(price_data, dtype="float").rename("USD-BTC")
)
```

The price stream name must follow the format `{quote}-{base}` (e.g., `USD-BTC` means "price of BTC in USD").

### Step 3: Create Portfolio

```python
from tensortrade.oms.instruments import Instrument, USD, BTC
from tensortrade.oms.wallets import Wallet, Portfolio

portfolio = Portfolio(USD, [
    Wallet(exchange, 10000 * USD),    # Starting cash
    Wallet(exchange, 0 * BTC),         # No initial BTC
])
```

### Step 4: Create Data Feed

```python
from tensortrade.feed.core import DataFeed, Stream

# Feature streams for the agent
price_stream = Stream.source(price_data, dtype="float").rename("price")
volume_stream = Stream.source(volume_data, dtype="float").rename("volume")

# Add derived features
returns = price_stream.pct_change().fillna(0).rename("returns")
volatility = price_stream.rolling(window=20).std().fillna(0).rename("volatility")

feed = DataFeed([price_stream, volume_stream, returns, volatility])
```

### Step 5: Choose Components

```python
import tensortrade.env.default as default

env = default.create(
    portfolio=portfolio,
    action_scheme="bsh",           # Buy/Sell/Hold
    reward_scheme="risk-adjusted", # Sharpe ratio
    feed=feed,
    window_size=20,                # Observation window
    max_episode_steps=500,         # Max steps per episode
)
```

Or construct manually:

```python
from tensortrade.env.generic import TradingEnv
from tensortrade.env.default.actions import BSH
from tensortrade.env.default.rewards import RiskAdjustedReturns
from tensortrade.env.default.observers import TensorTradeObserver
from tensortrade.env.default.stoppers import MaxLossStopper
from tensortrade.env.default.informers import TensorTradeInformer
from tensortrade.env.default.renderers import EmptyRenderer

cash = portfolio.get_wallet(exchange.id, USD)
asset = portfolio.get_wallet(exchange.id, BTC)

env = TradingEnv(
    action_scheme=BSH(cash, asset),
    reward_scheme=RiskAdjustedReturns(return_algorithm="sharpe", window_size=20),
    observer=TensorTradeObserver(portfolio, feed, window_size=20, min_periods=10),
    stopper=MaxLossStopper(max_allowed_loss=0.2),
    informer=TensorTradeInformer(),
    renderer=EmptyRenderer(),
    max_episode_steps=500,
)
```

## Creating Custom Components

### Custom Action Scheme

```python
from tensortrade.env.default.actions import TensorTradeActionScheme
from gymnasium.spaces import Discrete

class ThreeWayAction(TensorTradeActionScheme):
    """Buy 100%, Sell 100%, or Hold."""

    registered_name = "three-way"

    def __init__(self, cash, asset):
        super().__init__()
        self.cash = cash
        self.asset = asset

    @property
    def action_space(self):
        return Discrete(3)  # 0=hold, 1=buy, 2=sell

    def get_orders(self, action, portfolio):
        if action == 0:
            return []
        elif action == 1:
            return [proportion_order(portfolio, self.cash, self.asset, 1.0)]
        else:
            return [proportion_order(portfolio, self.asset, self.cash, 1.0)]
```

### Custom Reward Scheme

```python
from tensortrade.env.default.rewards import TensorTradeRewardScheme

class DrawdownPenalizedReward(TensorTradeRewardScheme):
    """Rewards net worth growth, penalizes drawdowns."""

    registered_name = "drawdown-penalized"

    def __init__(self, drawdown_penalty=2.0):
        self.drawdown_penalty = drawdown_penalty
        self._peak_net_worth = None

    def get_reward(self, portfolio):
        net_worth = portfolio.net_worth

        if self._peak_net_worth is None:
            self._peak_net_worth = net_worth
            return 0.0

        # Update peak
        if net_worth > self._peak_net_worth:
            self._peak_net_worth = net_worth

        # Calculate reward
        growth = net_worth / portfolio.initial_net_worth - 1.0
        drawdown = (self._peak_net_worth - net_worth) / self._peak_net_worth

        return growth - self.drawdown_penalty * drawdown

    def reset(self):
        self._peak_net_worth = None
```

### Custom Observer

```python
from tensortrade.env.generic import Observer
from gymnasium.spaces import Box
import numpy as np

class SimpleObserver(Observer):
    """A minimal observer returning raw price and volume."""

    def __init__(self, feed, window_size=10):
        self.feed = feed
        self.window_size = window_size
        self.history = []
        self._observation_space = Box(
            low=-np.inf, high=np.inf,
            shape=(window_size, 2),  # price + volume
            dtype=np.float32
        )

    @property
    def observation_space(self):
        return self._observation_space

    def observe(self, env):
        data = self.feed.next()
        self.history.append([data["price"], data["volume"]])
        if len(self.history) > self.window_size:
            self.history.pop(0)
        obs = np.array(self.history[-self.window_size:], dtype=np.float32)
        if len(obs) < self.window_size:
            padding = np.zeros((self.window_size - len(obs), 2))
            obs = np.vstack([padding, obs])
        return obs

    def has_next(self):
        return self.feed.has_next()

    def reset(self, random_start=0):
        self.history = []
        self.feed.reset(random_start)
```

## Working with the Feed System

### Stream Operations

```python
from tensortrade.feed.core import Stream

# Source from iterable
prices = Stream.source([100, 101, 99, 102, 103], dtype="float").rename("price")

# Arithmetic
returns = prices.pct_change().fillna(0).rename("returns")
log_prices = prices.log().rename("log_price")

# Window operations
sma_20 = prices.rolling(window=20).mean().rename("sma_20")
ema_12 = prices.ewm(span=12).mean().rename("ema_12")
expanding_max = prices.expanding().max().rename("expanding_max")

# Boolean streams
above_sma = prices.gt(sma_20).rename("above_sma")

# Combining streams
spread = prices.sub(sma_20).rename("spread")

# Sensor (live observation)
balance_stream = Stream.sensor(
    wallet, lambda w: w.balance.as_float(), dtype="float"
).rename("balance")
```

### NameSpace for Scoping

```python
from tensortrade.feed.core import NameSpace

with NameSpace("binance"):
    price = Stream.source(data, dtype="float").rename("BTC-USD")
    # Stream name becomes "binance:/BTC-USD"
```

### PushFeed for Live Data

```python
from tensortrade.feed.core import PushFeed, Stream

price = Stream.placeholder(dtype="float").rename("price")
volume = Stream.placeholder(dtype="float").rename("volume")

feed = PushFeed([price, volume])

# Each tick:
output = feed.push({"price": 50000.0, "volume": 1234.5})
```

## Using Stochastic Processes for Testing

```python
from tensortrade.stochastic.processes import gbm, heston

# Generate synthetic price data
prices_gbm = gbm(base_price=100, t=1.0, mu=0.1, sigma=0.3, steps=1000)
prices_heston = heston(base_price=100, t=1.0, mu=0.1, steps=1000)
```

Available processes:
- `gbm` -- Geometric Brownian Motion
- `heston` -- Stochastic volatility
- `merton` -- Jump diffusion
- `cox` -- Cox-Ingersoll-Ross
- `ornstein_uhlenbeck` -- Mean-reverting
- `fbm` -- Fractional Brownian Motion

## Using TradingContext for Configuration

```python
from tensortrade.core import TradingContext

config = {
    "shared": {
        "base_instrument": "USD",
    },
    "exchanges": {
        "commission": 0.001,
    },
    "actions": {
        "trade_sizes": [0.25, 0.5, 1.0],
    },
    "rewards": {
        "window_size": 10,
    }
}

with TradingContext(config):
    # Components auto-receive config via Component metaclass
    exchange = Exchange("sim", service=execute_order)
    # exchange.context.commission == 0.001
```

Load from file:

```python
context = TradingContext.from_json("config.json")
context = TradingContext.from_yaml("config.yaml")
```

## Integration with RL Libraries

### Stable Baselines3

```python
from stable_baselines3 import PPO, A2C, DQN

env = create_env(...)

# PPO
model = PPO("MlpPolicy", env, verbose=1, learning_rate=3e-4)
model.learn(total_timesteps=100_000)

# A2C
model = A2C("MlpPolicy", env, verbose=1)
model.learn(total_timesteps=100_000)
```

### Ray RLlib

```python
from ray.rllib.algorithms.ppo import PPOConfig

config = (
    PPOConfig()
    .environment(env=create_env)
    .training(lr=1e-4, train_batch_size=4000)
)
algo = config.build()
for _ in range(100):
    result = algo.train()
```

## Architecture Patterns

### Component Registration

All `Component` subclasses are automatically registered via `__init_subclass__`:

```python
class MyAction(ActionScheme):
    registered_name = "my-action"  # Used for context lookup
    # Automatically registered in global registry
```

### Observable Pattern

Orders, Wallets, and Streams use the Observable pattern:

```python
order.attach(broker)       # Broker listens to order events
order.attach(listener)     # Custom listener for trade events
feed.attach(portfolio)     # Portfolio updates on feed data
```

### Fund Conservation

The `Wallet.transfer()` static method enforces a conservation equation:

```
(source_locked_before - source_locked_after) - (quantity + commission)
    == (target_locked_after - target_locked_before) - converted_quantity
```

Both sides must equal zero (within quantization tolerance). Violations raise exceptions.

## Common Pitfalls

1. **Price stream naming**: Must match the instrument pair format. For `BTC/USD` trading pair, the stream should be named `USD-BTC` (quote first, then base).

2. **Quantity precision**: Use `Quantity.quantize()` to avoid floating-point precision issues. The framework uses `Decimal` internally.

3. **Feed compilation**: Always ensure the `DataFeed` has been compiled (or call `next()` which auto-compiles) before using it.

4. **Agent deprecation**: Built-in DQN/A2C agents are deprecated. Use Stable Baselines3, Ray RLlib, or other Gymnasium-compatible libraries.

5. **Random start**: Set `random_start_pct > 0` in `TradingEnv` to prevent overfitting to early data patterns during training.

6. **Observation space**: Ensure your observer's `observation_space` matches the actual shape of observations, or RL libraries will raise errors.

7. **Component clocks**: All components must share the same clock. The `TradingEnv` handles this automatically, but manual construction requires explicit clock propagation.

## Configuration Reference

### `default.create()` Environment Factory

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `portfolio` | `Portfolio` | _(required)_ | Portfolio with wallets for the trading environment |
| `action_scheme` | `ActionScheme` or `str` | _(required)_ | Action scheme: `"bsh"` (Buy/Sell/Hold), `"simple"`, `"managed-risk"`, or custom instance |
| `reward_scheme` | `RewardScheme` or `str` | _(required)_ | Reward scheme: `"simple"`, `"risk-adjusted"`, `"pbr"`, or custom instance |
| `feed` | `DataFeed` | _(required)_ | Data feed providing observation features to the agent |
| `window_size` | int | `1` | Number of time steps in the observation look-back window |
| `min_periods` | int | `None` | Minimum warmup steps before the feed produces valid data |
| `random_start_pct` | float | `0.0` | Randomize episode start within this percentage of data (0.0-1.0) |
| `max_allowed_loss` | float | `0.5` | Maximum portfolio loss before episode termination (via `MaxLossStopper`) |
| `renderer` | `Renderer`, `str`, or list | `EmptyRenderer()` | Visualization renderer(s) for episode playback |
| `informer` | `Informer` | `TensorTradeInformer()` | Informer providing step-level metadata in the `info` dict |
| `stopper` | `Stopper` | `MaxLossStopper(0.5)` | Custom stopper to override the default loss-based stopper |
| `renderer_feed` | `DataFeed` | `None` | Separate feed for rendering (avoids consuming the main feed) |
| `device` | str | `None` | Device for GPU-aware observations (e.g., `"cuda:0"`) |

### `ExchangeOptions`

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `commission` | float | `0.003` | Percentage of order size taken as commission (0.3%) |
| `min_trade_size` | float | `1e-6` | Minimum allowed trade size |
| `max_trade_size` | float | `1e6` | Maximum allowed trade size |
| `min_trade_price` | float | `1e-8` | Minimum allowed trade price |
| `max_trade_price` | float | `1e8` | Maximum allowed trade price |
| `is_live` | bool | `False` | Whether to submit orders to a live exchange |

### `RiskAdjustedReturns` Reward Scheme

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `return_algorithm` | str | `"sharpe"` | Risk metric: `"sharpe"` or `"sortino"` |
| `risk_free_rate` | float | `0.0` | Risk-free rate for ratio calculation |
| `target_returns` | float | `0.0` | Target returns for Sortino denominator |
| `window_size` | int | `1` | Rolling window size for computing the ratio |

### `TradingContext` Configuration Keys

| Key | Scope | Description |
|-----|-------|-------------|
| `shared.base_instrument` | Global | Base currency instrument (e.g., `"USD"`) |
| `exchanges.commission` | Exchange | Default commission for exchanges created in context |
| `actions.trade_sizes` | Actions | Available trade size fractions (e.g., `[0.25, 0.5, 1.0]`) |
| `rewards.window_size` | Rewards | Default window size for reward calculation |

### `SimpleProfit` Reward Scheme

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `window_size` | int | `1` | Look-back window for net worth change calculation |

## Troubleshooting

### 1. `ValueError: Price of trading pair ... is 0`
**Cause**: The price stream has a zero value, which is invalid for trading operations.
**Solution**: Check your input data for zero or missing price values. Ensure the price stream covers all time steps in your episode.

### 2. `KeyError` when accessing a trading pair on the exchange
**Cause**: The price stream name does not match the expected instrument pair format.
**Solution**: Price streams must follow `{quote}-{base}` naming. For BTC priced in USD, use `Stream.source(...).rename("USD-BTC")`, not `"BTC-USD"`.

### 3. `gymnasium.error.InvalidAction` or action space mismatch
**Cause**: The RL agent produces actions outside the environment's action space.
**Solution**: Ensure your policy network output matches `env.action_space`. For `BSH`, the space is `Discrete(2)` per exchange pair. Check with `print(env.action_space)`.

### 4. Observation shape mismatch with RL library
**Cause**: The observer's `observation_space` does not match actual observation dimensions.
**Solution**: Verify `window_size` and the number of feed streams. The default observer shape is `(window_size, n_features)`. Print `env.observation_space.shape` to debug.

### 5. `InsufficientFunds` error during order execution
**Cause**: The wallet does not have enough balance to execute the order after commission.
**Solution**: Reduce trade sizes, increase initial wallet balance, or check that commission settings are not unexpectedly high.

### 6. `TypeError: 'NoneType' object has no attribute 'value'` in Stream
**Cause**: The feed has not been compiled or has run out of data.
**Solution**: Ensure `DataFeed` is compiled before calling `next()`. The `TradingEnv` handles this automatically, but manual feed usage requires explicit `feed.compile()`.

### 7. Built-in DQN/A2C agent produces poor results
**Cause**: The built-in agents are deprecated and not well-maintained.
**Solution**: Use Stable Baselines3, Ray RLlib, or another Gymnasium-compatible RL library instead. The built-in agents exist only for backward compatibility.

### 8. `RecursionError` when building complex feed DAGs
**Cause**: Circular dependencies in stream operations.
**Solution**: Review your feed construction. Ensure no stream references itself (directly or transitively). Use `NameSpace` to scope streams and avoid accidental name collisions.

## Security Considerations

### API Key Management
- **CCXT execution service**: If using the CCXT execution backend for live trading, API keys are passed directly to the CCXT library. Never hardcode them; use environment variables.
- **Interactive Brokers**: IB connections use TWS/Gateway authentication. Ensure the TWS API port is not exposed to the network.
- Store API credentials in environment variables or a secrets manager, never in source code or configuration files checked into version control.

### Credential Storage
- `TradingContext.from_json()` and `from_yaml()` load configuration from files. If these files contain API keys, ensure they are excluded from version control (`.gitignore`).
- When using live execution services, credentials flow through the `Exchange` service callable. Audit the service function to understand what credentials it requires and how they are handled.

### Network Security
- Simulated execution (`execute_order`) is entirely local and makes no network calls.
- CCXT and IB execution services make outbound network connections. Ensure these run over TLS.
- If training RL agents with Ray, Ray's distributed execution opens network ports. Use Ray's authentication and TLS features in multi-node setups.

### Safe Practices
- Use `ExchangeOptions(is_live=False)` (the default) during development and testing to prevent accidental order submission.
- Validate trained agents extensively in simulated environments before switching to live execution.
- Set `max_trade_size` in `ExchangeOptions` to limit the maximum order size as a safety guard.
- Monitor wallet balances and portfolio net worth during live trading to detect anomalies early.
- Use the `MaxLossStopper` to automatically halt trading when losses exceed a threshold.

---
## See Also
- [README](README.md) — Project overview and quick start
- [Architecture](architecture.md) — System design and components
- [Workflow](workflow.md) — Event flows and processing pipelines
- [State Management](state-management.md) — State lifecycle and data models
