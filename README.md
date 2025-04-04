# analysis.py
import requests
import pandas as pd
import numpy as np
from ta.trend import EMAIndicator, MACD
from ta.momentum import RSIIndicator

def fetch_ohlcv(symbol: str, interval: str, limit=100):
    binance_intervals = {
        "1": "1m", "5": "5m", "15": "15m", "30": "30m", "60": "1h", "240": "4h", "1d": "1d"
    }

    if interval not in binance_intervals:
        return None, "Geçersiz zaman dilimi"

    url = f"https://api.binance.com/api/v3/klines?symbol={symbol.upper()}&interval={binance_intervals[interval]}&limit={limit}"
    res = requests.get(url)

    if res.status_code != 200:
        return None, "Veri çekilemedi"

    data = res.json()
    df = pd.DataFrame(data, columns=[
        "timestamp", "open", "high", "low", "close", "volume",
        "close_time", "quote_asset_volume", "number_of_trades",
        "taker_buy_base_volume", "taker_buy_quote_volume", "ignore"
    ])

    df["close"] = df["close"].astype(float)
    df["open"] = df["open"].astype(float)
    df["high"] = df["high"].astype(float)
    df["low"] = df["low"].astype(float)

    return df, None

def analyze_pair(symbol: str, interval: str):
    df, err = fetch_ohlcv(symbol, interval)

    if err:
        return f"Hata: {err}"

    df["rsi"] = RSIIndicator(df["close"]).rsi()
    macd = MACD(close=df["close"])
    df["macd"] = macd.macd()
    df["macd_signal"] = macd.macd_signal()
    df["ema"] = EMAIndicator(close=df["close"], window=20).ema_indicator()

    rsi_val = df["rsi"].iloc[-1]
    macd_val = df["macd"].iloc[-1]
    macd_signal = df["macd_signal"].iloc[-1]
    ema_val = df["ema"].iloc[-1]
    close_val = df["close"].iloc[-1]

    yorumlar = []

    if rsi_val < 30:
        yorumlar.append("📉 RSI düşük, aşırı satım bölgesinde.")
    elif rsi_val > 70:
        yorumlar.append("📈 RSI yüksek, aşırı alım bölgesinde.")
    else:
        yorumlar.append("🔁 RSI nötr seviyede.")

    if macd_val > macd_signal:
        yorumlar.append("✅ MACD al sinyali veriyor.")
    else:
        yorumlar.append("❌ MACD sat sinyali veriyor.")

    if close_val > ema_val:
        yorumlar.append("💹 Fiyat EMA üzerinde, yükseliş eğiliminde.")
    else:
        yorumlar.append("📉 Fiyat EMA altında, düşüş eğiliminde.")

    return f"""
🪙 {symbol.upper()} - {interval} dakikalık analiz:

RSI: {rsi_val:.2f}
MACD: {macd_val:.4f}
Signal: {macd_signal:.4f}
EMA(20): {ema_val:.2f}
Close: {close_val:.2f}

Yorum:
""" + "\n".join(yorumlar)
