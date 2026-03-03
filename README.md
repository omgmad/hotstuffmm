# 🔴 Hotstuff Market Maker Bot v3.0

An automated market making bot for [Hotstuff.trade](https://hotstuff.trade) perpetual futures exchange.

> ⚠️ **WARNING**: This bot trades with real funds. Always start with small sizes and test thoroughly.

---

## 📋 Table of Contents

- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [Running the Bot](#running-the-bot)
- [Utilities](#utilities)
- [Troubleshooting](#troubleshooting)
- [Emergency Stop](#emergency-stop)

---

## ✨ Features

- **Market Making** — Automatically places bid/ask orders around the mid price
- **Inventory Skew** — Adjusts quotes based on current position to stay balanced
- **Risk Management** — Daily loss limit and max drawdown circuit breakers
- **Auto Cancel/Replace** — Cancels and refreshes orders every cycle
- **Native Price Feed** — Uses Hotstuff's own mid price endpoint (no external feeds needed)
- **Auto Instrument Detection** — Fetches instrument IDs, tick sizes and lot sizes from the API
- **Logging** — Writes to both terminal and `mm_mainnet.log`

---

## 💻 Requirements

- Python 3.10+
- A [Hotstuff.trade](https://hotstuff.trade) account with USDC balance
- An agent wallet (with private key)

---

## 🚀 Installation

### 1. Clone the repository

```bash
git clone https://github.com/YOUR_USERNAME/hotstuff-mm-bot.git
cd hotstuff-mm-bot
```

### 2. Create a virtual environment

```bash
# Ubuntu / Mac
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

### 4. Generate an agent wallet

```bash
python generate_wallet.py
```

Save the printed **Address** and **PrivateKey** somewhere safe.

### 5. Register the agent on Hotstuff

1. Go to [hotstuff.trade](https://hotstuff.trade) and connect your main wallet
2. Navigate to **Settings → Agents → Add Agent**
3. Enter the **Address** from step 4 as the agent address
4. Confirm with MetaMask

> The agent wallet signs orders on behalf of your main account. Your main wallet's private key never touches the bot.

### 6. Create your .env file

```bash
cp .env.example .env
```

Edit `.env` and fill in your agent wallet details:

```env
PRIVATE_KEY=0x_your_agent_private_key
WALLET_ADDRESS=0x_your_agent_address
```

### 7. Verify instruments

```bash
python check_instruments.py
```

### 8. Run the bot

```bash
python hotstuff_mm_bot.py
```

Type `yes` when prompted to confirm.

---

## ⚙️ Configuration

Edit the `CONFIG` dictionary in `hotstuff_mm_bot.py`:

```python
CONFIG = {
    # ── Trading parameters ──
    "spread_bps":        7,      # Spread in bps (7 = 0.07%)
    "order_levels":      1,      # Orders per side (1 = 1 bid + 1 ask)
    "level_spacing_bps": 5,      # Spacing between levels in bps
    "order_size_usd":    10,     # Order size in USD (minimum $10)

    # ── Risk management ──
    "max_position_usd":      50,    # Max position per symbol in USD
    "daily_loss_limit_usd":  20.0,  # Daily loss limit in USD
    "max_drawdown_pct":       5.0,  # Max drawdown %

    # ── Speed ──
    "refresh_interval": 15,     # Seconds between cycles
}
```

### Recommended settings by experience level

| Level | order_levels | order_size_usd | max_position_usd |
|-------|-------------|----------------|-----------------|
| Beginner | 1 | 10 | 30 |
| Intermediate | 1–2 | 15 | 100 |
| Advanced | 2–3 | 20+ | 200+ |

### How it works

Each cycle the bot:
1. Cancels all existing orders
2. Fetches the current mid price from Hotstuff
3. Calculates bid/ask prices based on spread and inventory skew
4. Places new bid and ask orders
5. Waits for `refresh_interval` seconds

**Inventory skew** adjusts quotes when the bot has a large position — if long, it lowers the ask to encourage selling and reduce exposure.

---

## 🛠️ Utilities

| File | Purpose |
|------|---------|
| `generate_wallet.py` | Generate a new agent wallet |
| `check_instruments.py` | View instrument IDs, tick sizes, lot sizes |
| `cancel_all.py` | 🚨 Emergency cancel all open orders |

---

## ❌ Troubleshooting

### `invalid order signer`
- **Cause**: Signing mismatch or agent not registered
- **Fix**:
  1. Verify `PRIVATE_KEY` matches `WALLET_ADDRESS` in your `.env`
  2. Confirm the agent is registered on hotstuff.trade under Settings → Agents
  3. Ensure `eth-account==0.6.1` is installed: `pip show eth-account`

### `400 Bad Request`
- **Cause**: Invalid request format or order too small
- **Fix**: Ensure `order_size_usd >= 10` (minimum notional is $10)

### `ModuleNotFoundError`
- **Cause**: Dependencies not installed
- **Fix**: `pip install -r requirements.txt`

### `PRIVATE_KEY not set`
- **Cause**: `.env` file missing or empty
- **Fix**: `cp .env.example .env` then fill in your values

### Bot halted — `Daily loss limit reached`
- **Cause**: Bot hit the `daily_loss_limit_usd` threshold
- **Fix**: Restart the next day, or increase the limit in CONFIG if intentional

### High IM Utilization (>70%)
- **Cause**: Position too large relative to margin
- **Fix**:
  1. Set `order_levels` to `1`
  2. Reduce `max_position_usd`
  3. Add more USDC collateral on hotstuff.trade

---

## 🚨 Emergency Stop

To stop the bot and cancel all orders immediately:

```bash
# Step 1 — stop the bot
Ctrl+C

# Step 2 — cancel all open orders
python cancel_all.py
```

---

## 📊 Monitoring Logs

The bot writes to `mm_mainnet.log` while running:

```bash
# Follow live
tail -f mm_mainnet.log

# Search for errors
grep "ERROR\|WARNING\|HALTED" mm_mainnet.log
```

---

## 🔒 Security

- **Never** commit your `.env` file to GitHub — it is listed in `.gitignore`
- Use an **agent wallet** so your main wallet's private key never touches the bot
- Start with small sizes and increase only after confirming the bot works correctly

---

## 📚 Resources

- [Hotstuff Docs](https://docs.hotstuff.trade)
- [Hotstuff Discord](https://discord.gg/tradehotstuff)
- [Hotstuff Trade](https://hotstuff.trade)

---

## 📄 License

MIT License — free to use and modify.
