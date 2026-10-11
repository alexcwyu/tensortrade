# TensorTrade: workflow and architecture

Updated 2026-10-12. Source baseline: `161164e65108a4dec792db4a7a735fadeb8549bd` on `feature/ayu_develop`. See [development commands](development.md) and [validation evidence](validation.md) for the tested environment and current limitations.

## Purpose and features

Models a trading task as a Gymnasium environment with data streams, action schemes, rewards, observations, wallets, and simulated order execution.

- Gymnasium-compatible trading environments
- Composable streams and data feeds
- Wallet and portfolio accounting
- Configurable action, observation, reward, and stop components
- Stochastic price processes and reinforcement-learning integrations

## Component architecture

Environment components separate the learning agent from execution and accounting rules. A data feed advances streams while action schemes translate a discrete or continuous choice into orders. Exchange services apply fills and wallet changes, and the reward scheme scores the resulting transition. This division makes it possible to alter the reward or observation model without silently changing portfolio accounting. Episode termination and truncation follow the Gymnasium interaction boundary.

```mermaid
flowchart LR
    C0["Streams and data feed"]
    C1["Trading environment"]
    C2["Action and reward schemes"]
    C3["Broker and simulated exchange"]
    C4["Wallet portfolio and ledger"]
    C0 --> C1 --> C2 --> C3 --> C4
```

The diagram shows responsibility boundaries in the main research or execution path. Concrete entry points and implementation files are linked below.

## Core types

The types below belong to the public or native boundary described by the linked implementation. Arrows describe the dependency or ownership relationship used in the documented workflow. Listed Python methods are present in the linked source; native array types represent the binding boundary.

```mermaid
classDiagram
    class TradingEnv {
        +components()
        +step()
        +reset()
    }
    class DataFeed {
        +compile()
        +run()
        +forward()
    }
    class Stream {
        +run()
        +forward()
        +has_next()
    }
    class Portfolio {
        +wallets()
        +exchanges()
        +ledger()
    }
    class Wallet {
        +locked_balance()
        +total_balance()
        +locked()
    }
    TradingEnv --> DataFeed : advances observations
    DataFeed *-- Stream : organizes dependencies
    Portfolio *-- Wallet : holds balances
    TradingEnv --> Portfolio : executes and scores actions
```

## End-to-end workflow

1. Build streams, exchange, wallets, and feed.
2. Reset the environment and warm observation history.
3. Translate agent action into trading instructions.
4. Execute orders and update wallet balances.
5. Compute reward and terminal state for the next step.

```mermaid
flowchart TD
    W0["Build streams, exchange, wallets, and feed"]
    W1["Reset the environment and warm observation history"]
    W2["Translate agent action into trading instructions"]
    W3["Execute orders and update wallet balances"]
    W4["Compute reward and terminal state for the next step"]
    W0 --> W1 --> W2 --> W3 --> W4
    W4 --> R{"Review evidence"}
    R -->|Revise configuration| W0
    R -->|Retain result| A["Store reproducible artifacts"]
```

## Interaction sequence

The sequence follows one unit of work. Results return across the same boundaries that accepted the configuration and data. External I/O and numerical execution have different failure modes and should retain separate evidence.

```mermaid
sequenceDiagram
    actor User as Researcher or operator
    participant C0 as Streams and data feed
    participant C1 as Trading environment
    participant C2 as Action and reward schemes
    participant C3 as Broker and simulated exchange
    participant C4 as Wallet portfolio and ledger
    User->>C0: Supply configuration and inputs
    C0->>C1: Reset the environment and warm observation history
    C1->>C2: Translate agent action into trading instructions
    C2->>C3: Execute orders and update wallet balances
    C3->>C4: Compute reward and terminal state for the next step
    C4-->>C3: Results and retained state
    C3-->>C2: Execution or evaluation evidence
    C2-->>C1: Computed outputs
    C1-->>C0: Research or runtime result
    C0-->>User: Inspect outputs and provenance
```

