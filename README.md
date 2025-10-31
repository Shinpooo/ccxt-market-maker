**CCXT Market Maker**

Lightweight, modular market making framework built around CCXT. The system is split into three core components that communicate via a message bus (NATS):

- **Market Data**: Streams public order book data per exchange/symbol and publishes it.
- **Order Router**: Receives order intents, routes to exchanges, streams private fills/updates.
- **Strategy Engine**: Consumes market and private data, emits order intents via the router.

This repository currently includes an initial Market Data service prototype using `ccxt.pro` websockets and `nats` for pub/sub. Order Router and Strategy Engine are scaffolds to be implemented next.

**Repository Structure**

- `market_data/`: Exchange websockets and publishing logic.
- `orderrouter/`: Placeholder for order submission + fills streaming.
- `strategyengine/`: Placeholder for strategies and orchestration.
- `ccxt_wrapper/`: Placeholder for exchange abstractions/helpers.
- `main.py`: Entry point placeholder.
- `pyproject.toml`: Project metadata (Python >= 3.10.12).

**Runtime Architecture**

- **Bus**: NATS at `0.0.0.0:4222` (hard-coded in prototype).
- **Exchanges**: Bybit, MEXC, Binance via `ccxt.pro`.
- **Subjects**:
  - Subscriptions: `suiren.subscriptions.orderbook.{exchange}`
  - Heartbeats: `suiren.heartbeat.orderbook.{exchange}`
  - Order book stream: `suiren.orderbook.{exchange}.{symbol}`
- **Heartbeat**: A symbol subscription is closed if no heartbeat for 20s.

Payload examples (JSON):

- Subscription/Heartbeat request
  - `{ "symbol": "BTC/USDT" }`
- Published order book snapshot (abridged)
  - `{ "symbol": "BTC/USDT", "bids": [[price, size]], "asks": [[price, size]], ... }`

**Prerequisites**

- Python `>= 3.10.12`
- NATS server
- Dependencies: `ccxtpro` (for `ccxt.pro`), `nats-py`

Install dependencies using `pip`:

- `python -m venv .venv && source .venv/bin/activate`
- `pip install -U pip`
- `pip install ccxtpro nats-py`

Run a local NATS server (choose one):

- Docker: `docker run --rm -p 4222:4222 nats:2.10-alpine`
- Local binary: `nats-server -p 4222`

**Market Data Service**

Current prototype lives in `market_data/market_data.py` and:

- Connects to NATS at `0.0.0.0:4222`.
- Listens for subscription and heartbeat messages per exchange.
- Streams best bid/ask via `ccxt.pro` and publishes order book snapshots.
- Closes idle subscriptions without heartbeat after 20 seconds.

Start streaming for a symbol (example with Binance):

1) Publish a subscription request to `suiren.subscriptions.orderbook.binance` with body `{ "symbol": "BTC/USDT" }`.
2) Maintain a heartbeat to `suiren.heartbeat.orderbook.binance` with the same body every few seconds.
3) Subscribe to `suiren.orderbook.binance.BTC/USDT` to receive updates.

Minimal Python example to initiate a stream:

```
import asyncio, json, nats

async def main():
    nc = await nats.connect("0.0.0.0:4222")
    await nc.publish("suiren.subscriptions.orderbook.binance", json.dumps({"symbol": "BTC/USDT"}).encode())
    async def hb():
        while True:
            await asyncio.sleep(5)
            await nc.publish("suiren.heartbeat.orderbook.binance", json.dumps({"symbol": "BTC/USDT"}).encode())
    asyncio.create_task(hb())
    async def handler(msg):
        print("orderbook:", msg.subject, len(msg.data))
    await nc.subscribe("suiren.orderbook.binance.BTC/USDT", cb=handler)
    while True:
        await asyncio.sleep(1)

asyncio.run(main())
```

Note: The provided `main.py` is a placeholder; run your own orchestrator or import `MarketData` directly.

**Design Contracts**

- **Strategy → Router**: Strategies send idempotent order intents (price, size, side, time-in-force, tag/uuid). Router handles dedup, placement, and risk checks.
- **Router → Strategy**: Emits stateful private updates (ack, open, fill, cancel) and normalized error codes.
- **Market Data → Strategy/Router**: Stateless, high-frequency public data with clear schemas and no side effects.

Recommended message shapes (to refine as you build Router/Strategy):

- Order intent: `{ "id": "uuid", "symbol": "BTC/USDT", "side": "buy", "px": 50000.0, "qty": 0.01, "tif": "GTC" }`
- Private update: `{ "id": "uuid", "status": "filled", "filled_qty": 0.01, "avg_px": 49990.5, ... }`

**Configuration**

- Current prototype uses hard-coded NATS URL `0.0.0.0:4222` and exchange map in `market_data/market_data.py`.
- Next steps: promote these to environment variables (e.g., `NATS_URL`) and a config file for symbols, heartbeats, and logging.

**Development Notes**

- Add `ccxtpro`/`nats-py` to `pyproject.toml` when you finalize dependencies.
- Implement `orderrouter/` (REST/WebSocket auth, order placement, fills) and `strategyengine/` (core strategies, position/risk).
- Fix minor prototype issue: `main()` references `Suiren`; the class name is `MarketData`.
- Add graceful shutdown and reconnection logic for websockets and NATS.

**Cautions**

- Market making is risky; test with paper trading/sandbox before live.
- Keep API keys out of the repo. Use environment variables or secret managers.
- `ccxt.pro` is a paid package; ensure you have a valid license or provide a polling fallback with `ccxt` for dev.

**License**

- Add your preferred license file before distributing.
