# TensorTrade -- Workflows

## Environment Setup Workflow

```mermaid
flowchart TD
    A[Prepare Market Data] --> B[Create Price Streams]
    B --> C[Configure Exchange]
    C --> D[Create Wallets]
    D --> E[Build Portfolio]
    E --> F[Create Data Feed]
    F --> G[Select Components]
    G --> H[Build TradingEnv]
    H --> I[Train Agent]
    I --> J{Evaluate}
    J -->|Adjust| G
    J -->|Deploy| K[Live Trading]
```

## Step-by-Step Environment Construction

```mermaid
sequenceDiagram
    participant User
    participant Exchange
    participant Portfolio
    participant DataFeed
    participant TradingEnv
    participant Agent

    User->>Exchange: Create with price streams
    User->>Portfolio: Create with wallets (USD, BTC)
    User->>DataFeed: Create with feature streams
    User->>TradingEnv: Compose components
    Note over TradingEnv: action_scheme, reward_scheme,<br/>observer, stopper, informer, renderer

    Agent->>TradingEnv: reset()
    TradingEnv->>TradingEnv: Reset clock, components
    TradingEnv-->>Agent: (observation, info)

    loop Training Episode
        Agent->>TradingEnv: step(action)
        TradingEnv->>TradingEnv: action_scheme.perform(action)
        Note over TradingEnv: Broker submits orders,<br/>Exchange executes trades
        TradingEnv->>TradingEnv: observer.observe()
        TradingEnv->>TradingEnv: reward_scheme.reward()
        TradingEnv->>TradingEnv: stopper.stop()
        TradingEnv->>TradingEnv: clock.increment()
        TradingEnv-->>Agent: (obs, reward, terminated, truncated, info)
    end
```

## Order Execution Workflow

```mermaid
sequenceDiagram
    participant ActionScheme
    participant Broker
    participant Order
    participant Wallet as Source Wallet
    participant Exchange
    participant TargetWallet as Target Wallet
    participant Ledger

    ActionScheme->>ActionScheme: get_orders(action, portfolio)
    ActionScheme->>Broker: submit(order)
    Broker->>Broker: Add to unexecuted queue

    Note over Broker: On broker.update()
    Broker->>Order: Check is_executable
    
    alt Order is executable
        Broker->>Order: execute()
        Order->>Wallet: lock(quantity)
        Wallet->>Ledger: commit(LOCK)
        Order->>Exchange: execute_order(order, portfolio)
        Exchange->>Exchange: service(order, wallets, price)
        Exchange->>Order: fill(trade)
        Order->>Order: remaining -= filled
        
        alt Order is complete
            Order->>Order: complete()
            Order->>Wallet: unlock remaining
            Wallet->>Ledger: commit(UNLOCK)
            
            alt Has OrderSpec (follow-up)
                Order->>Order: create_order from spec
                Note over Order: e.g., stop-loss after entry fill
                Broker->>Broker: submit(next_order)
            end
        end
    end

    alt Order expired
        Broker->>Order: cancel()
        Order->>Wallet: release locked funds
        Wallet->>Ledger: commit(RELEASE)
    end
```

## Data Feed Pipeline

```mermaid
flowchart TD
    subgraph Sources["Data Sources"]
        PriceData["Price Data<br/>(IterableStream)"]
        VolumeData["Volume Data<br/>(IterableStream)"]
        WalletSensor["Wallet Balance<br/>(Sensor)"]
        NetWorthSensor["Net Worth<br/>(Sensor)"]
    end

    subgraph Transform["Transformations"]
        PctChange["pct_change()"]
        EWM["ewm(span=20).mean()"]
        Rolling["rolling(10).std()"]
        Fillna["fillna(0)"]
    end

    subgraph Groups["Grouping"]
        External["External Group<br/>(feature observations)"]
        Internal["Internal Group<br/>(portfolio state)"]
        RendererGroup["Renderer Group<br/>(display data)"]
    end

    subgraph Output["Output"]
        MasterFeed["DataFeed"]
        Observer["Observer"]
        PortfolioListener["Portfolio<br/>(performance tracking)"]
    end

    PriceData --> PctChange --> External
    PriceData --> EWM --> External
    VolumeData --> Rolling --> External
    PriceData --> Fillna --> External
    WalletSensor --> Internal
    NetWorthSensor --> Internal
    PriceData --> RendererGroup

    External --> MasterFeed
    Internal --> MasterFeed
    RendererGroup --> MasterFeed
    MasterFeed --> Observer
    MasterFeed --> PortfolioListener
```

