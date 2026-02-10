# AlphaGPT-Routine

AlphaGPT-Routine is an automated quantitative strategy discovery and execution tool supporting **CN** (A-share), **HK**, and **US** stock markets. It utilizes a Transformer-based model (AlphaGPT) to discover and optimize mathematical alpha formulas using Polish Notation, verifies them against historical data, and provides daily trading signals via DingTalk, Telegram, or webhook.

## 🚀 Key Features

- **Alpha Discovery**: Automated discovery of trading signals using a powerful Transformer model.
- **Multi-Market**: Supports Chinese A-shares (AkShare), Hong Kong (AkShare HK), and US stocks (yfinance).
- **Dynamic Encoding**: Convert human-readable formulas (e.g., `ADD(RET, RET5)`) directly into model tokens and vice versa.
- **Robust Backtesting**: Strict out-of-sample verification with realistic transaction costs and slippage modeling.
- **Multi-Channel Notifications**: DingTalk, Telegram Bot, and generic webhook (n8n/Zapier/Make).
- **Cloud Native**: Fully configurable via environment variables and ready for GitHub Actions automation.

## 🛠️ Configuration (Environment Variables)

The program is highly flexible and can be controlled entirely through environment variables.

### Core Settings

| Variable | Description | Default |
| :--- | :--- | :--- |
| `MARKET` | Market to use: `CN`, `HK`, or `US` | `CN` |
| `INDEX_CODE` | Stock or Index code (CN: `000001`, HK: `00700`, US: `SPY`) | Per market |
| `CODE_FORMULA` | Combined override in `CODE:FORMULA` format | - |
| `BEST_FORMULA` | Specific formula to use (skips training) | - |
| `START_DATE` | Data start date (YYYYMMDD) | `20220101` |
| `END_DATE` | Data end date (YYYYMMDD) | `20270101` |
| `TRAIN_ITERATIONS` | Number of training epochs | `100` |
| `BATCH_SIZE` | Model batch size | `1024` |
| `MAX_SEQ_LEN` | Maximum complexity of the formula | `10` |
| `COST_RATE` | Transaction cost (CN: `0.0004`, HK: `0.0025`, US: `0.0`) | Per market |
| `ONLY_LONG` | Long-only mode (CN: `True`, HK/US: `False`) | Per market |

### Notification Settings

| Variable | Description | Default |
| :--- | :--- | :--- |
| `DINGTALK_WEBHOOK` | DingTalk Bot Webhook URL | - |
| `DINGTALK_SECRET` | DingTalk Bot Signature Secret | - |
| `TELEGRAM_BOT_TOKEN` | Telegram Bot API token | - |
| `TELEGRAM_CHAT_ID` | Telegram chat/group ID to send to | - |
| `WEBHOOK_URL` | Generic webhook URL (for n8n, Zapier, etc.) | - |
| `WEBHOOK_SECRET` | HMAC-SHA256 secret for webhook signature | - |

Multiple notification channels can be active simultaneously.

## 📦 GitHub Actions Setup

You can run this project for free using GitHub Actions daily routine.

### 1. Configure Secrets & Variables
Go to your GitHub repository -> **Settings** -> **Secrets and variables** -> **Actions**.

- **Secrets** (for sensitive data):
  - `DINGTALK_WEBHOOK`, `DINGTALK_SECRET`: DingTalk bot credentials.
  - `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`: Telegram bot credentials.
  - `WEBHOOK_URL`, `WEBHOOK_SECRET`: Generic webhook credentials.
  - `CODE_FORMULA`: (Optional) If you want to lock a specific code and formula.

- **Variables** (for general config):
  - `MARKET`: `CN`, `HK`, or `US`.
  - `INDEX_CODE`: Target stock/index.
  - `TRAIN_ITERATIONS`: Set to `100` for discovery or `1` for daily reporting.
  - `ONLY_LONG`: Set to `True` for A-shares.

### 2. Per-Market Cron Schedules
The default cron (`13 8 * * *`) runs after A-share market close. For other markets, adjust:
- **CN**: `13 8 * * *` (08:13 UTC = 16:13 CST)
- **HK**: `30 8 * * *` (08:30 UTC = 16:30 HKT)
- **US**: `0 21 * * *` (21:00 UTC = 16:00 EST)

### 3. Manual Trigger
You can manually trigger the workflow from the **Actions** tab by selecting "A-Stock AlphaGPT routine" and clicking **Run workflow**.

## 💻 Local Setup

```bash
# Install dependencies
pip install torch pandas numpy akshare matplotlib pyarrow tqdm requests yfinance

# Run CN market (default)
python times_astock.py

# Run HK market
MARKET=HK INDEX_CODE=00700 python times_astock.py

# Run US market
MARKET=US INDEX_CODE=SPY python times_astock.py

# With Telegram notifications
export TELEGRAM_BOT_TOKEN="123456:ABC..."
export TELEGRAM_CHAT_ID="-100123456789"
python times_astock.py

# With generic webhook (for n8n)
export WEBHOOK_URL="https://your-n8n.example.com/webhook/alphagpt-signal"
python times_astock.py
```

## 🔗 n8n Integration

An importable n8n workflow is included at `n8n_telegram_workflow.json`. It routes webhook signals to Telegram:

1. Import `n8n_telegram_workflow.json` into your n8n instance
2. Configure a Telegram Bot credential in n8n
3. Set `TELEGRAM_CHAT_ID` as an n8n environment variable
4. Activate the workflow and copy the webhook URL
5. Set `WEBHOOK_URL=<n8n-webhook-url>` in your environment

## 📝 Disclaimer
This software is for educational purposes only. The quantitative strategies generated are based on historical data and do not guarantee future returns. Trading in the stock market involves significant risk.
