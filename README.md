# Partyproper Trading Workstation

Ubuntu 24.04 setup for Python algo-trading (Bybit perpetuals) with local LLM inference on a GTX 1060 3GB.

---

## What's Running Here

| Component | Purpose |
|---|---|
| `conda trading` env | Python 3.11 environment with all trading/ML libs |
| Ollama | Local LLM server — crypto analysis, code gen, reasoning |
| CUDA 12.1 / GTX 1060 3GB | GPU inference for PyTorch and Ollama |
| SenseiBot (`Documents/BOTS/4`) | ATR-band mean-reversion SHORT-only Bybit perps bot |
| TradingView MCP (`tradingview-mcp/`) | Claude Code bridge to TradingView Desktop via CDP |
| Redis | In-process pub/sub and state caching |
| JupyterLab | Notebook environment for analysis and backtesting |

---

## How It Works

### Python Environment

Everything runs inside the `trading` conda environment managed by Miniforge. This keeps trading libraries isolated from system Python.

Interpreter: `/home/partyproper/miniforge3/envs/trading/bin/python`

Key libraries installed:
- **Exchange**: `pybit` (Bybit SDK), `ccxt`, `python-binance`, `alpaca-trade-api`, `yfinance`
- **Technical analysis**: `ta-lib` (C lib), `ta`, `vectorbt`, `backtrader`
- **Data / ML**: `pandas`, `numpy`, `scipy`, `statsmodels`, `torch 2.5.1+cu121`, `optuna`, `numba`
- **Dev**: `jupyterlab`, `fastapi`, `uvicorn`, `loguru`, `pydantic`, `httpx`, `websockets`, `redis-py`

### Ollama (Local LLMs)

Ollama runs as a systemd service on `http://127.0.0.1:11434`. Models live in `/opt/ollama/models` (symlinked as `~/ollama-models`).

| Model | Size | Use |
|---|---|---|
| `trading-analyst:latest` | 1.1 GB | Crypto trading analysis, Bybit bot help |
| `qwen2.5-coder:3b` | 1.9 GB | Python code generation and debugging |
| `deepseek-r1:1.5b` | 1.1 GB | Reasoning and analysis |

**GPU constraint**: GTX 1060 3GB only fits one model in VRAM at a time. Running two simultaneously falls back to CPU RAM. `OLLAMA_NUM_PARALLEL=1` is set for this reason.

### SenseiBot

Located at `Documents/BOTS/4/`. Full documentation in `Documents/BOTS/4/README.md`.

Short version:
- Trades top Bybit USDT perpetuals on 5-minute candles
- SHORT-only, 2x leverage, 5% stop-loss
- Runs Bayesian optimisation (Optuna, 1000 trials) at startup and every 7 days
- Half-Kelly position sizing based on rolling backtest win rate
- Regime classifier (5-indicator voting) adapts parameters to market conditions
- Dashboard at `http://localhost:8181`
- Saves all trades, optimisation runs, and regime logs to `live_logs/sensei.db` (SQLite)

### TradingView MCP

Located at `tradingview-mcp/`. Bridges Claude Code to the TradingView Desktop app via Chrome DevTools Protocol (CDP). Lets you ask Claude to analyse your live chart, write Pine Script, manage indicators, and navigate the UI — all without leaving the terminal.

Architecture: `Claude Code <-> MCP Server (stdio) <-> CDP (localhost:9222) <-> TradingView Desktop`

---

## Manual Setup

### 1. Install Miniforge

```bash
curl -L https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh -o miniforge.sh
bash miniforge.sh -b -p ~/miniforge3
~/miniforge3/bin/conda init bash
source ~/.bashrc
```

### 2. Create the trading environment

```bash
conda create -n trading python=3.11 -y
conda activate trading
```

### 3. Install mamba (faster package resolution)

```bash
conda install -n trading mamba -c conda-forge -y
```

### 4. Install core packages via mamba

```bash
mamba install -n trading -c conda-forge \
  numpy pandas scipy scikit-learn statsmodels \
  jupyterlab fastapi uvicorn pydantic httpx websockets \
  sqlalchemy redis-py loguru psutil python-dotenv \
  optuna numba pyarrow -y
```

### 5. Install trading/exchange packages via pip

```bash
conda run -n trading pip install \
  pybit ccxt python-binance alpaca-trade-api yfinance \
  ta vectorbt backtrader structlog websocket-client
```

### 6. Install PyTorch with CUDA 12.1

```bash
conda run -n trading pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121
```

### 7. Install TA-Lib C library + Python wrapper

```bash
sudo apt-get install -y libta-lib-dev
conda run -n trading pip install ta-lib
```

