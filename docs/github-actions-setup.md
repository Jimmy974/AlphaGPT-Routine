# GitHub Actions Setup Guide

This guide walks you through setting up AlphaGPT-Routine to run automatically via GitHub Actions.

## Overview

The workflow (`.github/workflows/astock_analysis.yml`) runs daily after market close, fetches latest data, evaluates your strategy formulas, and sends notifications to your configured channels.

---

## Step 1: Configure Secrets & Variables

Go to your GitHub repository → **Settings** → **Secrets and variables** → **Actions**.

### Secrets (sensitive values — encrypted, never shown in logs)

| Secret | Value | When to set |
|---|---|---|
| `CODE_FORMULA` | Multi-stock formulas (see below) | For multi-stock mode |
| `DINGTALK_WEBHOOK` | `https://oapi.dingtalk.com/robot/send?access_token=...` | If using DingTalk |
| `DINGTALK_SECRET` | `SECxxxxx` | If using DingTalk |
| `TELEGRAM_BOT_TOKEN` | `123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11` | If using Telegram |
| `TELEGRAM_CHAT_ID` | `-100123456789` (group) or `123456789` (personal) | If using Telegram |
| `WEBHOOK_URL` | `https://your-n8n.example.com/webhook/alphagpt-signal` | If using n8n/webhook |
| `WEBHOOK_SECRET` | Any string for HMAC-SHA256 signature verification | Optional, for webhook security |

### Variables (non-sensitive config — visible in logs)

| Variable | Value | Description |
|---|---|---|
| `MARKET` | `CN`, `HK`, or `US` | Which stock market (default: `CN`) |
| `INDEX_CODE` | `600901`, `00700`, `SPY`, etc. | Single stock code (ignored if `CODE_FORMULA` is set) |
| `TRAIN_ITERATIONS` | `100` | Discovery mode. Set to `1` for daily reporting with known formula |
| `ONLY_LONG` | `True` or `False` | Long-only mode (default: per market) |
| `LAST_NDAYS` | `50` | Number of recent trading days to display |
| `BEST_FORMULA` | e.g. `NEG(RET)` | Use specific formula, skip training |

---

## Step 2: Single Stock vs Multi-Stock

### Single Stock Mode

Set `INDEX_CODE` as a Variable and optionally `BEST_FORMULA`:

```
INDEX_CODE = 600901
BEST_FORMULA = NEG(ADD(RET,VOL_CHG))
```

The workflow runs once for that stock, trains (or uses the given formula), and sends notifications.

### Multi-Stock Mode (CODE_FORMULA)

Set `CODE_FORMULA` as a **Secret** with one `CODE:FORMULA` pair per line:

```
600001:NEG(RET)
600901:ADD(RET,RET5)
002466:SUB(TREND,VOL_CHG)
000300:MUL(RET,SIGN(VOL_CHG))
```

How it works:
- The app loops through **each line** sequentially
- Each stock gets its own data fetch, evaluation, and notification
- Training is skipped (formulas are provided)
- `INDEX_CODE` and `BEST_FORMULA` variables are **ignored** when `CODE_FORMULA` is set

**Important**: Use the **Secrets** tab (not Variables) for `CODE_FORMULA` since multiline values work more reliably in secrets.

---

## Step 3: Notification Setup

Multiple channels can be active at the same time. Each fires independently — a failure in one doesn't block others.

### DingTalk

1. Create a DingTalk bot in your group chat
2. Copy the webhook URL and signature secret
3. Set `DINGTALK_WEBHOOK` and `DINGTALK_SECRET` as Secrets

### Telegram (Direct)

