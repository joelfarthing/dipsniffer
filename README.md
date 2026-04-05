# 🐊 DipSniffer  (AKA YOLOBot) — Kraken Swing Trading Bot

Auto-executing swing trader that monitors 17 coins on Kraken. 
Concentrates full balance into one asset at a time based on RSI + Bollinger Band signals, with 3 layers of Gemini Flash sentiment filters, a composite strength scoring system, and an ATR-based dynamic trailing stop.

**Script:** `kraken-swing-bot.py`
**State:** `~/.config/kraken/swing-bot-state.json`
**Log:** `~/.config/kraken/swing-bot.log`

---

## The Strategy

| Parameter | Value |
|---|---|
| **BUY when** | RSI(14) < 30 AND price ≤ lower Bollinger Band(20, 2σ) |
| **SELL when** | RSI(14) > 70 OR (price ≥ upper BB AND bb_width > 3%) OR stop-loss hit |
| **Stop-loss** | ATR dynamic trailing (3.0× ATR at entry → 1.5× ATR at ≥10% gain, 8% fallback) |
| **Vetoes** | <1% Sell Walls (3× imbalance), BTC Crash Guard (RSI < 35 + BB < 0.15) |
| **Boosts** | RSI Bullish Divergence, Funding Rate Squeeze (Rate < -0.01%), Relative Strength vs BTC/ETH |
| **Candles** | 1-hour OHLC data |
| **Poll interval** | Every 5 minutes |
| **Assets** | 17 coins across 3 tiers (see below) |
| **Min trade** | $5 USD (Kraken minimum) |

### The 3 Intelligence Layers
YOLOBot uses deterministic math for its baseline targets, but gates its actions through three intelligence filters to prevent buying into crashes and to squeeze more juice out of winners.

1. **Layer 1: Deterministic Crash Guard (Math-driven)**
   - **Blocks all buys** if BTC's RSI < 35 AND its Bollinger position < 0.15.
   - Prevents YOLOBot from catching falling knives during macro/geopolitical market dumps where everything looks "oversold".
2. **Layer 2: Gemini Buy Veto (LLM-driven)**
   - When a buy signal fires, YOLOBot asks Gemini Flash to quickly read recent news about the highest-ranked candidate it just picked.
   - Outputs `SAFE` (proceed) or `RISK` (block entry).
   - Functions strictly as a fundamental safety interlock to catch hacks, regulatory crackdowns, or sudden market-wide shocks that math can't see.
3. **Layer 3: Gemini Sell Extension (LLM-driven)**
   - When a sell signal fires (e.g. RSI > 70), YOLOBot asks Gemini Flash if there's an ongoing bullish catalyst.
   - Outputs `SELL` (take profit) or `HOLD` (extend trade, trailing stop stays active).
   - **Never overrides stop-loss.** Stop-losses always fire instantly.

### The YOLO Hunt
When the market is bleeding and YOLOBot is idle in cash, it goes on the offensive. 
1. **Conditions:** No existing position + No math-based buy signals + Idle for ≥ 6 hours + Alternative.me Fear & Greed Index ≤ 40 (Fear).
2. **The Hunt:** Instead of waiting for standard buy signals, YOLOBot ranks all assets by a composite strength score (RSI, Bollinger Band positioning, funding rates, block volume, and relative strength vs BTC/ETH) to mechanically identify the single strongest coin resisting the dump.
3. **The Veto:** The top candidate is then structurally vetted by the Layer 2 Gemini Buy Veto (checks for news of hacks/scams/black swans) before execution.

### Additional Features
- **Stale Position Eject**: If a trade is held for >48 hours with <1.5% profit and hasn't made a new high in 12 hours, YOLOBot mathematically identifies stronger candidates. If one passes a fresh Gemini veto, the capital is immediately rotated out of the stale asset to maximize opportunity cost.
- **SQLite Data Log Engine**: Every market cycle, snapshot, and trade decision metadata is logged to `~/.config/kraken/market_history.db` for offline backtesting and performance attribution.
- **Order Book Veto**: Scans the Kraken order book ±1% for sell walls. Buys are vetoed if sell volume > 3× buy volume in the near-term range.
- **Funding Rate Overlay**: Adds a +25 composite strength bonus for coins in active short squeezes (negative funding rates).

---

## Operations

### Quick Commands

```bash
# Setup: We recommend asking your LLM Agent (like Antigravity) to read `agent_setup.md` to handle dependencies and authorization dynamically!

# Run once (check signals, no loop)
python3 kraken-swing-bot.py --status

# Start in background natively with Dashboard
./start-dipsniffer.sh

# Stop (find PID and kill)
pkill -f kraken-swing-bot
```

### tmux (Recommended)

```bash
# Start YOLOBot in a named tmux session
tmux new -s yolobot "./start-dipsniffer.sh"

# Detach (leave YOLOBot running, return to terminal): Ctrl+B, then D

# Re-attach (check on YOLOBot)
tmux attach -t yolobot

# Quick peek without attaching
tmux capture-pane -t yolobot -p | tail -10

# Kill the session entirely
tmux kill-session -t yolobot
```

### Logs

```bash
# Watch live
tail -f ~/.config/kraken/swing-bot.log

# Last 20 entries
tail -20 ~/.config/kraken/swing-bot.log

# See all trades only
grep "BOUGHT\|SOLD" ~/.config/kraken/swing-bot.log
```

## Watchlist (17 Coins)

| Tier | Assets | Why |
|---|---|---|
| **Tier 1** — Large cap | BTC, ETH | Stable, reliable signals |
| **Tier 2** — Mid cap | SOL, AVAX, LINK, DOT, ATOM, NEAR, SUI, INJ | Good volatility, decent swings |
| **Tier 3** — High vol | DOGE, FET, RENDER, PEPE, HYPE, ONDO, TAO | Frequent RSI extremes, wilder swings, momentum narratives |

*(Edit `PAIRS` in the script to add/remove coins)*

## Run Modes
```bash
--status     # Show signals only, no trades (skips Gemini logic for speed)
--dry-run    # Show what it WOULD do, no trades (runs full logic flow)
--loop       # Run continuously every 5 min
(no flags)   # Run once and exit
```

---
*Note: YOLOBot talks to Kraken through the Python `ccxt` client and reads `~/.config/kraken/config.toml` API credentials unless `KRAKEN_API_KEY` and `KRAKEN_API_SECRET` are set in the environment.*
