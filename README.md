# ai-trading-bot
import time
from datetime import datetime, timedelta
import numpy as np
import pandas as pd
import plotly.graph_objects as go
import streamlit as st

# -----------------------------------------------------------------------------
# PAGE CONFIGURATION & MOBILE-FRIENDLY DARK THEME CSS
# -----------------------------------------------------------------------------
st.set_page_config(
    page_title="AI Trading Bot | AISC [FOB]",
    page_icon="📈",
    layout="wide",
    initial_sidebar_state="collapsed",
)

st.markdown(
    """
    <style>
    /* Base dark theme and mobile-friendly spacing */
    .stApp {
        background-color: #0b0e14;
        color: #e6edf3;
    }
    .block-container {
        padding-top: 1.2rem;
        padding-bottom: 2rem;
        padding-left: 1rem;
        padding-right: 1rem;
        max-width: 1200px;
    }
    /* Metric Cards */
    div[data-testid="stMetric"] {
        background-color: #131722;
        border: 1px solid #2a2e39;
        border-radius: 10px;
        padding: 12px 14px;
    }
    div[data-testid="stMetricLabel"] {
        color: #848e9c;
        font-size: 0.82rem;
    }
    div[data-testid="stMetricValue"] {
        color: #f0f3fa;
        font-size: 1.35rem;
        font-weight: 700;
    }
    /* Containers & Panels */
    .panel-card {
        background-color: #131722;
        border: 1px solid #2a2e39;
        border-radius: 10px;
        padding: 14px;
        margin-bottom: 12px;
    }
    .status-badge {
        display: inline-block;
        padding: 4px 10px;
        border-radius: 999px;
        font-size: 0.78rem;
        font-weight: 600;
        letter-spacing: 0.4px;
    }
    .status-running { background-color: rgba(0, 200, 83, 0.15); color: #00e676; border: 1px solid #00c853; }
    .status-paused  { background-color: rgba(255, 171, 0, 0.15); color: #ffd740; border: 1px solid #ffab00; }
    .status-stopped { background-color: rgba(255, 82, 82, 0.15); color: #ff5252; border: 1px solid #ff1744; }
    /* Full-width touch buttons on mobile */
    .stButton > button {
        width: 100%;
        border-radius: 8px;
        font-weight: 600;
        min-height: 42px;
    }
    </style>
    """,
    unsafe_allow_html=True,
)

# -----------------------------------------------------------------------------
# MARKET SPECIFICATIONS & SAMPLE DATA GENERATOR
# -----------------------------------------------------------------------------
MARKET_SPECS = {
    "XAUUSD (Gold)": {"base": 2745.00, "pip": 0.10, "volatility": 2.40, "contract": 100},
    "EURUSD": {"base": 1.0850, "pip": 0.0001, "volatility": 0.0012, "contract": 100000},
    "NAS100": {"base": 20450.0, "pip": 1.0, "volatility": 28.0, "contract": 1},
    "BTCUSD": {"base": 91200.0, "pip": 1.0, "volatility": 180.0, "contract": 1},
}

TIMEFRAME_MINUTES = {"5m": 5, "15m": 15, "1H": 60, "4H": 240}


def generate_initial_candles(market: str, timeframe: str, count: int = 65) -> pd.DataFrame:
    spec = MARKET_SPECS[market]
    tf_min = TIMEFRAME_MINUTES[timeframe]
    rng = np.random.default_rng(seed=abs(hash(f"{market}_{timeframe}")) % (2**32))

    now = datetime.now().replace(second=0, microsecond=0)
    timestamps = [now - timedelta(minutes=tf_min * (count - i)) for i in range(count)]

    price = spec["base"]
    rows = []
    for ts in timestamps:
        delta = rng.normal(0, spec["volatility"] * 0.55)
        open_p = price
        close_p = open_p + delta
        wick_up = abs(rng.normal(0, spec["volatility"] * 0.35))
        wick_dn = abs(rng.normal(0, spec["volatility"] * 0.35))
        high_p = max(open_p, close_p) + wick_up
        low_p = min(open_p, close_p) - wick_dn
        rows.append(
            {
                "time": ts,
                "open": round(open_p, 5),
                "high": round(high_p, 5),
                "low": round(low_p, 5),
                "close": round(close_p, 5),
            }
        )
        price = close_p

    df = pd.DataFrame(rows)
    return apply_aisc_confluence_placeholder(df)


