# CLAUDE.md

## Project Overview

AlphaGPT-Routine is an automated quantitative strategy discovery tool for the A-share (Chinese stock market). It uses a Transformer-based neural network to discover trading alpha formulas expressed in Polish Notation via reinforcement learning, then backtests and reports daily signals.

Single-file Python application: `times_astock.py` (~1,150 lines).

## Tech Stack

- **Language**: Python 3.10+
- **Deep Learning**: PyTorch (with automatic CUDA/CPU detection)
- **Data**: Pandas, NumPy, AkShare (market data API), Parquet (caching)
- **Notifications**: DingTalk API
- **CI/CD**: GitHub Actions (daily scheduled runs)
- **License**: GPL-3.0

## Setup & Running

```bash
# Install dependencies (no requirements.txt — install directly)
pip install torch pandas numpy akshare matplotlib pyarrow tqdm requests

# Run with defaults
python times_astock.py

# Run with configuration
INDEX_CODE="600001" TRAIN_ITERATIONS=100 python times_astock.py
```

All configuration is via environment variables (see README.md for full list). Key ones:
`INDEX_CODE`, `START_DATE`, `END_DATE`, `BATCH_SIZE`, `TRAIN_ITERATIONS`, `MAX_SEQ_LEN`, `COST_RATE`, `HOLD_PERIOD`, `LAST_NDAYS`, `FORCE_TRAIN`, `ONLY_LONG`, `BEST_FORMULA`, `CODE_FORMULA`, `DINGTALK_WEBHOOK`, `DINGTALK_SECRET`.

## Architecture

Three main classes in `times_astock.py`:

1. **AlphaGPT** (nn.Module) — Transformer encoder with actor/critic heads. 18-token vocabulary (6 features + 12 operators), 64D embeddings, 4 attention heads, 2 layers.
2. **DataEngine** — Fetches OHLCV + margin data from AkShare, computes 6 normalized features (RET, RET5, VOL_CHG, V_RET, TREND, F_BUY_F_REPLAY), 80/20 train/test split.
3. **DeepQuantMiner** — Reinforcement learning loop: generates token sequences via policy gradient, evaluates formulas, backtests with Sharpe-like reward, tracks best formula.

Execution flow: `DataEngine.load()` → `DeepQuantMiner.train()` → `final_reality_check()` → `show_latest_positions()` (+ optional DingTalk notification).

## Code Conventions

- 4-space indentation
- `UPPER_CASE` for global constants, `snake_case` for functions/variables, `CamelCase` for classes
- `_prefix` for private/internal functions
- Mixed English/Chinese comments
- Heavy use of `@torch.jit.script` for performance-critical tensor ops

## Testing

No automated test suite. Validation is via out-of-sample backtesting built into the execution flow (`final_reality_check`). CI runs daily via GitHub Actions (`.github/workflows/astock_analysis.yml`).

## Important Notes

- `margin_balance/` directory contains cached Parquet data files (130+ MB) — auto-committed by CI
- No `.gitignore`, no linter config, no `requirements.txt` or `pyproject.toml`
- The project is a single Python file; avoid splitting without good reason
- Educational/research tool — not financial advice
