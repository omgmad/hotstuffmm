# 🔴 Hotstuff Market Maker Bot v4.0

An automated market making bot for [Hotstuff.trade](https://app.hotstuff.trade/join/hot) perpetual futures — BTC-PERP and ETH-PERP simultaneously.

> ⚠️ **WARNING**: This bot trades with real funds on mainnet. Always start with small sizes and test thoroughly.
> For educational purposes only. NFA, DYOR.

Built by [@0mgm4d](https://x.com/0mgm4d)

---

## 📋 Table of Contents

- [How It Works](#how-it-works)
- [Features](#features)
- [Performance](#performance)
- [Risk Warning](#risk-warning)
- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [Running the Bot](#running-the-bot)
- [Scaling Up](#scaling-up)
- [Monitoring](#monitoring)
- [Troubleshooting](#troubleshooting)
- [Emergency Stop](#emergency-stop)
- [Security](#security)

---

## ⚙️ How It Works

Market making is a strategy where the bot continuously places both a **buy (bid)** and **sell (ask)** order around the current market price. The bot earns a **maker rebate** on every filled order.

```
Market Price: $70,000

Bot places:
  BID @ $69,965  ← 5bps below mid
  ASK @ $70,035  ← 5bps above mid

When BID fills → bot is long $100
When ASK fills → bot is short $100 (position returns toward 0)
Rebate earned on each fill: 0.0025% × $100 = $0.0025
```

### Cycle Flow (every 5 seconds)

```
1. Fetch current mid price from Hotstuff API
2. Check current position via REST API
3. Calculate bid/ask prices with spread + inventory skew
4. Check open orders to prevent duplicates
5. Place BID if position is not at max long limit
6. Place ASK if position is not at max short limit
7. If position exceeds limit → auto reduce via market order (instant)
8. Wait 5 seconds → repeat
```

### Position Management

The bot tracks positions in real-time using WebSocket fills + REST API polling:

```
max_position_usd       = $400  (per symbol)
max_total_exposure_usd = $800  (BTC + ETH combined)

Position > max  →  auto reduce (market order, fills instantly)
Position = max  →  stop placing orders on that side
Position < max  →  place BID + ASK normally
```

### Maker Rebate Tiers

Hotstuff pays a rebate to limit order makers on every fill:

| Tier | 14-day Volume | Maker Rebate |
|------|--------------|--------------|
| Standard | $0 – $1M | -0.002% |
| VIP 1 | > $1M | -0.0025% |
| VIP 2 | > $5M | -0.003% |
| VIP 3 | > $20M | -0.0035% |
| VIP 4 | > $100M | -0.005% |

Negative fee = **you receive money** on every fill. The higher your volume, the higher your rebate rate.

---

## ✨ Features

- **Dual market making** — BTC-PERP + ETH-PERP running simultaneously
- **Maker rebate farming** — Earns rebates on every filled limit order
- **Real-time position tracking** — WebSocket fills + REST API fallback
- **Auto reduce** — Market order reduce when position exceeds limit (instant fill)
- **Duplicate order prevention** — Checks open orders before placing new ones
- **Inventory skew** — Adjusts quotes based on current position to stay balanced
- **Adaptive spread** — Widens spread automatically during high volatility
- **Circuit breaker** — Pauses bot on consecutive losses or daily loss limit hit
- **Auto instrument detection** — Fetches instrument IDs, tick sizes, lot sizes from API
- **Detailed logging** — Logs to both terminal and log file

---

## 💰 Performance

### Expected Daily Volume & Rebate

| Config | Fill Rate | Daily Volume | Rebate VIP1 (-0.0025%) |
|--------|----------|-------------|------------------------|
| $100 order, 2 symbols | 40% | ~$1.4M | ~$35/day |
| $200 order, 2 symbols | 40% | ~$2.8M | ~$70/day |
| $500 order, 2 symbols | 40% | ~$7M | ~$175/day |

### Key Insight: Position Balance = Profit

```
Position near $0  →  BID + ASK both active  →  Maximum volume  →  Maximum rebate
Position at max   →  Only one side active   →  Half volume     →  Half rebate
Position locked   →  No orders             →  Zero volume     →  Zero rebate
```

This is why the auto-reduce feature is critical — it keeps position balanced near zero so both sides keep filling and generating rebate income.

---

## ⚠️ Risk Warning

**Read this carefully before running the bot.**

### Leverage Risk
- Default Hotstuff leverage is **50x**
- At 50x: a **2% price move** against your position wipes **100% of your margin**
- **Strongly recommended: Set leverage to 11x–20x on Hotstuff UI before running**

### Capital Risk
- Bot uses real USDC on mainnet
- Losses can exceed rebate income during volatile markets
- A single liquidation can erase multiple days of rebate profit

### Recommended minimum capital by leverage

| Leverage | Min Capital | Notes |
|----------|------------|-------|
| 50x | $500+ | Very high risk |
| 20x | $300+ | Moderate risk |
| 11x | $200+ | Recommended for beginners |

### General Rules
- Always start with `order_size_usd: 100` and verify stability for 24+ hours
- Never run with capital you cannot afford to lose
- Monitor the bot regularly — do not leave unattended for days
- Past performance does not guarantee future results
- This is not financial advice (NFA, DYOR)

---

## 💻 Requirements

- Python 3.10+
- Ubuntu / Linux (recommended) or macOS
- [Hotstuff.trade](https://app.hotstuff.trade/join/hot) account with USDC deposited
- API wallet created on Hotstuff

---

## 🚀 Installation

### 1. Clone the repository

```bash
git clone https://github.com/omgmad/hotstuffmm.git
cd hotstuffmm
```

### 2. Create a virtual environment

```bash
# Ubuntu / macOS
python3 -m venv venv
source venv/bin/activate

# Windows
python -m venv venv
venv\Scripts\activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Create an API wallet on Hotstuff

1. Go to [Hotstuff](https://app.hotstuff.trade/join/hot) and connect your main wallet
2. Navigate to **Settings → API Wallets → Create API Wallet**
3. Save the **Private Key** securely
4. Note your **main wallet address** (the one you logged in with)

> The API wallet signs orders on behalf of your main account. Your main wallet's private key never touches the bot. The API wallet cannot withdraw funds.

### 5. Configure environment variables

```bash
cp .env.example .env
nano .env
```

Fill in your values:

```env
PRIVATE_KEY=0xYOUR_API_WALLET_PRIVATE_KEY
WALLET_ADDRESS=0xYOUR_MAIN_WALLET_ADDRESS
```

> ⚠️ **Important**: `WALLET_ADDRESS` must be your **main wallet** (the one you use to login to Hotstuff), NOT the API wallet address. Positions and fills are tracked under the main wallet.

### 6. Set leverage on Hotstuff UI

Before running the bot:
1. Go to [Hotstuff Trade](https://app.hotstuff.trade/join/hot)
2. Open BTC-PERP → click the leverage button → set to **11x** or **20x**
3. Repeat for ETH-PERP

### 7. Run the bot

```bash
python hotstuff_mm_bot.py
```

---

## ⚙️ Configuration

Edit the `CONFIG` section at the top of `hotstuff_mm_bot.py`:

```python
CONFIG = {
    # Trading parameters
    "order_size_usd":            100,   # USD size per order — start small
    "spread_bps":                5.0,   # Bid-ask spread in basis points
    "refresh_interval":          5,     # Seconds between order cycles

    # Risk management
    "max_position_usd":          200,   # Max position per symbol (USD)
    "max_total_exposure_usd":    400,   # Max combined BTC+ETH exposure (USD)
    "daily_loss_limit_usd":      20.0,  # Bot pauses if daily loss exceeds this
    "unrealized_loss_limit_usd": 5.0,   # Emergency close threshold
    "consecutive_loss_limit":    10,    # Pause after N consecutive losses
}
```

### Recommended settings by stage

| Stage | order_size | max_position | max_total | Leverage |
|-------|-----------|--------------|-----------|----------|
| Testing | $100 | $200 | $400 | 11x |
| Stage 1 | $200 | $400 | $800 | 11x–20x |
| Stage 2 | $500 | $800 | $1,600 | 20x |

---

## ▶️ Running the Bot

```bash
# Activate virtual environment
source venv/bin/activate

# Run with log output saved to file
python hotstuff_mm_bot.py 2>&1 | tee bot_log.txt
```

### Sample log output

```
2026-03-06 10:01:15 | INFO | ── BTC-PERP | Mid: $70,800 | Pos: $+0 | Spread: 5.0bps
2026-03-06 10:01:15 | INFO |    ✅ BID: 0.00141 @ 70,782
2026-03-06 10:01:15 | INFO |    ✅ ASK: 0.00141 @ 70,818
2026-03-06 10:01:20 | INFO |    BTC-PERP net_pos: $+200
2026-03-06 10:01:20 | INFO |    BID тавихгүй (pos: $200 / max: ±$200)
2026-03-06 10:01:25 | INFO |    📉 REDUCE MARKET SELL: 0.00070 (excess: $50)
2026-03-06 10:01:26 | INFO |    BTC-PERP net_pos: $+150
```

---

## 📈 Scaling Up

Only scale after confirming all of the following for 24+ hours:

- ✅ Position stays within limits consistently
- ✅ No unexpected duplicate orders
- ✅ Reduce logic fires correctly when position exceeds limit
- ✅ Daily rebate is being credited in Hotstuff UI
- ✅ No unexpected warnings or errors in logs

```bash
# Check for issues before scaling
grep "WARNING\|ERROR\|HALTED" bot_log.txt

# Verify position behavior
grep "net_pos\|REDUCE" bot_log.txt | tail -20
```

---

## 📊 Monitoring

```bash
# Follow live logs
tail -f bot_log.txt

# Check current position values
grep "net_pos" bot_log.txt | tail -10

# Check reduce activity
grep "REDUCE" bot_log.txt | tail -10

# Check for errors or warnings
grep "ERROR\|WARNING\|HALTED\|алдаа" bot_log.txt | tail -20

# Check volume (count fills)
grep "Fill:" bot_log.txt | wc -l
```

---

## ❌ Troubleshooting

### `ModuleNotFoundError: No module named 'websocket'`
```bash
pip install websocket-client
# If still fails, install directly to venv:
pip install websocket-client --target venv/lib/python3.12/site-packages/
```

### `positions: []` — API returns empty positions
Make sure `WALLET_ADDRESS` in `.env` is your **main wallet**, not the API wallet. Test:
```bash
python3 -c "
import requests
r = requests.post('https://api.hotstuff.trade/info',
    json={'method': 'positions', 'params': {'user': 'YOUR_MAIN_WALLET_HERE'}})
print(r.text[:300])
"
```

### Position keeps growing past the limit
- Verify `WALLET_ADDRESS` is correct main wallet
- Check reduce is triggering: `grep "REDUCE" bot_log.txt`
- Check for errors: `grep "place_market" bot_log.txt`

### Duplicate orders appearing (more than 1 BID or ASK)
- Cancel all open orders on Hotstuff UI → restart bot
- Check: `grep "аль хэдийн\|open_orders" bot_log.txt`

### `400 Bad Request` on orders
- Order size may be too small (minimum ~$10 notional)
- Verify your API wallet is registered on Hotstuff

### `daily_loss_limit_usd` triggered — bot paused
- Bot paused automatically to protect capital
- Review logs to understand the cause
- Restart when ready: `python hotstuff_mm_bot.py 2>&1 | tee bot_log.txt`

---

## 🚨 Emergency Stop

```bash
# Step 1 — immediately stop the bot
Ctrl+C

# Step 2 — cancel all open orders
# Go to Hotstuff UI → Open Orders tab → Close All

# Step 3 — close positions if needed
# Go to Hotstuff UI → Positions tab → Close All
```

---

## 🔒 Security

- **Never** commit your `.env` file to GitHub — it is listed in `.gitignore`
- Use an **API wallet** — your main wallet private key never touches the bot
- The API wallet **cannot withdraw funds** — it can only place/cancel orders
- Store your private key securely, preferably in a password manager
- Regularly verify your positions on Hotstuff UI match what the bot reports

---

## 📁 File Structure

```
hotstuffmm/
├── hotstuff_mm_bot.py    ← Main bot (v4.0)
├── .env.example          ← Environment variables template
├── requirements.txt      ← Python dependencies
├── README.md             ← This file
└── .gitignore            ← Prevents .env from being committed
```

---

## 📚 Resources

- [Hotstuff Exchange](https://app.hotstuff.trade/join/hot)
- [Hotstuff Documentation](https://docs.hotstuff.trade)
- [Hotstuff Fee Structure](https://docs.hotstuff.trade/hotstuff-docs/trading/fees.md)
- [Hotstuff Discord](https://discord.gg/tradehotstuff)

---

## 🎁 Support & Referral

If this bot has been helpful, consider signing up to Hotstuff using my referral link — it supports continued development:

👉 **[Join Hotstuff](https://app.hotstuff.trade/join/hot)**

---

## 👤 Author

Built by [@0mgm4d](https://x.com/0mgm4d) — follow on X for updates and new releases.

---

## 📄 License

MIT License — free to use, modify, and distribute.