def apply_aisc_confluence_placeholder(df: pd.DataFrame) -> pd.DataFrame:
    """
    Placeholder for AISC [FOB] AI Signal Confluence logic.
    Replace this function's condition checks when porting your TradingView Pine Script rules.
    """
    df = df.copy()
    df["ema_fast"] = df["close"].ewm(span=9, adjust=False).mean()
    df["ema_slow"] = df["close"].ewm(span=21, adjust=False).mean()

    # RSI(14) calculation for confluence scoring
    delta = df["close"].diff()
    gain = delta.clip(lower=0).rolling(14).mean()
    loss = (-delta.clip(upper=0)).rolling(14).mean()
    rs = gain / loss.replace(0, np.nan)
    df["rsi"] = 100 - (100 / (1 + rs))
    df["rsi"] = df["rsi"].fillna(50.0)

    signals = [None] * len(df)
    confidences = [0] * len(df)

    for i in range(2, len(df)):
        bull_cross = df.loc[i - 1, "ema_fast"] <= df.loc[i - 1, "ema_slow"] and df.loc[i, "ema_fast"] > df.loc[i, "ema_slow"]
        bear_cross = df.loc[i - 1, "ema_fast"] >= df.loc[i - 1, "ema_slow"] and df.loc[i, "ema_fast"] < df.loc[i, "ema_slow"]

        # Simulated AISC [FOB] confluence conditions
        if bull_cross and df.loc[i, "rsi"] < 68:
            signals[i] = "BUY"
            confidences[i] = int(np.clip(78 + (68 - df.loc[i, "rsi"]) * 0.4, 75, 96))
        elif bear_cross and df.loc[i, "rsi"] > 32:
            signals[i] = "SELL"
            confidences[i] = int(np.clip(78 + (df.loc[i, "rsi"] - 32) * 0.4, 75, 96))

    df["signal"] = signals
    df["confidence"] = confidences
    return df


# -----------------------------------------------------------------------------
# SESSION STATE INITIALIZATION
# -----------------------------------------------------------------------------
def init_state():
    defaults = {
        "bot_status": "STOPPED",  # RUNNING | PAUSED | STOPPED
        "market": "XAUUSD (Gold)",
        "timeframe": "15m",
        "initial_balance": 10000.0,
        "balance": 10000.0,
        "lot_size": 0.10,
        "tp_pips": 80,
        "sl_pips": 40,
        "auto_trade_signals": True,
        "candles": None,
        "open_trades": [],
        "closed_trades": [],
        "signal_history": [],
        "trade_counter": 1000,
    }
    for k, v in defaults.items():
        if k not in st.session_state:
            st.session_state[k] = v

    if st.session_state.candles is None:
        df = generate_initial_candles(st.session_state.market, st.session_state.timeframe)
        st.session_state.candles = df
        # Seed initial signal history from historical sample candles
        sig_rows = df[df["signal"].notna()].tail(5)
        for _, r in sig_rows.iterrows():
            st.session_state.signal_history.insert(
                0,
                {
                    "Time": r["time"].strftime("%H:%M"),
                    "Market": st.session_state.market,
                    "TF": st.session_state.timeframe,
                    "Signal": r["signal"],
                    "Price": r["close"],
                    "Confluence": f"{r['confidence']}%",
                    "Source": "AISC [FOB]",
                },
            )


init_state()


# -----------------------------------------------------------------------------
# SIMULATION ENGINE (CANDLE TICK, SIGNAL EXECUTION, TP/SL CHECKS)
# -----------------------------------------------------------------------------
def calculate_trade_pnl(trade: dict, current_price: float) -> float:
    spec = MARKET_SPECS[trade["market"]]
    diff = (current_price - trade["entry"]) if trade["side"] == "BUY" else (trade["entry"] - current_price)
    return round(diff * trade["lot"] * spec["contract"], 2)