1. Create a bot via [@BotFather](https://t.me/BotFather) on Telegram
2. Get the bot token (e.g. `123456:ABC-DEF...`)
3. Get your chat ID:
   - For personal chat: send a message to the bot, then visit `https://api.telegram.org/bot<TOKEN>/getUpdates`
   - For group: add bot to group, send a message, check `getUpdates` for the negative group ID
4. Set `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` as Secrets

### Webhook (for n8n / Zapier / Make)

1. Set up a webhook trigger in your automation tool
2. Copy the webhook URL
3. Set `WEBHOOK_URL` as a Secret
4. Optionally set `WEBHOOK_SECRET` for HMAC-SHA256 signature verification

The webhook receives a structured JSON payload:

```json
{
  "event": "strategy_signal",
  "version": "1.0",
  "data": {
    "source": "AlphaGPT",
    "index_code": "600001",
    "generated_at": "2026-02-10T08:13:00",
    "formula": "NEG(ADD(RET,VOL_CHG))",
    "trades": [
      {
        "date": "2026-02-07",
        "position": 1,
        "return_pct": 0.0235,
        "entry_price": "3.456",
        "exit_price": "3.537",
        "exit_date": "02-14",
        "exit_offset": 5
      }
    ],
    "summary": {
      "valid_days": 42,
      "investment_count": 15,
      "profit_count": 10,
      "win_rate": 0.6667,
      "simple_return": 0.1234,
      "compound_return": 0.1456
    }
  }
}
```

If `WEBHOOK_SECRET` is set, the request includes an `X-Signature-256` header with the HMAC-SHA256 hex digest.

See also: `n8n_telegram_workflow.json` in the repo root for a ready-to-import n8n workflow that routes webhook signals to Telegram.

---

## Step 4: Schedule (Cron)

The workflow runs daily via cron. The default schedule is set for **after A-share market close**:

```yaml
schedule:
  - cron: '13 8 * * *'   # 08:13 UTC = 16:13 CST (after CN close)
```

For other markets, edit the cron line in the workflow file:

| Market | Close Time | Suggested Cron | UTC Time |
|---|---|---|---|
| **CN** | 15:00 CST | `13 8 * * *` | 08:13 UTC |
| **HK** | 16:00 HKT | `30 8 * * *` | 08:30 UTC |
| **US** | 16:00 EST | `0 21 * * *` | 21:00 UTC |

The workflow also supports **manual trigger** via the **Actions** tab → **Run workflow** button.

---

## Step 5: Margin Data Cache (CN Only)

For the CN market, the workflow automatically commits updated margin balance data (Parquet files in `margin_balance/`) after each run. This step is **skipped for HK/US markets** since margin data is A-share specific.

---

## Quick Start Examples

### Example 1: Single CN stock with Telegram

Variables:
```
MARKET = CN
INDEX_CODE = 600901
TRAIN_ITERATIONS = 100
```

Secrets:
```
TELEGRAM_BOT_TOKEN = 123456:ABC-DEF...
TELEGRAM_CHAT_ID = -100123456789
```

### Example 2: Multiple CN stocks with known formulas + DingTalk

Secrets:
```
CODE_FORMULA:
600001:NEG(RET)
600901:ADD(RET,RET5)
002466:SUB(TREND,VOL_CHG)

DINGTALK_WEBHOOK = https://oapi.dingtalk.com/robot/send?access_token=...
DINGTALK_SECRET = SEC...
```

### Example 3: US market with webhook

Variables:
```
MARKET = US
INDEX_CODE = SPY
BEST_FORMULA = NEG(RET)
TRAIN_ITERATIONS = 1
```

Secrets:
```
WEBHOOK_URL = https://your-n8n.example.com/webhook/alphagpt-signal
```

---

## Troubleshooting

| Problem | Solution |
|---|---|
| No data retrieved | Check `INDEX_CODE` is valid for the `MARKET`. CN uses 6-digit codes, HK uses 5-digit, US uses tickers |
| Telegram not sending | Verify bot token and chat ID. Bot must be added to the group first |
| Workflow not triggering | Ensure the workflow file is on the default branch (main/master) |
| Margin data commit fails | Only happens on CN market. Check repository write permissions |
| Message truncated in Telegram | Telegram has a 4096-char limit. Reduce `LAST_NDAYS` if needed |
