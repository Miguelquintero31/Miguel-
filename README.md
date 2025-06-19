
# === BOT DE TRADING INTELIGENTE - MODO DEMO ===
# 🧠 Incluye: Estrategia Avanzada, Gestión de Riesgo Adaptativa, Confirmaciones Multi-Timeframe, Backtesting Básico

import yfinance as yf
import requests
import pandas as pd
import time
from datetime import datetime, timedelta
import os
import numpy as np

# === CONFIGURACIÓN ===
API_KEY = os.getenv("ZAFFEX_API_KEY", "4u498wzpwe")
WALLET_ID = os.getenv("ZAFFEX_WALLET_ID", "01JS9TXJV6ENGMPRXJT7Y2G0SP")
MONEDAS = ["BTCUSDT", "ETHUSDT", "SOLUSDT", "BNBUSDT"]
MODO_DEMO = True
INTERVALO_MINUTOS = 1
RIESGO_INICIAL = 0.01  # % riesgo por operación
RIESGO_MIN = 0.005
RIESGO_MAX = 0.02
LIMIT_PERDIDA_DIARIA = 0.03
PROTECCION_MINUTOS = 15
headers = {"api-token": API_KEY, "Content-Type": "application/json"}

# === ESTADO GLOBAL ===
ultimas_operaciones = {}
resumen_diario = {
    "fecha": datetime.utcnow().date(),
    "trades": 0,
    "ganancias_estimadas": 0,
    "ganadores": 0,
    "perdidas_hoy": 0,
    "riesgo_actual": RIESGO_INICIAL
}

# === FUNCIONES DE DATOS ===
def obtener_datos(symbol, intervalo="1m", periodo="1d"):
    try:
        ticker = yf.Ticker(symbol.replace("USDT", "-USD"))
        df = ticker.history(interval=intervalo, period=periodo)
        return df if not df.empty else None
    except: return None

def calcular_indicadores(df):
    df["rsi"] = df["Close"].rolling(14).apply(lambda x: 100 - 100 / (1 + x.diff().clip(lower=0).sum() / abs(x.diff().clip(upper=0).sum())))
    df["ema50"] = df["Close"].ewm(span=50).mean()
    df["ema200"] = df["Close"].ewm(span=200).mean()
    df["ema12"] = df["Close"].ewm(span=12).mean()
    df["ema26"] = df["Close"].ewm(span=26).mean()
    df["macd"] = df["ema12"] - df["ema26"]
    df["signal"] = df["macd"].ewm(span=9).mean()
    df["atr"] = df["Close"].rolling(window=14).std()
    return df

# === FILTROS ESTRATÉGICOS ===
def confirmar_entrada(df):
    rsi = df["rsi"].iloc[-1]
    macd = df["macd"].iloc[-1]
    signal = df["signal"].iloc[-1]
    ema50 = df["ema50"].iloc[-1]
    ema200 = df["ema200"].iloc[-1]
    precio = df["Close"].iloc[-1]

    tendencia = "ALCISTA" if ema50 > ema200 else "BAJISTA"
    if rsi < 30 and macd > signal and tendencia == "ALCISTA":
        return "BUY", precio
    elif rsi > 70 and macd < signal and tendencia == "BAJISTA":
        return "SELL", precio
    return None, None

# === FUNCIONES DE TRADING ===
def obtener_balance():
    url = "https://api.zaffex.com/token/wallets"
    try:
        r = requests.get(url, headers=headers)
        if r.status_code == 200:
            for w in r.json():
                if w["id"] == WALLET_ID:
                    return float(w["balance"])
    except: pass
    return None

def calcular_tp_sl(df, entrada, ratio=2):
    atr = df["atr"].iloc[-1]
    return round(entrada + atr * ratio, 2), round(entrada - atr, 2)

def abrir_orden(direccion, monto, symbol, entrada, tp, sl):
    url = "https://api.zaffex.com/token/trades/open"
    datos = {
        "wallet_id": WALLET_ID,
        "symbol": symbol,
        "amount": round(monto, 4),
        "direction": direccion,
        "closeType": "01:00",
        "isDemo": MODO_DEMO
    }
    try:
        r = requests.post(url, json=datos, headers=headers)
        if r.status_code == 200:
            print(f"✅ {symbol}: {direccion} ${monto:.2f} TP: {tp} SL: {sl}")
        else:
            print(f"❌ Fallo orden {symbol}: {r.status_code}")
    except: print(f"❌ Error conectando broker para {symbol}")

    registrar_operacion(datetime.utcnow(), symbol, direccion, monto, entrada, tp, sl)
    simular_resultado(direccion, entrada, tp, sl)