def open_simulated_trade(side: str, entry_price: float, reason: str = "AISC [FOB] Signal"):
    spec = MARKET_SPECS[st.session_state.market]
    pip = spec["pip"]
    lot = st.session_state.lot_size
    tp_dist = st.session_state.tp_pips * pip
    sl_dist = st.session_state.sl_pips * pip

    tp_price = round(entry_price + tp_dist if side == "BUY" else entry_price - tp_dist, 5)
    sl_price = round(entry_price - sl_dist if side == "BUY" else entry_price + sl_dist, 5)

    st.session_state.trade_counter += 1
    st.session_state.open_trades.append(
        {
            "id": f"SIM-{st.session_state.trade_counter}",
            "market": st.session_state.market,
            "side": side,
            "lot": lot,
            "entry": entry_price,
            "sl": sl_price,
            "tp": tp_price,
            "opened_at": datetime.now().strftime("%H:%M:%S"),
            "reason": reason,
        }
    )


def close_simulated_trade(trade_id: str, exit_price: float, close_reason: str = "Manual Close"):
    remaining = []
    for t in st.session_state.open_trades:
        if t["id"] == trade_id:
            pnl = calculate_trade_pnl(t, exit_price)
            st.session_state.balance = round(st.session_state.balance + pnl, 2)
            st.session_state.closed_trades.insert(
                0,
                {
                    "id": t["id"],
                    "market": t["market"],
                    "side": t["side"],
                    "lot": t["lot"],
                    "entry": t["entry"],
                    "exit": exit_price,
                    "pnl": pnl,
                    "reason": close_reason,
                    "closed_at": datetime.now().strftime("%H:%M:%S"),
                },
            )
        else:
            remaining.append(t)
    st.session_state.open_trades = remaining


def step_simulation():
    """Advances one simulated candle and evaluates AISC [FOB] signals & open positions."""
    df = st.session_state.candles.copy()
    spec = MARKET_SPECS[st.session_state.market]
    tf_min = TIMEFRAME_MINUTES[st.session_state.timeframe]

    last_row = df.iloc[-1]
    next_time = last_row["time"] + timedelta(minutes=tf_min)
    open_p = float(last_row["close"])

    # Gentle mean-reverting random walk with occasional momentum bursts
    drift = (spec["base"] - open_p) * 0.03
    move = np.random.normal(drift, spec["volatility"] * 0.65)
    close_p = round(open_p + move, 5)
    high_p = round(max(open_p, close_p) + abs(np.random.normal(0, spec["volatility"] * 0.3)), 5)
    low_p = round(min(open_p, close_p) - abs(np.random.normal(0, spec["volatility"] * 0.3)), 5)

    new_row = pd.DataFrame(
        [{"time": next_time, "open": open_p, "high": high_p, "low": low_p, "close": close_p}]
    )
    df = pd.concat([df.iloc[-79:], new_row], ignore_index=True)
    df = apply_aisc_confluence_placeholder(df)

    # Occasionally trigger an AISC [FOB] confluence signal on strong momentum candles
    latest_idx = len(df) - 1
    if df.loc[latest_idx, "signal"] is None and abs(move) > spec["volatility"] * 1.15:
        sig = "BUY" if move > 0 else "SELL"
        df.loc[latest_idx, "signal"] = sig
        df.loc[latest_idx, "confidence"] = int(np.random.randint(81, 96))

    st.session_state.candles = df
    latest = df.iloc[-1]

    # Check TP / SL hits on open trades for current market
    for trade in list(st.session_state.open_trades):
        if trade["market"] != st.session_state.market:
            continue
        if trade["side"] == "BUY":
            if low_p <= trade["sl"]:
                close_simulated_trade(trade["id"], trade["sl"], "Stop Loss Hit")
            elif high_p >= trade["tp"]:
                close_simulated_trade(trade["id"], trade["tp"], "Take Profit Hit")
        else:
            if high_p >= trade["sl"]:
                close_simulated_trade(trade["id"], trade["sl"], "Stop Loss Hit")
            elif low_p <= trade["tp"]:
                close_simulated_trade(trade["id"], trade["tp"], "Take Profit Hit")

    # Record new signal and open simulated trade if auto-execution is enabled
    if pd.notna(latest["signal"]):
        sig_type = latest["signal"]
        conf = int(latest["confidence"])
        st.session_state.signal_history.insert(
            0,
            {
                "Time": latest["time"].strftime("%H:%M"),
                "Market": st.session_state.market,
                "TF": st.session_state.timeframe,
                "Signal": sig_type,
                "Price": close_p,
                "Confluence": f"{conf}%",
                "Source": "AISC [FOB]",
            },
        )
        st.session_state.signal_history = st.session_state.signal_history[:25]

        if st.session_state.auto_trade_signals and len(st.session_state.open_trades) < 3:
            open_simulated_trade(sig_type, close_p, f"AISC [FOB] ({conf}%)")


