# Operations

The runtime is designed around one operational fact: an autonomous agent must
be safe while nobody is watching, and observable the moment anyone looks.

## Services

| Service | Role |
|---|---|
| `com.bliss.agent` | Trading daemon: RTH options session, evening futures/FX dry-run book, 19:00 ET daily report |
| `com.bliss.api` | FastAPI service: scan, journal, agent status, explain/ask |
| `com.bliss.web` | Next.js site |
| `com.bliss.sentinel` (staged) | Read-only journal tailer: per-event notifications, heartbeat staleness alarm, stuck-order detection |

All under `launchd` with keep-alive; scheduled tasks handle a 9:15 ET
pre-market readiness check (gateway port, services, log tail) and a 7:25 ET
evening report delivery. Logs and heartbeats are gitignored artifacts.

## Broker link

IB Gateway (paper account), API on localhost:4002, delayed market data by
default with model Greeks. The adapter is read-only for data; the agent's
paper broker uses a separate client id. Gateway auto-restart is scheduled at
03:00 ET — a designed gap the overnight scheduler treats as such (pause
pending entries into it, reconcile after). Real-time data subscription is a
hard prerequisite for any live graduation, and for paper trading at the
fastest horizon rungs.

## Risk rails (enforced in code, tested)

- **Fixed-fractional sizing:** base risk 2% of bankroll per trade; option
  premium capped at 25% of bankroll.
- **Global kill switch:** −3% daily loss halts everything, fleet-wide, all
  hours; day boundary fixed at 17:00 ET. Conviction-ladder tiers are sized so
  one full-risk loss at the top tier is a one-strike halt.
- **Concentration:** max open positions capped; correlated underlyings count
  as one position; same underlying across books/accounts nets to one.
- **Account-regulation awareness:** margin mode enforces pattern-day-trading
  budgets (3 day-trades / rolling 5 sessions under $25k); cash mode enforces a
  settled-funds ledger (T+1) that makes good-faith violations impossible by
  construction. Paper simulates the target account mode so records are
  live-replicable.
- **Instrument granularity gate:** any instrument whose minimum contract risk
  exceeds the per-trade budget is measured but never traded, and says so.

## Notifications

A standalone sentinel tails the append-only journal and pushes every money
event (placed / filled with chase vs plan / cancelled with reason / closed
with realized R and bankroll / halts) to macOS notifications and a Discord
webhook (env-configured, never committed). It is read-only by construction,
exactly-once across restarts, and market-calendar aware — silence during a
closed session is normal; silence during an open one is an alarm.

## The live-enable protocol

Live trading does not exist behind a flag. The live broker constructor raises
unconditionally; the path from paper to live, per market agent, is:

1. That agent's own market × horizon cell clears the honest ruler
   (calibrated, non-overfit, positive expectancy net of costs).
2. ≥ 1 month of paper at any active sizing tier, rails unbreached.
3. Real-time market data subscription for that market.
4. Startup reconciliation green (internal book vs broker positions).
5. A reviewed code change, with explicit human sign-off, arms that agent —
   and nothing else.

Until all five hold, the strongest instruction anyone can give the system is
declined by the code itself.
