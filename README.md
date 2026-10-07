# solana-early-alert-bot
Core gates:

import os
def env_int(name, default):
    try: return int(os.getenv(name, default))
    except ValueError: return default
CONFIG = {
 "buys_5m": env_int("BUYS_5M",100),
 "unique_buyers_5m": env_int("UNIQUE_BUYERS_5M",50),
 "buy_sell_ratio": float(os.getenv("BUY_SELL_RATIO","1.5")),
 "volume_5m": float(os.getenv("VOLUME_5M","25000")),
 "liquidity": float(os.getenv("LIQUIDITY","20000")),
 "social_5m": env_int("SOCIAL_5M",15),
 "smart_money": env_int("SMART_MONEY",3),
 "dev_pct": float(os.getenv("DEV_PCT","2")),
 "bundled_pct": float(os.getenv("BUNDLED_PCT","5")),
 "top10_pct": float(os.getenv("TOP10_PCT","20")),
 "risk_score_min": env_int("RISK_SCORE_MIN",75),
}
import os,requests
def send_telegram(text):
    token=os.getenv("TELEGRAM_BOT_TOKEN"); chat_id=os.getenv("TELEGRAM_CHAT_ID")
    if not token or not chat_id: return False
    r=requests.post(f"https://api.telegram.org/bot{token}/sendMessage",json={"chat_id":chat_id,"text":text},timeout=15)
    r.raise_for_status(); return True

def format_alert(m):
    return f"""🚨 EARLY MOMENTUM DETECTED ${m.get('symbol','UNKNOWN')} — Solana 🕐 Age: {m.get('age_minutes','?')} minutes 💰 MC: ${m.get('market_cap_usd',0):,.0f} 💧 Liquidity: ${m.get('liquidity_usd',0):,.0f} ⚡ BUY PRESSURE • {m.get('buys_5m',0)} buys / 5 min • {m.get('unique_buyers_5m',0)} unique buyers • Buy/Sell: {m.get('buy_sell_ratio',0):.2f}× • Volume: ${m.get('volume_5m',0):,.0f} 📣 SOCIAL EXPLOSION • {m.get('social_5m',0)} posts / 5 min • Telegram: {m.get('telegram_social_5m',0)} • X: {m.get('x_social_5m',0)} 🧠 SMART MONEY • {m.get('smart_money_wallets',0)} wallets entered • Combined buy: ${m.get('smart_money_usd',0):,.0f} 🛡️ TOKEN CHECK • Mint: {'✅' if m.get('mint_authority_off') else '❌'} • Freeze: {'✅' if m.get('freeze_authority_off') else '❌'} • Dev: {m.get('dev_pct',999):.2f}% • Bundled: {m.get('bundled_pct',999):.2f}% • Top 10: {m.get('top10_pct',999):.2f}% • Risk score: {m.get('risk_score',0):.0f}/100 🚀 DETECTION MC: ${m.get('market_cap_usd',0):,.0f} CA: `{m.get('mint','')}` ⚠️ Alert-only. Early momentum is not a guaranteed pump."""
from flask import Flask,jsonify
import os
app=Flask(__name__)
@app.get("/")
def home():
    return '<html><body style="font-family:system-ui;background:#0b0b0d;color:white;padding:40px"><h1>⚡ Solana Early Alert Bot</h1><p>Cloud deployment is ready.</p><p>Configure secrets in the host environment, then connect the live provider adapters.</p></body></html>'
@app.get("/health")
def health(): return jsonify(status="ok",telegram_configured=bool(os.getenv("TELEGRAM_BOT_TOKEN")))Core gates:



100+ buys / 5 min

50+ unique buyers

buy/sell >= 1.5x

volume >= $25K

liquidity >= $20K

15+ social posts / 5 min, exact count + Telegram/X breakdown

3+ tracked smart-money wallets / 10 min

mint + freeze authority OFF

dev <= 2%, bundled < 5%, top 10 < 20%

risk score >= 75

fail closed when critical security evidence is missing
import os,time,threading
from app import app
from provider import get_candidates,enrich
from scanner import qualifies
from alerts import send_telegram,format_alert
def worker():
    seen=set()
    while True:
        try:
            for mint in get_candidates():
                if mint in seen: continue
                data=enrich(mint)
                ok,_=qualifies(data)
                if ok:
                    send_telegram(format_alert(data)); seen.add(mint)
        except Exception as e: print("scanner error:",e,flush=True)
        time.sleep(15)
if __name__=="__main__":
    threading.Thread(target=worker,daemon=True).start()
    app.run(host="0.0.0.0",port=int(os.getenv("PORT","10000")))
def security_gate(x,cfg): checks={
      "mint_off":x.get("mint_authority_off") is True,
      "freeze_off":x.get("freeze_authority_off") is True,
      "sellable":x.get("sellable") is True,
      "no_honeypot":x.get("honeypot") is False,
      "liquidity":float(x.get("liquidity_usd",0))>=cfg["liquidity"],
      "dev":float(x.get("dev_pct",999))<=cfg["dev_pct"],
      "bundled":float(x.get("bundled_pct",999))<cfg["bundled_pct"],
      "top10":float(x.get("top10_pct",999))<cfg["top10_pct"],
      "linked_wallet_risk":x.get("linked_wallet_risk") is False,
      "risk_score":float(x.get("risk_score",0))>=cfg["risk_score_min"]}
    return all(checks.values()),checks
# Production adapters go here.
# Helius: Solana transaction/wallet activity
# DexScreener: market, liquidity, volume
# X + Telegram data source: social counts
# RPC/security provider: authorities, holders, sellability, LP status
def get_candidates(): return []
def enrich(mint): raise NotImplementedError("Connect live provider adapters first.")
from config import CONFIG from security import security_gate def qualifies(m): momentum=(m.get("buys_5m",0)>=CONFIG["buys_5m"] and
              m.get("unique_buyers_5m",0)>=CONFIG["unique_buyers_5m"] and
              m.get("buy_sell_ratio",0)>=CONFIG["buy_sell_ratio"] and
              m.get("volume_5m",0)>=CONFIG["volume_5m"] and
              m.get("liquidity_usd",0)>=CONFIG["liquidity"])
    ok_security,checks=security_gate(m,CONFIG)
    return momentum and m.get("social_5m",0)>=CONFIG["social_5m"] and m.get("smart_money_wallets",0)>=CONFIG["smart_money"] and ok_security,checks
