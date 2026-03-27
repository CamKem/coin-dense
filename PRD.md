# CoinDense — Autonomous Trading Bot

## Product Requirements Document

**Version:** 1.0
**Date:** 2026-03-27
**Status:** Draft

---

## 1. Overview

CoinDense is a personal-use autonomous trading bot built on Laravel that leverages AI agents (via the Laravel Agents SDK) to research, formulate, execute, and continuously adapt trading strategies across multiple asset classes. It combines hard-coded technical analysis algorithms with LLM-driven reasoning, real-time data ingestion, and a conversational interface for collaborative strategy refinement.

The system is personal-first but designed with user-scoped data (`user_id` on all records) to allow future SaaS conversion.

---

## 2. Architecture Summary

### Tech Stack

| Layer | Technology |
|---|---|
| **Backend** | Laravel 12, PHP 8.3+ |
| **Frontend** | React (Inertia.js), Tailwind CSS, shadcn/ui |
| **Auth** | Laravel Starter Kit (built-in auth) |
| **AI** | Laravel Agents SDK, multi-provider (Anthropic Claude + OpenAI GPT) |
| **Queue/Jobs** | Laravel Queues (Redis driver), Horizon |
| **WebSockets** | Laravel Reverb for real-time UI updates |
| **Database** | PostgreSQL (primary), Redis (cache/queues) |
| **Deployment** | Laravel Forge on DigitalOcean VPS |

### Key Patterns

- **Broker Manager Pattern** — A `BrokerManager` service with swappable drivers (Binance, Alpaca, etc.) following Laravel's Manager pattern
- **Strategy Engine** — Coordinated multi-strategy execution with shared risk budgets
- **Campaign System** — User-defined trading campaigns with allocated funds, timeframes, and strategy constraints
- **Agent Loop** — Queue-driven continuous cycle: gather data → analyze → decide → execute → monitor → adapt

---

## 3. Broker Integrations

### Primary Brokers

| Broker | Asset Classes | Purpose |
|---|---|---|
| **Binance** | Crypto spot, crypto futures | Primary crypto trading |
| **Alpaca** | US stocks, ETFs | Stock trading (commission-free API) |

### Broker Service Architecture

```
BrokerManager (Laravel Manager Pattern)
├── BinanceDriver (crypto spot + futures)
├── AlpacaDriver (stocks + ETFs)
└── PaperDriver (simulated, for dev/testing)
```

Each driver implements a unified `BrokerInterface`:

- `getAccount(): AccountInfo`
- `getBalances(): Collection`
- `getPositions(): Collection`
- `placeLimitOrder(...): Order`
- `placeMarketOrder(...): Order`
- `cancelOrder(string $orderId): bool`
- `getOrderStatus(string $orderId): Order`
- `getCandles(string $symbol, string $interval, ...): Collection`
- `getOrderBook(string $symbol): OrderBook`
- `getTicker(string $symbol): Ticker`
- `streamPrices(string $symbol, Closure $callback): void` (WebSocket)

A `PaperDriver` wraps any real driver to simulate execution against live market data without placing real orders — used during development and testing.

---

## 4. Trading Algorithms

Each algorithm is encapsulated as an `Indicator` class implementing `IndicatorInterface`:

### Technical Indicators

| Algorithm | Class | Description |
|---|---|---|
| **Fibonacci Retracements** | `FibonacciRetracement` | Key support/resistance levels based on Fibonacci ratios (23.6%, 38.2%, 50%, 61.8%, 78.6%) |
| **Chart Patterns** | `ChartPatternDetector` | Head & shoulders, double tops/bottoms, triangles, flags, wedges |
| **Elliott Wave** | `ElliottWaveAnalyzer` | 5-wave impulse + 3-wave corrective structure identification |
| **Fair Value Gaps (FVG)** | `FairValueGap` | Three-candle imbalance zones indicating potential price magnets |
| **Support & Resistance** | `SupportResistance` | Horizontal levels from historical price pivots and volume clusters |
| **Volume Analysis** | `VolumeAnalyzer` | Volume profile, VWAP, OBV, volume divergences |
| **Supply & Demand** | `SupplyDemandZone` | Institutional order flow zones based on rally-base-rally / drop-base-drop |
| **Break of Structure** | `BreakOfStructure` | Higher-high/lower-low violations signaling trend shifts |
| **Change of Character** | `ChangeOfCharacter` | First break of structure against the prevailing trend |

### Indicator Interface

