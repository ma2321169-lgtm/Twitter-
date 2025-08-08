# Twitter-
A simple Telegram bot for personal trading signals
import os
import hmac
import hashlib
from flask import Flask, request, jsonify
import requests

# Environment variables (set these in Render / Railway / PythonAnywhere)
TELEGRAM_BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")
TELEGRAM_USER_ID = os.getenv("TELEGRAM_USER_ID")  # your telegram user id (as integer/string)
WEBHOOK_SECRET = os.getenv("WEBHOOK_SECRET")      # a random secret string you choose

if not all([TELEGRAM_BOT_TOKEN, TELEGRAM_USER_ID, WEBHOOK_SECRET]):
    raise RuntimeError("Set TELEGRAM_BOT_TOKEN, TELEGRAM_USER_ID, and WEBHOOK_SECRET in env vars")

TG_API = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"
app = Flask(__name__)

def send_telegram_message(text: str):
    payload = {
        "chat_id": TELEGRAM_USER_ID,
        "text": text,
        "parse_mode": "Markdown"
    }
    r = requests.post(TG_API, json=payload, timeout=10)
    return r.status_code, r.text

def verify_signature(body_bytes: bytes, secret: str, header_sig: str) -> bool:
    if not header_sig:
        return False
    computed = hmac.new(secret.encode(), body_bytes, hashlib.sha256).hexdigest()
    if header_sig.startswith("sha256="):
        header_sig = header_sig.split("=", 1)[1]
    return hmac.compare_digest(computed, header_sig)

@app.route("/", methods=["GET"])
def index():
    return "Signal Relay Bot is alive", 200

@app.route("/webhook", methods=["POST"])
def webhook():
    body_bytes = request.get_data()
    try:
        payload = request.get_json(force=True)
    except Exception:
        return jsonify({"error": "Invalid JSON"}), 400

    # Option A: require a JSON field "secret" that matches WEBHOOK_SECRET
    if payload.get("secret") == WEBHOOK_SECRET:
        authenticated = True
    else:
        # Option B: support HMAC header 'X-Signature'
        header_sig = request.headers.get("X-Signature", "")
        if verify_signature(body_bytes, WEBHOOK_SECRET, header_sig):
            authenticated = True
        else:
            return jsonify({"error": "Forbidden"}), 403

    # Format the signal message
    symbol = payload.get("symbol", "N/A")
    action = payload.get("action", "N/A")
    price = payload.get("price", "N/A")
    risk = payload.get("risk", "")
    note = payload.get("note", "")

    msg = f"*Signal Alert*\nSymbol: `{symbol}`\nAction: *{action}*\nPrice: `{price}`"
    if risk:
        msg += f"\nRisk: `{risk}`"
    if note:
        msg += f"\nNote: _{note}_"

    status, resp = send_telegram_message(msg)
    if status == 200:
        return jsonify({"ok": True}), 200
    else:
        return jsonify({"error": "Telegram send failed", "details": resp}), 500

if __name__ == "__main__":
    app.run()