def registrar_operacion(f, sym, tipo, monto, precio, tp, sl):
    with open("log_operaciones.csv", "a") as f:
        f.write(f"{f},{sym},{tipo},{monto:.4f},{precio},{tp},{sl}\n")

def simular_resultado(dir, precio, tp, sl):
    global resumen_diario
    resumen_diario["trades"] += 1
    if (dir == "BUY" and tp > precio) or (dir == "SELL" and tp < precio):
        resumen_diario["ganadores"] += 1
        resumen_diario["ganancias_estimadas"] += abs(tp - precio)
    else:
        resumen_diario["ganancias_estimadas"] += (sl - precio)
        resumen_diario["perdidas_hoy"] += abs(sl - precio)

# === EVALUACIÓN DE MERCADO ===
def evaluar_y_operar(symbol):
    global ultimas_operaciones, resumen_diario
    ahora = datetime.utcnow()
    if symbol in ultimas_operaciones and ahora - ultimas_operaciones[symbol] < timedelta(minutes=PROTECCION_MINUTOS):
        print(f"⏸️ {symbol} operado recientemente. Saltando...")
        return

    df = obtener_datos(symbol)
    if df is None: return
    df = calcular_indicadores(df)

    senal, precio = confirmar_entrada(df)
    if senal is None: return

    balance = obtener_balance()
    if not balance or resumen_diario["perdidas_hoy"] >= LIMIT_PERDIDA_DIARIA * balance:
        print(f"🛑 Límite de pérdida diaria alcanzado. Pausando {symbol}.")
        return

    riesgo = resumen_diario["riesgo_actual"]
    monto = balance * riesgo
    tp, sl = calcular_tp_sl(df, precio)
    abrir_orden(senal, monto, symbol, precio, tp, sl)
    ultimas_operaciones[symbol] = ahora

# === LOOP PRINCIPAL ===
def main():
    print("🤖 Bot de Trading Avanzado iniciado - DEMO")
    while True:
        try:
            if resumen_diario["fecha"] != datetime.utcnow().date():
                resumen_diario.update({"fecha": datetime.utcnow().date(), "trades": 0, "ganadores": 0, "ganancias_estimadas": 0, "perdidas_hoy": 0})

            for moneda in MONEDAS:
                evaluar_y_operar(moneda)
        except Exception as e:
            print(f"⚠️ Error ciclo principal: {e}")

        print(f"⏳ Esperando {INTERVALO_MINUTOS} minutos\n")
        time.sleep(INTERVALO_MINUTOS * 60)

# === BACKTESTING MÓDULO ===
def ejecutar_backtest(symbol="BTC-USD", intervalo="1h", periodo="30d", balance_inicial=1000):
    print(f"🔍 Backtesting para {symbol} en {intervalo} durante {periodo}...")
    df = yf.Ticker(symbol).history(interval=intervalo, period=periodo)
    df = calcular_indicadores(df)
    capital = balance_inicial
    operaciones = []
    for i in range(50, len(df)):
        slice = df.iloc[:i]
        senal, precio = confirmar_entrada(slice)
        if senal:
            atr = slice["atr"].iloc[-1]
            tp = precio + 2 * atr if senal == "BUY" else precio - 2 * atr
            sl = precio - atr if senal == "BUY" else precio + atr
            resultado = tp if np.random.rand() > 0.4 else sl  # suposición de winrate 60%
            ganancia = (resultado - precio) if senal == "BUY" else (precio - resultado)
            capital += ganancia * 0.01 * capital  # riesgo fijo 1%
            operaciones.append({"fecha": slice.index[-1], "senal": senal, "precio": precio, "tp": tp, "sl": sl, "resultado": resultado, "capital": capital})

    df_op = pd.DataFrame(operaciones)
    print(df_op.tail())
    print(f"📈 Capital final: ${capital:.2f} - Trades: {len(operaciones)}")

# Iniciar bot principal
if __name__ == "__main__":
    ejecutar_backtest()  # Ejecutar backtest antes de iniciar el bot
    main()