### Feed Compilation

When `DataFeed.compile()` is called:

1. **Gather**: Traverses the stream DAG to collect all edges
2. **Topological Sort**: Orders streams so dependencies run first
3. **Execute**: On each `next()` call, runs all streams in sorted order
4. **Output**: Returns a dictionary mapping stream names to current values

## Training Workflow

### With External RL Library (Recommended)

```python
# Using Stable Baselines3
from stable_baselines3 import PPO

env = create_trading_env(...)  # TradingEnv instance

model = PPO("MlpPolicy", env, verbose=1)
model.learn(total_timesteps=100_000)

# Evaluate
obs, info = env.reset()
for _ in range(1000):
    action, _ = model.predict(obs)
    obs, reward, terminated, truncated, info = env.step(action)
    if terminated or truncated:
        obs, info = env.reset()
```

### With Built-in DQN Agent (Deprecated)

```python
from tensortrade.agents import DQNAgent

agent = DQNAgent(env)
mean_reward = agent.train(
    n_steps=1000,
    n_episodes=50,
    batch_size=256,
    learning_rate=0.01,
    discount_factor=0.95,
    eps_start=0.9,
    eps_end=0.05,
)
```

### Training Loop Detail

```mermaid
stateDiagram-v2
    [*] --> Reset: env.reset()
    Reset --> GetAction: observation
    
    GetAction --> Step: agent.get_action(obs)
    Step --> StoreTransition: (obs, reward, done, info)
    StoreTransition --> CheckMemory: Push to replay memory
    
    CheckMemory --> GradientDescent: memory >= batch_size
    CheckMemory --> GetAction: memory < batch_size
    
    GradientDescent --> UpdateTarget: Every N steps
    UpdateTarget --> CheckDone
    GradientDescent --> CheckDone
    
    CheckDone --> GetAction: Not done
    CheckDone --> Render: Done
    Render --> SaveCheckpoint: Every K episodes
    SaveCheckpoint --> Reset: More episodes
    SaveCheckpoint --> [*]: Training complete
```

## Live Trading Workflow

```mermaid
flowchart TD
    A[Create PushFeed] --> B[Connect to Exchange API]
    B --> C[Create Live Exchange]
    C --> D[Build Environment]
    D --> E[Load Trained Agent]
    
    E --> F{Market Open?}
    F -->|Yes| G[Receive Market Data]
    G --> H[Push to PushFeed]
    H --> I[Agent Predicts Action]
    I --> J[Execute on Live Exchange]
    J --> K[Update Portfolio]
    K --> F
    F -->|No| L[Wait]
    L --> F
```

For live trading, use `PushFeed` instead of `DataFeed`:

```python
from tensortrade.feed.core import PushFeed, Stream

# Create placeholder streams for live data
price_placeholder = Stream.placeholder(dtype="float").rename("price")
volume_placeholder = Stream.placeholder(dtype="float").rename("volume")

feed = PushFeed([price_placeholder, volume_placeholder])

# On each tick:
data = feed.push({
    "price": current_price,
    "volume": current_volume,
})
```

## Reward Engineering Workflow

```mermaid
flowchart TD
    A[Choose Reward Scheme] --> B{Type?}
    B -->|Simple| C[SimpleProfit<br/>Net worth change]
    B -->|Risk-Adjusted| D[RiskAdjustedReturns<br/>Sharpe or Sortino]
    B -->|Position-Based| E[PBR<br/>Price change * position]
    B -->|Advanced| F[AdvancedPBR<br/>PBR + penalties + bonuses]
    B -->|Custom| G[Subclass RewardScheme]
    
    C --> H[Train Agent]
    D --> H
    E --> H
    F --> H
    G --> H
    
    H --> I{Agent Profitable?}
    I -->|No| J[Adjust reward params]
    J --> A
    I -->|Yes| K[Evaluate robustness]
```

## Component Customization

Every component can be replaced by subclassing the abstract base:

```python
from tensortrade.env.generic import RewardScheme

class CustomReward(RewardScheme):
    registered_name = "custom"

    def reward(self, env):
        portfolio = env.action_scheme.portfolio
        # Custom reward logic
        return portfolio.net_worth / portfolio.initial_net_worth - 1.0

    def reset(self):
        pass
```

The `registered_name` attribute enables lookup via the component registry and configuration via `TradingContext`.

---
## See Also
- [README](README.md) — Project overview and quick start
- [Architecture](architecture.md) — System design and components
- [State Management](state-management.md) — State lifecycle and data models
- [Development](development.md) — Development guide and best practices