# -----------------------------------------------------------------------------
# HEADER & BOT ENGINE CONTROLS (START / PAUSE / STOP)
# -----------------------------------------------------------------------------
header_left, header_right = st.columns([2, 1])
with header_left:
    st.markdown("### 🤖 AI Trading Bot")
    st.caption("Strategy Engine: **AISC [FOB] AI Signal Confluence** • Simulation Mode Only")

with header_right:
    status = st.session_state.bot_status
    badge_cls = {
        "RUNNING": "status-running",
        "PAUSED": "status-paused",
        "STOPPED": "status-stopped",
    }[status]
    st.markdown(
        f'<div style="text-align:right; padding-top:8px;">'
        f'<span class="status-badge {badge_cls}">● {status}</span>'
        f"</div>",
        unsafe_allow_html=True,
    )

ctrl_c1, ctrl_c2, ctrl_c3, ctrl_c4 = st.columns(4)
with ctrl_c1:
    if st.button("▶ Start", type="primary", use_container_width=True):
        st.session_state.bot_status = "RUNNING"
        st.rerun()
with ctrl_c2:
    if st.button("⏸ Pause", use_container_width=True):
        st.session_state.bot_status = "PAUSED"
        st.rerun()
with ctrl_c3:
    if st.button("⏹ Stop", use_container_width=True):
        st.session_state.bot_status = "STOPPED"
        st.rerun()
with ctrl_c4:
    if st.button("⏭ Step 1 Bar", use_container_width=True):
        step_simulation()
        st.rerun()

# -----------------------------------------------------------------------------
# ACCOUNT PERFORMANCE METRICS (BALANCE, SIMULATED PNL, WIN RATE)
# -----------------------------------------------------------------------------
df_candles = st.session_state.candles
current_price = float(df_candles.iloc[-1]["close"])

floating_pnl = sum(
    calculate_trade_pnl(t, current_price)
    for t in st.session_state.open_trades
    if t["market"] == st.session_state.market
)
realized_pnl = round(st.session_state.balance - st.session_state.initial_balance, 2)
total_pnl = round(realized_pnl + floating_pnl, 2)

closed_count = len(st.session_state.closed_trades)
wins = sum(1 for t in st.session_state.closed_trades if t["pnl"] > 0)
win_rate = f"{(wins / closed_count * 100):.1f}%" if closed_count > 0 else "0.0%"

m1, m2, m3, m4 = st.columns(4)
m1.metric("Simulated Balance", f"${st.session_state.balance:,.2f}")
m2.metric("Total Sim P/L", f"${total_pnl:+,.2f}", delta=f"{floating_pnl:+.2f} floating")
m3.metric("Win Rate", win_rate, delta=f"{wins}W / {closed_count - wins}L")
m4.metric("Active Market", st.session_state.market.split(" ")[0], delta=f"TF: {st.session_state.timeframe}")

