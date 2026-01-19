# app.py
import streamlit as st
import yfinance as yf
import pandas as pd
import plotly.graph_objects as go

# ----------------------
# Portfolio CSV
# ----------------------
PORTFOLIO_PATH = "portfolio.csv"

# Create file if it doesn't exist
try:
    df = pd.read_csv(PORTFOLIO_PATH)
except:
    df = pd.DataFrame(columns=["Symbol", "Shares", "Buy_Price"])
    df.to_csv(PORTFOLIO_PATH, index=False)

# ----------------------
# Page title
# ----------------------
st.title("📈 My Stock Dashboard")

# ----------------------
# Add Stock
# ----------------------
st.subheader("Add a Stock")
col1, col2, col3 = st.columns(3)
with col1:
    symbol = st.text_input("Symbol", "")
with col2:
    shares = st.number_input("Shares", min_value=0)
with col3:
    buy_price = st.number_input("Buy Price", min_value=0.0)

if st.button("Add Stock"):
    symbol_clean = symbol.strip().upper().replace("$","")
    if symbol_clean == "":
        st.error("❌ Enter a valid symbol")
    elif symbol_clean in df["Symbol"].values:
        st.error(f"❌ {symbol_clean} already in portfolio")
    else:
        try:
            hist = yf.Ticker(symbol_clean).history(period="1d")
            if hist.empty:
                st.error(f"❌ {symbol_clean} does not exist or has no data")
            else:
                df = pd.concat([df, pd.DataFrame([{
                    "Symbol": symbol_clean,
                    "Shares": shares,
                    "Buy_Price": buy_price
                }])], ignore_index=True)
                df.to_csv(PORTFOLIO_PATH, index=False)
                st.success(f"✅ Added {symbol_clean}!")
        except Exception as e:
            st.error(f"❌ Error fetching {symbol_clean}: {e}")

# ----------------------
# Delete Stock
# ----------------------
st.subheader("Delete a Stock")
symbol_to_delete = st.selectbox("Select stock to delete", df["Symbol"].tolist())
if st.button("Delete Stock"):
    df = df[df["Symbol"] != symbol_to_delete]
    df.to_csv(PORTFOLIO_PATH, index=False)
    st.success(f"🗑️ Deleted {symbol_to_delete}")

# ----------------------
# Show Table
# ----------------------
st.subheader("Portfolio Table")

# Refresh prices
results = []
for _, row in df.iterrows():
    sym = row["Symbol"]
    shares_ = row["Shares"]
    buy_price_ = row["Buy_Price"]
    hist = yf.Ticker(sym).history(period="3mo")
    if hist.empty:
        continue
    current_price = hist["Close"][-1]
    profit = (current_price - buy_price_) * shares_
    results.append({
        "Symbol": sym,
        "Shares": shares_,
        "Buy Price": round(buy_price_,2),
        "Current Price": round(current_price,2),
        "Profit/Loss": round(profit,2)
    })

table_df = pd.DataFrame(results)
st.dataframe(table_df)

# ----------------------
# Show Chart
# ----------------------
st.subheader("Stock Prices (Last 3 Months)")
fig = go.Figure()
for sym in table_df["Symbol"]:
    hist = yf.Ticker(sym).history(period="3mo")
    fig.add_trace(go.Scatter(x=hist.index, y=hist["Close"], mode="lines", name=sym))
fig.update_layout(title="Stock Prices", xaxis_title="Date", yaxis_title="Close Price", height=500)
st.plotly_chart(fig)