```php
interface IndicatorInterface
{
    public function analyze(CandleCollection $candles, array $params = []): IndicatorResult;
    public function getSignal(): Signal; // BULLISH, BEARISH, NEUTRAL
    public function getConfidence(): float; // 0.0 - 1.0
    public function getKeyLevels(): array; // Price levels of interest
}
```

The **Strategy Engine** aggregates signals from multiple indicators, weights them, and produces a composite `TradeSignal` that the agent uses alongside its own LLM-driven analysis.

---

## 5. AI Agent System

### Agent Architecture (Laravel Agents SDK)

The system uses a multi-provider setup (Anthropic Claude + OpenAI) configurable per agent task.

#### Agent Roles

| Agent | Provider | Responsibility |
|---|---|---|
| **StrategyAgent** | Claude (primary) | Core decision-making: formulates and adjusts trading strategies by combining indicator signals with market context |
| **ResearchAgent** | Claude/OpenAI | Searches the web for news, events, social sentiment, economic data. Feeds findings to StrategyAgent |
| **ExecutionAgent** | Claude | Translates strategy decisions into precise order parameters (entry, stop-loss, take-profit, position sizing) and executes via BrokerManager |
| **MonitorAgent** | Claude | Watches open positions, triggers alerts, detects when conditions change enough to warrant strategy reassessment |

#### Agent Tools

Each agent is equipped with tools registered via the Laravel Agents SDK:

- **Market Tools** — `GetCandles`, `GetTicker`, `GetOrderBook`, `GetPositions`, `GetBalance`
- **Analysis Tools** — `RunIndicator`, `RunAllIndicators`, `GetStrategySignal`
- **Execution Tools** — `PlaceOrder`, `CancelOrder`, `ModifyOrder`, `ClosePosition`
- **Research Tools** — `WebSearch`, `ReadNews`, `GetEconomicCalendar`, `GetSocialSentiment`
- **Campaign Tools** — `GetCampaignStatus`, `UpdateCampaignStrategy`, `LogDecision`

#### Agent Loop (Queue-Driven)

```
┌─────────────────────────────────────────────────────┐
│                   CAMPAIGN LOOP                      │
│                                                      │
│  1. ResearchAgent gathers latest data/news           │
│  2. Indicators run against fresh candle data         │
│  3. StrategyAgent evaluates all signals + research   │
│  4. StrategyAgent decides: HOLD / ENTER / EXIT /     │
│     ADJUST                                           │
│  5. ExecutionAgent places/modifies orders             │
│  6. MonitorAgent watches positions                    │
│  7. Loop repeats (interval based on timeframe)       │
│                                                      │
│  At any point, user can converse with StrategyAgent  │
│  to discuss, question, or override decisions         │
└─────────────────────────────────────────────────────┘
```

---

## 6. Campaign System

A **Campaign** is a user-defined trading mission with explicit constraints:

### Campaign Model

| Field | Type | Description |
|---|---|---|
| `id` | ulid | Primary key |
| `user_id` | foreignId | Owner |
| `name` | string | e.g., "BTC Swing Q1" |
| `broker` | string | Driver name (binance, alpaca) |
| `symbols` | json | Tradeable symbols for this campaign |
| `budget` | decimal | Maximum allocated funds |
| `spent` | decimal | Current funds deployed |
| `timeframe` | string | Trading timeframe (scalp, day, swing, position) |
| `duration_start` | datetime | Campaign start |
| `duration_end` | datetime | Campaign end (nullable for indefinite) |
| `strategy_config` | json | Initial strategy parameters, indicator weights |
| `risk_config` | json | Max drawdown %, per-trade risk %, max positions |
| `status` | enum | draft, active, paused, completed, stopped |
| `performance` | json | Running P&L, win rate, Sharpe ratio, etc. |

### Campaign Lifecycle

1. **Draft** — User configures campaign via chat or UI form
2. **Active** — Agent loop running, executing trades within constraints
3. **Paused** — Loop suspended, positions maintained
4. **Completed** — Duration ended or budget exhausted, positions closed
5. **Stopped** — User manually stopped, positions closed

Multiple campaigns can run concurrently. The **StrategyAgent** is aware of all active campaigns and coordinates between them (shared risk budget awareness, avoiding conflicting positions).

---

## 7. Conversational Interface

### Agent Chat

The main UI is a chat interface (React + Inertia) where the user converses with the StrategyAgent:

**Capabilities:**