# -----------------------------------------------------------------------------
# MARKET, TIMEFRAME & TRADE SETTINGS PANEL
# -----------------------------------------------------------------------------
with st.expander("⚙️ Market, Timeframe & Trade Settings", expanded=True):
    s_col1, s_col2 = st.columns(2)
    with s_col1:
        selected_market = st.selectbox(
            "Selected Market",
            list(MARKET_SPECS.keys()),
            index=list(MARKET_SPECS.keys()).index(st.session_state.market),
        )
    with s_col2:
        selected_tf = st.selectbox(
            "Timeframe",
            list(TIMEFRAME_MINUTES.keys()),
            index=list(TIMEFRAME_MINUTES.keys()).index(st.session_state.timeframe),
        )

    if selected_market != st.session_state.market or selected_tf != st.session_state.timeframe:
        st.session_state.market = selected_market
        st.session_state.timeframe = selected_tf
        st.session_state.candles = generate_initial_candles(selected_market, selected_tf)
        st.rerun()

    t_col1, t_col2, t_col3 = st.columns(3)
    with t_col1:
        st.session_state.lot_size = st.number_input(
            "Manual Lot Size",
            min_value=0.01,
            max_value=50.0,
            value=float(st.session_state.lot_size),
            step=0.01,
        )
    with t_col2:
        st.session_state.tp_pips = st.number_input(
            "Take Profit (Pips)",
            min_value=5,
            max_value=5000,
            value=int(st.session_state.tp_pips),
            step=5,
        )
    with t_col3:
        st.session_state.sl_pips = st.number_input(
            "Stop Loss (Pips)",
            min_value=5,
            max_value=5000,
            value=int(st.session_state.sl_pips),
            step=5,
        )

    b_col1, b_col2, b_col3 = st.columns([1.2, 1, 1])
    with b_col1:
        st.session_state.auto_trade_signals = st.toggle(
            "Auto-Execute AISC [FOB] Signals",
            value=st.session_state.auto_trade_signals,
        )
    with b_col2:
        if st.button("🟢 Sim BUY", use_container_width=True):
            open_simulated_trade("BUY", current_price, "Manual Simulation Order")
            st.rerun()
    with b_col3:
        if st.button("🔴 Sim SELL", use_container_width=True):
            open_simulated_trade("SELL", current_price, "Manual Simulation Order")
            st.rerun()

# -----------------------------------------------------------------------------
# LIVE-STYLE CANDLESTICK CHART + BUY/SELL SIGNAL INDICATORS
# -----------------------------------------------------------------------------
fig = go.Figure()

fig.add_trace(
    go.Candlestick(
        x=df_candles["time"],
        open=df_candles["open"],
        high=df_candles["high"],
        low=df_candles["low"],
        close=df_candles["close"],
        name=st.session_state.market,
        increasing_line_color="#00e676",
        decreasing_line_color="#ff5252",
    )
)

# Overlay EMA lines representing AISC [FOB] trend baseline
fig.add_trace(
    go.Scatter(
        x=df_candles["time"],
        y=df_candles["ema_fast"],
        mode="lines",
        name="Fast Baseline (9)",
        line=dict(color="#29b6f6", width=1.3),
    )
)
fig.add_trace(
    go.Scatter(
        x=df_candles["time"],
        y=df_candles["ema_slow"],
        mode="lines",
        name="Slow Baseline (21)",
        line=dict(color="#ffa726", width=1.3),
    )
)

# BUY Signal Markers
buy_df = df_candles[df_candles["signal"] == "BUY"]
if not buy_df.empty:
    offset = MARKET_SPECS[st.session_state.market]["volatility"] * 0.65
    fig.add_trace(
        go.Scatter(
            x=buy_df["time"],
            y=buy_df["low"] - offset,
            mode="markers+text",
            name="BUY Signal",
            text=["BUY" for _ in range(len(buy_df))],
            textposition="bottom center",
            textfont=dict(color="#00e676", size=10),
            marker=dict(symbol="triangle-up", size=13, color="#00e676", line=dict(width=1, color="#0b0e14")),
        )
    )

# SELL Signal Markers
sell_df = df_candles[df_candles["signal"] == "SELL"]
if not sell_df.empty:
    offset = MARKET_SPECS[st.session_state.market]["volatility"] * 0.65
    fig.add_trace(
        go.Scatter(
            x=sell_df["time"],
            y=sell_df["high"] + offset,
            mode="markers+text",
            name="SELL Signal",
            text=["SELL" for _ in range(len(sell_df))],
            textposition="top center",
            textfont=dict(color="#ff5252", size=10),
            marker=dict(symbol="triangle-down", size=13, color="#ff5252", line=dict(width=1, color="#0b0e14")),
        )
    )

