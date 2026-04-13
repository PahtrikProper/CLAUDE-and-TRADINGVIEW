# TradingView + Claude Code

Claude Code with eyes and hands on your TradingView Desktop. Read live chart data, control the UI, develop Pine Scripts, scan symbols, practice in replay — all from the terminal by talking to Claude.

---

## How It Works

```
You (terminal)
     |
Claude Code  <-->  MCP Server (src/server.js)  <-->  CDP port 9222  <-->  TradingView Desktop
```

The bridge at `~/tradingview-mcp` is an MCP server that talks to TradingView via **Chrome DevTools Protocol** — the same debug interface used by VS Code and Chrome devtools. You enable it once with a launch flag. After that, Claude has 78 tools to read and control your chart.

**Everything stays local.** No TradingView servers are contacted by the bridge.

---

## Setup

### Option A — Let Claude do it (recommended)

Open a terminal, run `claude`, paste this:

```
Set up the TradingView MCP. The repo is already at ~/tradingview-mcp.
Run npm install, add the server to ~/.claude/.mcp.json, then restart
and verify with tv_health_check.
```

Claude reads `SETUP_GUIDE.md` automatically and follows it step by step.

### Option B — Manual (4 steps)

**1. Install**

```bash
cd ~/tradingview-mcp
npm install
```

**2. Add to MCP config** (`~/.claude/.mcp.json`)

```json
{
  "mcpServers": {
    "tradingview": {
      "command": "node",
      "args": ["/home/partyproper/tradingview-mcp/src/server.js"]
    }
  }
}
```

**3. Launch TradingView with the debug port**

```bash
~/tradingview-mcp/scripts/launch_tv_debug_linux.sh
```

Auto-detects your install, kills any running instance, relaunches with `--remote-debugging-port=9222`, and waits for CDP to be ready. If auto-detect fails:

```bash
/path/to/tradingview --remote-debugging-port=9222
```

**4. Restart Claude Code, then verify**

```
Use tv_health_check to verify TradingView is connected.
```

---

## Skills (pre-built workflows)

These are slash commands — type them in Claude Code to run a full workflow. No need to know which tools to call.

### `/chart-analysis`

Full technical analysis on any symbol. Sets up the chart, adds indicators, marks support/resistance, takes a screenshot, and gives you a bias.

```
/chart-analysis BTCUSDT 1h

/chart-analysis ES1! 5m — look at last week's price action

/chart-analysis — analyse whatever is on my chart right now
```

What it does: `chart_set_symbol` → `chart_set_timeframe` → add indicators → `data_get_ohlcv` → `draw_shape` for levels → `capture_screenshot` → written analysis with bias

---

### `/pine-develop`

Full Pine Script development loop. Write, inject, compile, fix errors, iterate.

```
/pine-develop — write a script that draws ATR bands around a 20-period EMA

/pine-develop — add an alert condition to my current script when price crosses the upper band

/pine-develop — the script is showing a "mismatched input" error on line 14, fix it
```

What it does: writes to `scripts/current.pine` → `pine_push.js` injects to TV editor → compiles → reads errors → fixes → loops until clean → `capture_screenshot` to verify

---

### `/strategy-report`

Comprehensive backtest report for whatever strategy is loaded on your chart.

```
/strategy-report

/strategy-report — focus on the drawdown and suggest improvements
```

What it does: `data_get_strategy_results` → `data_get_trades` → `data_get_equity` → `chart_get_state` → screenshots of chart + strategy tester → structured report with metrics, strengths, weaknesses, recommendations

---

### `/multi-symbol-scan`

Scan a list of symbols for setups or compare strategy performance across instruments.

```
/multi-symbol-scan BTCUSDT ETHUSDT SOLUSDT BNBUSDT — which has the strongest setup on 1h?

/multi-symbol-scan — scan my watchlist for RSI below 30 on the daily

/multi-symbol-scan ES1! NQ1! YM1! — compare strategy results across futures
```

What it does: iterates symbols via `batch_run` or `chart_set_symbol` loop → reads values/results per symbol → builds comparison table → screenshots top setups

---

### `/replay-practice`

Step through historical bars in TradingView replay mode, take trades, track P&L.

```
/replay-practice BTCUSDT 5m from 2025-01-15

/replay-practice — start from the last major swing high on this chart
```

What it does: `replay_start` at a date → `replay_step` bar by bar → `replay_trade` for entries/exits → `replay_status` for P&L → `replay_stop` when done → session summary

---

## Agent: `performance-analyst`

A dedicated subagent focused entirely on strategy performance. More thorough than `/strategy-report` — use when you want a deep independent analysis.