- Discuss current strategy reasoning and ask "why" about any decision
- Review portfolio state, open positions, P&L
- Propose strategy adjustments ("What if we tighten the stop-loss to 2%?")
- Create new campaigns via natural language ("Start a BTC scalping campaign with $500 for the next 2 weeks")
- Get explanations of indicator signals ("Why is Elliott Wave showing bearish?")
- Override agent decisions ("Close the ETH position now")
- Review trade history and performance analytics

**Chat is contextual** — the agent has access to all campaign data, positions, indicators, and trade history within the conversation.

### Real-Time Updates

Via Laravel Reverb WebSockets:

- Trade execution notifications
- Strategy adjustment alerts
- Price alerts and stop-loss triggers
- Campaign status changes
- Indicator signal changes

---

## 8. Data Sources

| Source | Type | Integration |
|---|---|---|
| **Exchange APIs** | Market data (candles, orderbook, tickers) | Direct via broker drivers |
| **News APIs** | Financial news headlines + articles | Agent web search tool + dedicated news API |
| **Social Sentiment** | Twitter/X, Reddit, crypto-specific forums | Agent web search + sentiment scoring |
| **Economic Calendar** | GDP, CPI, interest rates, employment data | Public economic data APIs |
| **On-Chain Data** | Whale movements, exchange flows (crypto) | Blockchain analytics APIs |

---

## 9. Risk Management

### Hard Limits (Code-Enforced)

- **Per-trade risk** — Maximum % of campaign budget per single trade (default: 2%)
- **Max drawdown** — Campaign pauses automatically if drawdown exceeds threshold (default: 10%)
- **Max open positions** — Per campaign and global limits
- **Daily loss limit** — Trading halts for the day if daily losses exceed threshold
- **Position sizing** — Kelly criterion or fixed-fraction, calculated automatically

### Soft Limits (Agent-Managed)

- Correlation awareness (don't over-expose to correlated assets)
- Volatility-adjusted position sizing
- Time-based exposure limits (reduce size before major events)

---

## 10. Notifications

| Channel | Events |
|---|---|
| **In-App** | All events — trades, alerts, strategy changes, campaign updates |
| **Telegram** | Trade executions, stop-loss triggers, campaign status changes, daily summaries |

Notifications are dispatched via Laravel's notification system with `TelegramChannel` and `DatabaseChannel`.

---

## 11. Dashboard Pages

| Page | Description |
|---|---|
| **Dashboard** | Portfolio overview, total P&L, active campaigns summary, recent trades |
| **Agent Chat** | Conversational interface with the StrategyAgent |
| **Campaigns** | List, create, edit, pause/resume campaigns |
| **Campaign Detail** | Deep dive into a single campaign — chart, positions, trades, performance metrics |
| **Trade History** | Filterable log of all trades with P&L |
| **Indicators** | Live indicator dashboard — run indicators on any symbol/timeframe |
| **Settings** | Broker API keys, notification config, risk defaults, LLM provider config |

---

## 12. Non-Functional Requirements

- **Latency** — Order placement within 500ms of decision
- **Uptime** — Queue workers monitored via Horizon, auto-restart on failure
- **Data Retention** — All trades, decisions, and agent reasoning logged permanently
- **Security** — Broker API keys encrypted at rest (Laravel encrypted casts), minimal withdrawal permissions on API keys
- **Audit Trail** — Every agent decision logged with reasoning, indicator signals, and market context at time of decision

---

## 13. Development Phases

### Phase 1 — Foundation
Laravel project setup (React starter kit), database schema, broker manager pattern, Binance driver, paper trading driver.

### Phase 2 — Technical Analysis Engine
All 9 indicator classes, candle data pipeline, indicator aggregation and signal generation.

### Phase 3 — AI Agent System
Laravel Agents SDK integration, multi-provider config, agent tools, strategy agent, research agent.

### Phase 4 — Campaign & Execution
Campaign CRUD, execution agent, order management, position tracking, risk enforcement.

### Phase 5 — Conversational Interface
Agent chat UI, real-time updates via Reverb, contextual agent conversations.

### Phase 6 — Monitoring & Notifications
Monitor agent, Telegram integration, in-app notifications, alerting system.

### Phase 7 — Dashboard & Analytics
Dashboard pages, charts, performance metrics, trade history, indicator visualization.

### Phase 8 — Alpaca Integration & Polish
Alpaca driver for stocks, cross-broker campaign support, final testing, deployment.