# Horizontal lines for active open trades (Entry, TP, SL)
for trade in st.session_state.open_trades:
    if trade["market"] == st.session_state.market:
        fig.add_hline(y=trade["entry"], line_dash="dot", line_color="#90caf9", annotation_text=f"{trade['side']} Entry")
        fig.add_hline(y=trade["tp"], line_dash="dash", line_color="#00e676", annotation_text="TP")
        fig.add_hline(y=trade["sl"], line_dash="dash", line_color="#ff5252", annotation_text="SL")

fig.update_layout(
    template="plotly_dark",
    paper_bgcolor="#131722",
    plot_bgcolor="#131722",
    height=420,
    margin=dict(l=8, r=8, t=28, b=8),
    xaxis_rangeslider_visible=False,
    legend=dict(orientation="h", yanchor="bottom", y=1.01, xanchor="left", x=0, font=dict(size=11)),
    yaxis=dict(side="right", gridcolor="#1e222d"),
    xaxis=dict(gridcolor="#1e222d"),
)

st.plotly_chart(fig, use_container_width=True, config={"displayModeBar": False})

# -----------------------------------------------------------------------------
# OPEN TRADES PANEL (WITH MANUAL CLOSE TRADE BUTTON) & SIGNAL HISTORY PANEL
# -----------------------------------------------------------------------------
panel_left, panel_right = st.columns([1.15, 1])

with panel_left:
    st.markdown("#### 📂 Open Simulated Trades")
    if not st.session_state.open_trades:
        st.info("No open trades. Start the bot or use Sim BUY / Sim SELL to open a simulated position.")
    else:
        for trade in list(st.session_state.open_trades):
            mark_price = (
                current_price
                if trade["market"] == st.session_state.market
                else MARKET_SPECS[trade["market"]]["base"]
            )
            pnl = calculate_trade_pnl(trade, mark_price)
            pnl_color = "#00e676" if pnl >= 0 else "#ff5252"
            side_icon = "🟢" if trade["side"] == "BUY" else "🔴"

            with st.container(border=True):
                row1_a, row1_b = st.columns([2, 1])
                with row1_a:
                    st.markdown(
                        f"**{side_icon} {trade['side']} {trade['market']}** • `{trade['lot']} lots`  \n"
                        f"<span style='color:#848e9c;font-size:0.82rem;'>Entry: {trade['entry']} | TP: {trade['tp']} | SL: {trade['sl']}</span>",
                        unsafe_allow_html=True,
                    )
                with row1_b:
                    st.markdown(
                        f"<div style='text-align:right; font-weight:700; color:{pnl_color}; font-size:1.05rem;'>"
                        f"${pnl:+.2f}</div>",
                        unsafe_allow_html=True,
                    )
                    if st.button("Close Trade", key=f"close_{trade['id']}", use_container_width=True):
                        close_simulated_trade(trade["id"], mark_price, "Manual Close")
                        st.rerun()

with panel_right:
    st.markdown("#### 📡 Signal History (AISC [FOB])")
    if not st.session_state.signal_history:
        st.info("Waiting for AISC [FOB] confluence signals...")
    else:
        sig_df = pd.DataFrame(st.session_state.signal_history)
        st.dataframe(
            sig_df[["Time", "Market", "TF", "Signal", "Price", "Confluence"]],
            use_container_width=True,
            hide_index=True,
            height=260,
        )

# -----------------------------------------------------------------------------
# RECENT CLOSED TRADES LOG
# -----------------------------------------------------------------------------
if st.session_state.closed_trades:
    with st.expander("🧾 Closed Simulation Trades History", expanded=False):
        closed_df = pd.DataFrame(st.session_state.closed_trades)
        st.dataframe(closed_df, use_container_width=True, hide_index=True)

# -----------------------------------------------------------------------------
# AUTO-TICK LOOP WHEN BOT STATUS IS "RUNNING"
# -----------------------------------------------------------------------------
if st.session_state.bot_status == "RUNNING":
    time.sleep(1.5)
    step_simulation()
    st.rerun()