```
Use the performance-analyst agent on my current strategy.
```

Evaluates: profitability (net profit, profit factor, avg trade), consistency (win rate, consecutive losses, equity curve), risk (max drawdown, worst trade), and edge quality. Returns a structured report with specific, actionable recommendations.

---

## Just Talking to Claude

You don't need skills for everything. Just describe what you want:

**Reading the chart**
```
What's on my chart right now?

What are the current RSI and MACD values?

Read the price levels from my "NY Levels" indicator.

Read the session stats table from my "Profiler" indicator.

Take a screenshot and describe what you see.
```

**Controlling the chart**
```
Switch to BTCUSDT on the 15-minute chart.

Set up a 2x2 grid: ES1! 5m, NQ1! 5m, BTCUSDT 1h, ETHUSDT 1h.

Add the 200 EMA. Remove the MACD.

Zoom to last Tuesday.
```

**Drawings and alerts**
```
Draw a horizontal line at 24500.

Draw a trend line from last Monday's high to yesterday's open.

Set an alert when price crosses 24550.

List my active alerts. Delete the ones below 24000.

Clear all drawings.
```

**Placing orders on signal**
```
Read my "Entry Signals" indicator. If there's a SHORT label on the last
closed bar, place a Sell market order on BTCUSDT for 0.01 contracts
using pybit. API keys are in ~/.env.
```

```
Check the RSI on my chart. If it's above 70, close any open BTCUSDT position.
```

Claude reads the signal from TradingView, evaluates the condition, then calls a Python snippet with pybit to execute on Bybit. For this to work, have a small order script ready — ask Claude to write one:

```
Write a ~/quick_order.py using pybit that I can call to place market
orders and close positions on Bybit USDT perps. Load API keys from ~/.env.
```

**Polling on a schedule**
```
/loop 5m
Check my TradingView chart for a SHORT signal from "Entry Signals" on
the last closed 5m bar. If there's a new one, place a Sell order on
BTCUSDT for 0.01 contracts via ~/quick_order.py.
```

---

## CLI (`tv` command)

Every MCP tool is also a `tv` CLI command. JSON output — pipe with `jq`.

```bash
# Install globally (optional)
cd ~/tradingview-mcp && npm link
```

```bash
tv status                              # check connection
tv quote                               # current price + OHLC
tv symbol BTCUSDT                      # change symbol
tv timeframe 15                        # change timeframe
tv ohlcv --summary                     # compact price stats
tv screenshot -r chart                 # screenshot to screenshots/
tv pine compile                        # compile Pine Script
tv pane layout 2x2                     # set 4-chart grid
tv pane symbol 1 ES1!                  # set pane 1 symbol
```

**Streaming** (polls your live chart via CDP, stays local)

```bash
tv stream quote                        # price tick stream
tv stream bars                         # bar-by-bar updates
tv stream values                       # indicator value stream
tv stream lines --filter "NY Levels"   # price level stream
tv stream tables --filter Profiler     # table data stream
tv stream all                          # all panes simultaneously
```

---

## Tool Quick Reference

### Read the chart

| Tool | What you get |
|---|---|
| `chart_get_state` | Symbol, timeframe, all indicator names + IDs |
| `quote_get` | Latest price, OHLC, volume |
| `data_get_study_values` | RSI, MACD, BB, EMA current numeric values |
| `data_get_ohlcv` | Price bars — use `summary: true` for compact stats |

### Custom indicator output (Pine drawings)

These read what your indicators draw on the chart (`line.new`, `label.new`, `table.new`, `box.new`). Always pass `study_filter` to target one indicator by name.

| Tool | Reads |
|---|---|
| `data_get_pine_lines` | Horizontal price levels (support/resistance, session levels) |
| `data_get_pine_labels` | Text annotations + prices ("PDH 24550", "Bias Long ✓") |
| `data_get_pine_tables` | Data tables (session stats, analytics panels) |
| `data_get_pine_boxes` | Price zones as {high, low} pairs |

### Control the chart

| Tool | Does |
|---|---|
| `chart_set_symbol` | Change ticker |
| `chart_set_timeframe` | 1, 5, 15, 60, D, W, M |
| `chart_manage_indicator` | Add/remove — use **full names**: "Relative Strength Index" not "RSI" |
| `pane_set_layout` | Grid: `s`, `2h`, `2v`, `2x2`, `4`, `6`, `8` |
| `pane_set_symbol` | Set symbol on any pane |
| `chart_scroll_to_date` | Jump to date (ISO: "2025-01-15") |
| `draw_shape` | 