## Data and artifact lifecycle

```mermaid
flowchart LR
    Inputs["Configuration and source data"] --> Runtime["Validated runtime inputs"]
    Runtime --> Work["Action and reward schemes"]
    Work --> Outputs["Wallet portfolio and ledger"]
    Outputs --> Review["Metrics, logs, and retained artifacts"]
    Review --> Inputs
```

Inputs include explicit strategy or model settings, data selection, and execution assumptions. Retaining these with outputs makes a rerun reviewable. Temporary environments and caches remain development artifacts, while source-controlled configuration documents intended behavior.

## Execution and failure boundaries

TensorFlow and stochastic numerical dependencies constrain available Python versions. Ray RLlib integration tests require its additional environment and training dependencies. Broker adapters remain external integrations. The maintenance retains all existing tests and records missing integrations rather than interpreting collection success as training validation.

```mermaid
stateDiagram-v2
    [*] --> Configured
    Configured --> Prepared: Inputs and dependencies available
    Prepared --> Running: Start unit of work
    Running --> Completed: Retain result
    Running --> Failed: Record exception and inputs
    Failed --> Configured: Correct configuration or implementation
    Completed --> [*]
```

This state diagram describes the documented workflow, not a claim that these names are an implementation enum. Failure evidence should identify whether input preparation, external services, native compilation, or application execution failed.

## Development layout

Python package src/tensortrade and Hatchling packaging are preserved. uv manages runtime, documentation, and test dependencies. Native TensorFlow wheels remain third-party binary requirements.

## Source map

| Existing source | Responsibility evidence |
|---|---|
| [src/tensortrade/env/generic/environment.py](../src/tensortrade/env/generic/environment.py#L35) | Implementation and module/type documentation, beginning at line 35 |
| [src/tensortrade/env/default/__init__.py](../src/tensortrade/env/default/__init__.py#L17) | Implementation and module/type documentation, beginning at line 17 |
| [src/tensortrade/env/default/actions.py](../src/tensortrade/env/default/actions.py#L24) | Implementation and module/type documentation, beginning at line 24 |
| [src/tensortrade/env/default/rewards.py](../src/tensortrade/env/default/rewards.py#L12) | Implementation and module/type documentation, beginning at line 12 |
| [src/tensortrade/env/default/observers.py](../src/tensortrade/env/default/observers.py#L19) | Implementation and module/type documentation, beginning at line 19 |
| [src/tensortrade/feed/core/feed.py](../src/tensortrade/feed/core/feed.py#L8) | Implementation and module/type documentation, beginning at line 8 |
| [src/tensortrade/feed/core/base.py](../src/tensortrade/feed/core/base.py#L24) | Implementation and module/type documentation, beginning at line 24 |
| [src/tensortrade/oms/orders/broker.py](../src/tensortrade/oms/orders/broker.py#L23) | Implementation and module/type documentation, beginning at line 23 |
| [src/tensortrade/oms/orders/order.py](../src/tensortrade/oms/orders/order.py#L28) | Implementation and module/type documentation, beginning at line 28 |
| [src/tensortrade/oms/exchanges/exchange.py](../src/tensortrade/oms/exchanges/exchange.py#L23) | Implementation and module/type documentation, beginning at line 23 |
| [src/tensortrade/oms/wallets/portfolio.py](../src/tensortrade/oms/wallets/portfolio.py#L32) | Implementation and module/type documentation, beginning at line 32 |
| [src/tensortrade/oms/wallets/wallet.py](../src/tensortrade/oms/wallets/wallet.py#L37) | Implementation and module/type documentation, beginning at line 37 |
| [src/tensortrade/oms/wallets/ledger.py](../src/tensortrade/oms/wallets/ledger.py#L21) | Implementation and module/type documentation, beginning at line 21 |

The links target files present in this checkout. Follow their imports and adjacent tests when changing a boundary. The validation document distinguishes successful builds, complete test runs, inherited skips, and unresolved prerequisites.
