# Hotstuff MM — Market Maker Bot

An automated market making bot for [Hotstuff.trade](https://app.hotstuff.trade/join/hot) perpetual futures.  
Earns **maker rebates** by continuously quoting tight bid/ask spreads on BTC-PERP and ETH-PERP.

[![Python](https://img.shields.io/badge/Python-3.10%2B-3776ab?logo=python&logoColor=white)](https://python.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-22c55e)](LICENSE)
[![Exchange](https://img.shields.io/badge/Exchange-Hotstuff.trade-f97316)](https://app.hotstuff.trade/join/hot)
[![X](https://img.shields.io/badge/X-@0mgm4d-000000?logo=x)](https://x.com/0mgm4d)

> Built by [@0mgm4d](https://x.com/0mgm4d) — follow on X for updates.

---

## ⚠️ Disclaimer

**Use at your own risk.** This software trades with real funds on mainnet.  
The author is not responsible for any financial losses.  
Start with minimum capital, read the code, and understand what it does before increasing sizes.  
For educational purposes only. NFA, DYOR.

---

## 🎁 Get Started on Hotstuff

New to Hotstuff? Sign up using the referral link below:

👉 **[Join Hotstuff.trade](https://app.hotstuff.trade/join/hot)**

---

## Features

- **Maker rebate farming** — post-only limit orders earn −0.0025% rebates
- **Adaptive spread** — dynamically adjusts width using rolling volatility (std dev)
- **Position limits** — hard per-symbol and total portfolio exposure caps
- **Auto-reduce** — places reduce-only orders when position exceeds the limit
- **Circuit breaker** — pauses quoting after N consecutive losses
- **Unrealized loss guard** — emergency closes all positions at configurable drawdown
- **WebSocket fills** — real-time position sync via WS subscription
- **Fill deduplication** — prevents double-counting across REST and WebSocket
- **Dual monitor** — terminal TUI (curses) + web dashboard, single file, zero extra deps

---

## Visual Overview

<table>
<tr>
<td width="50%" align="center">

**Spread Mechanism**<br>
<img src="gif1_spread.gif" width="100%">

</td>
<td width="50%" align="center">

**Maker Rebate Tiers**<br>
<img src="gif2_rebate.gif" width="100%">

</td>
</tr>
<tr>
<td width="50%" align="center">

**Position Balance & Auto-Reduce**<br>
<img src="gif3_position.gif" width="100%">

</td>
<td width="50%" align="center">

**Leverage & Risk**<br>
<img src="gif4_leverage.gif" width="100%">

</td>
</tr>
</table>

---

## How It Works

### Quoting

Every cycle (default 5 s), the bot:

1. Fetches the current mid price from Hotstuff
2. Calculates a volatility-adjusted spread
3. Cancels all existing quotes
4. Places **1 BID** below mid and **1 ASK** above mid per symbol

```
◄────── spread ──────►
BID          MID          ASK
$69,930    $70,000    $70,070    (5 bps each side)
```

When filled: earned the spread + maker rebate.

### Position Management

```
Max position per symbol : $200
Max total exposure       : $400  (BTC + ETH combined)

pos < max   →  quote both sides normally
pos → max   →  stop quoting that side
pos >> max  →  auto-reduce: place a closing order
```

### Risk Layers

| Layer | Trigger | Action |
|---|---|---|
| Daily loss limit | Loss > $20 | Halt for the day |
| Max drawdown | PnL drops 5% from peak | Halt |
| Consecutive losses | 10 losses in a row | Pause 5 min (circuit breaker) |
| Unrealized loss | Open loss > $5 | Emergency close all positions |

### Rebate Tiers (14-day rolling volume)

| Volume | Tier | Rebate |
|---|---|---|
| < $1M | Standard | −0.0020% |
| ≥ $1M | VIP 1 | −0.0025% |
| ≥ $5M | VIP 2 | −0.0030% |
| ≥ $20M | VIP 3 | −0.0035% |
| ≥ $100M | VIP 4 | −0.0050% |

---

## Installation

### Requirements

- Python **3.10** or newer
- A [Hotstuff.trade](https://app.hotstuff.trade/join/hot) account with an **Agent Key**

### 1 — Clone the repository

```bash
git clone https://github.com/omgmad/hotstuffmm.git
cd hotstuffmm
```

### 2 — Create a virtual environment

```bash
python3 -m venv venv

# macOS / Linux
source venv/bin/activate

# Windows
venv\Scripts\activate
```

### 3 — Install dependencies

```bash
pip install -r requirements.txt
```

### 4 — Configure

```bash
cp .env.example .env
```

Edit `.env`:

```env
PRIVATE_KEY=your_private_key_here
WALLET_ADDRESS=0xYourAgentWalletAddress
```

> **Where to get an Agent Key**  
> Log in to [hotstuff.trade](https://app.hotstuff.trade/join/hot) → Settings → Agents → Add Agent.  
> An Agent Key signs orders on behalf of your main wallet. Your main wallet's private key never touches the bot.

### 5 — Run

```bash
python hotstuff_mm_bot.py
```

Type `yes` at the confirmation prompt. The bot begins quoting immediately.

---

## Monitor

Open a second terminal while the bot runs:

```bash
python3 monitor.py
```

Select a mode:

```
  ╔══════════════════════════════════════╗
  ║   HOTSTUFF MM · MONITOR              ║
  ╠══════════════════════════════════════╣
  ║  [1] Terminal  — curses TUI          ║
  ║  [2] Browser   — web dashboard       ║
  ╚══════════════════════════════════════╝
  Select mode [1/2]:
```

### Mode 1 — Terminal TUI

Full-screen live dashboard. Works over SSH. No browser needed.

```
┌─ HOTSTUFF MM · TERMINAL MONITOR ────────────────── [LIVE] 14:23:07 ─┐
│ 24H VOL │ 12H VOL │  7D VOL  │ ALL VOL  │ NET PNL  │ REBATE │  FR  │
│  $3.8M  │  $1.9M  │  $16.2M  │  $84.1M  │ +$0.0091 │$0.0112 │  96% │
├──────────────────┬───────────────────────────────────────────────────│
│ POSITIONS        │  CUMULATIVE PNL  ▂▃▄▅▆▇█▇▆▅▄▅▆▇                  │
│ BTC-PERP  LONG   │  VOLUME TREND    ▁▁▂▃▄▅▆▇█▇▆▅                    │
│ ETH-PERP  SHORT  │  HOUR HEATMAP    ·░▒▓█▓▒░·                        │
├──────────────────┤                                                    │
│ REBATE TIER      ├───────────────────────────────────────────────────│
│ Standard  ░░░░░  │ RECENT FILLS              │ LOG                   │
│ VIP 1     ░░░░   │ 14:23 BTC BUY  $69,990    │ [OK] cycle #1421      │
└──────────────────┴───────────────────────────────────────────────────┘
  [q] Quit   [r] Refresh   [↑↓] Scroll fills
```

### Mode 2 — Web Dashboard

Opens `http://localhost:8888`. Dark/light mode toggle included.

Panels: KPI strip · 24h volume chart · live positions · rebate tier · streaks · hourly heatmap · cumulative PnL · Sharpe / max drawdown / win rate · fill history table.

### Direct launch

```bash
# Terminal TUI with wallet
python monitor.py terminal 0xYourWallet

# Browser dashboard
python monitor.py browser 0xYourWallet
# Using .env
source .env && python monitor.py terminal $WALLET_ADDRESS
```

> The monitor requires **no extra dependencies** — Python standard library only.

---

## Configuration

All settings live in the `CONFIG` dict at the top of `hotstuff_mm_bot.py`.

```python
CONFIG = {
    "symbols": ["BTC-PERP", "ETH-PERP"],

    # Spread
    "spread_bps":     5.0,   # Base spread (5 bps = 0.05%)
    "spread_min_bps": 1,
    "spread_max_bps": 5,

    # Sizing
    "order_size_usd": 100,

    # Position limits
    "max_position_usd":          200,
    "max_total_exposure_usd":    400,

    # Risk stop conditions
    "daily_loss_limit_usd":       20.0,
    "max_drawdown_pct":            5.0,
    "unrealized_loss_limit_usd":   5.0,

    # Circuit breaker
    "consecutive_loss_limit":     10,
    "circuit_breaker_pause_sec":  300,

    # Timing
    "refresh_interval": 5,   # seconds per cycle
}
```

### Conservative starting config (recommended for first run)

```python
"order_size_usd":               50,
"max_position_usd":            100,
"max_total_exposure_usd":      200,
"daily_loss_limit_usd":         10.0,
"consecutive_loss_limit":        5,
"unrealized_loss_limit_usd":     3.0,
```

---

## Project Structure

```
hotstuffmm/
├── hotstuff_mm_bot.py    # Main bot: quoting, risk, EIP-712 signing
├── monitor.py            # Monitor: terminal TUI + web dashboard (stdlib only)
├── requirements.txt      # Python dependencies
├── .env.example          # Environment variable template
├── .gitignore
├── Dockerfile
├── docker-compose.yml
├── LICENSE
├── README.md
└── docs/
    ├── gif1_spread.gif
    ├── gif2_rebate.gif
    ├── gif3_position.gif
    └── gif4_leverage.gif
```

---

## Security Best Practices

- **Never commit `.env`** — it's in `.gitignore` and contains your private key
- **Use an Agent Key**, not your main wallet private key
- Run on a VPS with a firewall (UFW / iptables)
- Monitor the log file: `tail -f hotstuff_mm.log`
- Set conservative risk limits when starting out

---

## Troubleshooting

| Problem | Solution |
|---|---|
| `ValueError: Set PRIVATE_KEY in .env` | Run `cp .env.example .env` and edit it |
| `ImportError: No module named 'eth_account'` | Run `pip install -r requirements.txt` |
| Bot shows 0 fills | Check `WALLET_ADDRESS` is your **agent** wallet, not main |
| Orders not filling | Spread may be too wide; check market depth on Hotstuff |
| `Circuit breaker` message | Normal — bot resumes automatically after 5 minutes |
| WebSocket keeps reconnecting | Expected on poor connections — bot reconnects automatically |

---

## Expected Performance

With default config ($100 orders, 5 s cycles, BTC + ETH):

| Metric | Estimate |
|---|---|
| Cycles / day | ~17,000 |
| Volume / day | $500K–$3M |
| Rebate (Standard) | ~$1–7 / day |
| Rebate (VIP 1) | ~$1.25–9 / day |

> Fill rate depends on spread tightness and market conditions.

---

## Support & Referral

If this bot has been helpful, consider signing up to Hotstuff using the referral link — it helps support continued development:

👉 **[Join Hotstuff.trade](https://app.hotstuff.trade/join/hot)**

---

## Links

- 🌐 [Hotstuff.trade](https://app.hotstuff.trade/join/hot)
- 📖 [Hotstuff Docs](https://docs.hotstuff.trade)
- 💬 [Hotstuff Discord](https://discord.gg/tradehotstuff)
- 🐦 [@0mgm4d on X](https://x.com/0mgm4d)
- 💻 [GitHub](https://github.com/omgmad/hotstuffmm)

---

## Contributing

Pull requests welcome.

1. Fork this repo
2. `git checkout -b feat/your-feature`
3. Commit your changes
4. Open a pull request

---

## License

[MIT](LICENSE)
