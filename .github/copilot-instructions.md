# TradePipeline Copilot Instructions

## Architecture Overview

This is a real-time trading system using **event streaming** architecture with three primary components:

1. **NiFi** (data ingestion) → Consumes Massive WebSocket stock trades, publishes to Kafka topics (one per symbol)
2. **Flink** (stream processing) → Aggregates trades into 60-second risk metrics windows, publishes to `agg-{SYMBOL}` topics
3. **Python Agent** (trading logic) → Consumes aggregated metrics, ranks stocks, executes trades via Alpaca API

Data flows: `Massive WS → NiFi → Kafka (T.SYMBOL) → Flink → Kafka (agg-SYMBOL) → Agent → Alpaca`

## Development Workflow

### Building and Running

```bash
# Build Flink job (required before first run or after Scala changes)
make build-flink

# Start all services (builds Flink, then docker-compose up)
make run

# Submit Flink job to running cluster (for iterative development)
make submit-flink
```

**Critical**: Flink job manager runs on port **8083** (not default 8081). Access at http://localhost:8083

### Environment Setup

Create `.env` file with:
```
MASSIVE_TOKEN=<your_key>
APCA_KEY=<alpaca_key>
APCA_TOKEN=<alpaca_secret>
```

The agent uses Alpaca **paper trading** by default (`APCA_URL=https://paper-api.alpaca.markets`).

## Code Patterns

### Flink Stream Processing ([TradeJob.scala](api/flink/src/main/scala/org/neutron/TradeJob.scala))

- **Dynamic topic subscription**: Flink queries Kafka at startup for all topics and subscribes to them dynamically
- **Keyed by symbol**: Trades are keyed by symbol for parallel processing per stock
- **60-second tumbling windows**: Aggregations use `TumblingProcessingTimeWindows.of(Time.seconds(60))`
- **Risk metrics calculated**: VWAP, realized variance (RV), bipower variation (BV), jump ratio = (RV - BV) / RV
- **Custom output routing**: `KeyedSerializationSchema` routes each symbol's metrics to `agg-{SYMBOL}` topic

### Python Agent Pattern ([Agent.py](py/Agent.py))

- **Stateful aggregation**: Maintains running sums in `_nodestate` dict: `{symbol: {'current': data, 'count': n, 'agg': {...}}}`
- **Kafka polling**: Uses `poll()` in while loop, then batch processes messages before sleeping for `UPDATEFREQ` (60s)
- **Ranking system**: Combines multiple factors (VWAP change, jump ratio, trade size) via `.rank()` on standardized metrics
- **Top-N trading**: Selects top 10 ranked stocks, uses Dirichlet weighting scheme from [util.py](py/util.py)
- **Order cancellation**: Always cancels open orders for active assets before rebalancing

### Broker Abstraction ([broker.py](py/broker.py), [alpaca.py](py/alpaca.py))

- `Broker` is abstract base class defining interface: `hist_price()`, `market_open()`, `last()`, etc.
- `Alpaca` implements concrete broker with API credentials from environment variables
- Uses `threading` to wait for market open before executing strategies
- **Paper trading default**: Production code uses paper API; change `APCA_URL` for live trading
- **Note**: Alpaca's built-in Polygon integration has been deprecated; use Massive REST API directly for market data snapshots

### Portfolio Management ([portfolio.py](py/portfolio.py))

- **Asset tracking**: Each stock is an `Asset` object with side ('long'/'short'), qty, broker reference
- **Dynamic sync**: If `update_qty=True`, syncs share quantities with broker positions on updates
- **Risk metrics**: Calculates portfolio-level jump, BV, RV using `price_functions.py` helpers

## Dependencies

- **Scala/Flink**: Scala 2.11, Flink 1.11.2, Kafka connector, Gson for JSON
- **Python**: `alpaca_trade_api`, `kafka-python`, `empyrical` (risk metrics), `python-dotenv`
- **Infrastructure**: Kafka 5.5.1, Zookeeper 3.4.9, custom NiFi image with Massive WebSocket processor

## Service Communication

- Kafka brokers on `localhost:29092` (external) and `kafka:9092` (inter-container)
- Flink job manager UI: `localhost:8083`
- NiFi UI: `localhost:8088`
- Agent container linked to Kafka, reads from all `agg-*` topics
- Massive WebSocket: `wss://socket.massive.com/stocks` (real-time) or `wss://delayed.massive.com/stocks` (15-min delayed)

## Massive WebSocket Integration

The system uses [Massive WebSocket API](https://massive.com/docs/websocket/quickstart) for real-time trade data:
- **Authentication**: `{"action":"auth","params":"YOUR_API_KEY"}`
- **Subscription format**: `{"action":"subscribe","params":"T.SYMBOL1,T.SYMBOL2"}` where `T.` prefix indicates trade events
- **Supported channels**: Trade events (T), quotes (Q), aggregates (AM/A), and more - see [WebSocket docs](https://massive.com/docs/websocket/stocks/overview)
- Configure subscriptions via `MASSIVE_SUBSCRIPTIONS` environment variable in docker-compose.yml

## Common Issues

- **Missing `.env`**: Docker services fail without Massive/Alpaca credentials
- **Case sensitivity**: Agent looks for `agent.py` but Dockerfile references `Agent.py` (Python imports work, but CMD uses lowercase)
- **Kafka topic lag**: Agent won't receive data until Flink job is submitted AND NiFi is configured with Massive subscriptions