### 8. Install Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh
sudo systemctl enable --now ollama
```

Pull models (stay under ~2.5 GB each for full GPU inference, use Q4_K_M quants):

```bash
ollama pull qwen2.5-coder:3b
ollama pull deepseek-r1:1.5b
# trading-analyst is a custom model — see ollama-configs/trading-analyst.modelfile
ollama create trading-analyst -f ~/ollama-configs/trading-analyst.modelfile
```

### 9. Install Redis

```bash
sudo apt-get install -y redis-server
sudo systemctl enable --now redis
```

### 10. CUDA / GPU setup

The NVIDIA driver (535+) and CUDA toolkit 12.6 should be installed via the Ubuntu driver manager or:

```bash
sudo apt-get install -y nvidia-driver-535 nvidia-cuda-toolkit
```

Enable persistence mode on boot:

```bash
sudo systemctl enable nvidia-perf  # if the service unit exists, otherwise:
sudo nvidia-smi -pm 1
```

Verify CUDA is working:

```bash
conda run -n trading python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"
```

### 11. Performance tuning (optional but recommended)

```bash
# Reduce swappiness
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# CPU performance governor
sudo apt-get install -y cpufrequtils
echo 'GOVERNOR="performance"' | sudo tee /etc/default/cpufrequtils
sudo systemctl restart cpufrequtils

# Raise file descriptor limit (for many WebSocket connections)
echo '* soft nofile 1048576' | sudo tee -a /etc/security/limits.conf
echo '* hard nofile 1048576' | sudo tee -a /etc/security/limits.conf
```

### 12. TradingView MCP

```bash
cd ~/tradingview-mcp
npm install
```

Then add the MCP server to Claude Code:

```bash
claude mcp add tradingview -- node /home/partyproper/tradingview-mcp/dist/server.js
```

Start TradingView Desktop with the CDP debug port enabled:

```bash
tradingview --remote-debugging-port=9222
```

### 13. SenseiBot first run

```bash
cd ~/Documents/BOTS/4
conda run -n trading pip install -r requirements.txt
conda run -n trading python main.py
# Enter your Bybit API key and secret when prompted (saved to .env, chmod 600)
```

---

## Letting Claude Code Set Everything Up

Instead of following the steps above manually, you can describe what you need and let Claude Code handle it. Open a terminal and run:

```bash
claude
```

Then paste or type something like:

```
Set up this machine from scratch as a Python algo-trading workstation.
Follow the instructions in ~/README.md. Install Miniforge to ~/miniforge3,
create a conda env called 'trading' with Python 3.11, install all the
packages listed in the manual setup section, install Ollama and pull the
three models, install Redis, verify CUDA is working, and apply the
performance tuning steps. Tell me when it's done and flag anything that
needs my input (e.g. API keys, driver installation that needs a reboot).
```

Claude Code will read this file, run the commands in sequence, report progress, and stop to ask you before anything that needs elevated permissions or credentials.

**Useful follow-up prompts:**

```
# Verify everything installed correctly
Check that the trading conda env has torch with CUDA, pybit, optuna, and
ta-lib installed. Run a quick import test for each.

# Set up SenseiBot only
cd into ~/Documents/BOTS/4, install its requirements into the trading env,
and explain what I need to provide before I can run it.

# Set up TradingView MCP only
Install and configure the TradingView MCP from ~/tradingview-mcp so I can
use it with Claude Code. Tell me how to launch TradingView with CDP enabled.

# Reinstall a broken package
The ta-lib Python wrapper is failing to import. Diagnose and fix it.
```

---

## Daily Usage

```bash
# Start SenseiBot
cd ~/Documents/BOTS/4 && conda run -n trading python main.py

# JupyterLab for analysis
conda run -n trading jupyter lab --no-browser --port=8888

# Query a local LLM
ollama run trading-analyst
ollama run qwen2.5-coder:3b

# Monitor GPU
nvtop

# Check services
sudo systemctl status ollama redis nvidia-perf

# Query trade history
sqlite3 ~/Documents/BOTS/4/live_logs/sensei.db \
  "SELECT ts_utc, symbol, side, fill_price, pnl_net_10x FROM trades ORDER BY id DESC LIMIT 20;"
```

---

## File Layout

```
~/
  Documents/BOTS/4/       # SenseiBot — ATR-band mean-reversion perps bot
  tradingview-mcp/        # Claude Code <-> TradingView Desktop bridge
  ollama-configs/         # Ollama modelfiles (e.g. trading-analyst.modelfile)
  ollama-models -> /opt/ollama/models   # symlink to model storage
  CLAUDE.md               # Claude Code project instructions
  README.md               # this file
```

---

## Notes

- Never commit `.env` files — Bybit API keys are stored there with `chmod 600`
- Only one Ollama model fits in the GTX 1060's 3 GB VRAM at a time
- New Ollama models: stay under ~2.5 GB, use Q4_K_M quantisation
- SenseiBot places real orders — there is no paper trading mode
- First optimisation run takes ~10 minutes per coin (1000 Optuna trials)
