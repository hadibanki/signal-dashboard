import streamlit as st
import pandas as pd
import requests
import matplotlib.pyplot as plt

st.set_page_config(page_title="Crypto Signal Dashboard", layout="wide")

st.title("📊 Crypto Signal Dashboard")

# لینک فایل CSV گوگل شیت
sheet_url = "https://docs.google.com/spreadsheets/d/16M0I02KZKuAgBF00Lw_DKYhEC7MCi0NOPzYFGbZefRA/export?format=csv&gid=0"

@st.cache_data
def load_data(url):
    df = pd.read_csv(url)
    df.columns = [col.strip() for col in df.columns]
    return df

df = load_data(sheet_url)

# تابع گرفتن قیمت لحظه‌ای از Binance API
def get_live_price(symbol):
    try:
        url = f"https://api.binance.com/api/v3/ticker/price?symbol={symbol.upper()}USDT"
        res = requests.get(url)
        return float(res.json()['price'])
    except:
        return None

# تابع تحلیل سیگنال
def analyze_signal(row, initial_capital=1000):
    try:
        symbol = row['SYMBOL']
        position = row['POSITION'].lower()
        entry = float(row['ENTRY'])
        leverage = float(str(row['LEVERAGE']).replace('x','').strip())
        mm_percent = float(str(row['MM%']).replace('%','').strip())
        sl = float(row['SL'])
        tps = [float(row.get(f'TP{i}', None)) for i in range(1, 5)]
        price = get_live_price(symbol)
        if price is None:
            return pd.Series(["❌ Error", 0, 0])

        active_capital = initial_capital * mm_percent / 100

        if position == 'buy':
            if price <= sl:
                pnl_percent = ((sl - entry) / entry) * leverage * 100
                return pd.Series(["🔴 Stopped", pnl_percent, active_capital * pnl_percent / 100])
            for i, tp in enumerate(tps):
                if price >= tp:
                    pnl_percent = ((tp - entry) / entry) * leverage * 100
                    return pd.Series([f"🎯 Hit TP{i+1}", pnl_percent, active_capital * pnl_percent / 100])
            return pd.Series(["⏳ Running", ((price - entry) / entry) * leverage * 100,
                              active_capital * ((price - entry) / entry) * leverage])
        elif position == 'sell':
            if price >= sl:
                pnl_percent = ((entry - sl) / entry) * leverage * 100
                return pd.Series(["🔴 Stopped", pnl_percent, active_capital * pnl_percent / 100])
            for i, tp in enumerate(tps):
                if price <= tp:
                    pnl_percent = ((entry - tp) / entry) * leverage * 100
                    return pd.Series([f"🎯 Hit TP{i+1}", pnl_percent, active_capital * pnl_percent / 100])
            return pd.Series(["⏳ Running", ((entry - price) / entry) * leverage * 100,
                              active_capital * ((entry - price) / entry) * leverage])
        else:
            return pd.Series(["❌ Invalid Position", 0, 0])
    except:
        return pd.Series(["❌ Error", 0, 0])

# تحلیل همه سیگنال‌ها
df[['Status', 'PnL_%', 'PnL_$']] = df.apply(analyze_signal, axis=1)
df['Balance'] = 1000 + df['PnL_$'].cumsum()

# آمار کلی
win_rate = len(df[df['Status'].str.contains("Hit")]) / len(df) * 100
final_balance = df['Balance'].iloc[-1]
total_profit = df['PnL_$'].sum()
open_signals = df[df['Status'] == '⏳ Running']

# نمایش آمار
col1, col2, col3 = st.columns(3)
col1.metric("💰 Final Balance", f"${final_balance:.2f}")
col2.metric("📈 Total PnL", f"${total_profit:.2f}")
col3.metric("✅ Win Rate", f"{win_rate:.2f}%")

st.markdown("---")

# جدول سیگنال‌ها با وضعیت
st.subheader("📋 All Signals")
st.dataframe(df[['SYMBOL', 'POSITION', 'LEVERAGE', 'ENTRY', 'TP1', 'TP2', 'TP3', 'TP4', 'SL', 'MM%', 'Status', 'PnL_%', 'PnL_$']], use_container_width=True)

# نمودار رشد سرمایه
st.subheader("📊 Capital Growth Chart")
fig, ax = plt.subplots()
ax.plot(df['Balance'], marker='o')
ax.set_title('Growth of $1000 Initial Capital')
ax.set_xlabel('Signals')
ax.set_ylabel('Balance ($)')
ax.grid(True)
st.pyplot(fig)

# سیگنال‌های باز
st.subheader("⏳ Active Signals")
st.dataframe(open_signals[['SYMBOL', 'POSITION', 'ENTRY', 'SL', 'TP1', 'TP2', 'TP3', 'TP4', 'Status']], use_container_width=True)

st.markdown("---")
st.caption("Built with ❤️ using Streamlit")
