import email
import imaplib
import os
import random
import re
import string
from datetime import datetime
from email.header import decode_header
from email.utils import parsedate_to_datetime

from bs4 import BeautifulSoup
from dotenv import load_dotenv
from flask import Flask, jsonify, render_template, request

load_dotenv()

app = Flask(__name__)

BASE_GMAIL = os.getenv("BASE_GMAIL", "").strip()
GMAIL_APP_PASSWORD = os.getenv("GMAIL_APP_PASSWORD", "").strip()
IMAP_HOST = "imap.gmail.com"


def require_config():
    if not BASE_GMAIL or not GMAIL_APP_PASSWORD:
        raise RuntimeError("Missing BASE_GMAIL or GMAIL_APP_PASSWORD in .env")
    if not BASE_GMAIL.lower().endswith("@gmail.com"):
        raise RuntimeError("BASE_GMAIL must end with @gmail.com")


def make_alias(label="web"):
    require_config()
    name, domain = BASE_GMAIL.split("@", 1)
    safe_label = re.sub(r"[^a-zA-Z0-9_-]", "", label)[:20] or "web"
    stamp = datetime.now().strftime("%Y%m%d%H%M%S")
    suffix = "".join(random.choices(string.ascii_lowercase + string.digits, k=8))
    return f"{name}+{safe_label}-{stamp}-{suffix}@{domain}"


def decode_text(value):
    if not value:
        return ""

    parts = decode_header(value)
    decoded = []
    for text, charset in parts:
        if isinstance(text, bytes):
            decoded.append(text.decode(charset or "utf-8", errors="replace"))
        else:
            decoded.append(text)
    return "".join(decoded)


def get_message_parts(message):
    html_body = ""
    text_body = ""

    if message.is_multipart():
        for part in message.walk():
            content_type = part.get_content_type()
            disposition = str(part.get("Content-Disposition", ""))

            if "attachment" in disposition:
                continue

            payload = part.get_payload(decode=True)
            if not payload:
                continue

            charset = part.get_content_charset() or "utf-8"
            body = payload.decode(charset, errors="replace")

            if content_type == "text/html" and not html_body:
                html_body = body
            elif content_type == "text/plain" and not text_body:
                text_body = body
    else:
        payload = message.get_payload(decode=True)
        if payload:
            charset = message.get_content_charset() or "utf-8"
            body = payload.decode(charset, errors="replace")
            if message.get_content_type() == "text/html":
                html_body = body
            else:
                text_body = body

    if not html_body and text_body:
        html_body = f"<pre>{text_body}</pre>"

    plain_preview = text_body or BeautifulSoup(html_body, "html.parser").get_text(" ", strip=True)
    return html_body, plain_preview


def connect_mailbox():
    require_config()
    mailbox = imaplib.IMAP4_SSL(IMAP_HOST)
    mailbox.login(BASE_GMAIL, GMAIL_APP_PASSWORD)
    mailbox.select("INBOX")
    return mailbox


def fetch_alias_messages(alias):
    mailbox = connect_mailbox()
    try:
        status, data = mailbox.search(None, "TO", f'"{alias}"')
        if status != "OK":
            return []

        ids = data[0].split()[-25:]
        messages = []

        for msg_id in reversed(ids):
            status, msg_data = mailbox.fetch(msg_id, "(RFC822)")
            if status != "OK":
                continue

            raw = msg_data[0][1]
            parsed = email.message_from_bytes(raw)
            html_body, preview = get_message_parts(parsed)

            date_value = parsed.get("Date")
            try:
                received_at = parsedate_to_datetime(date_value).isoformat() if date_value else ""
            except Exception:
                received_at = date_value or ""

            messages.append({
                "id": msg_id.decode("utf-8", errors="replace"),
                "from": decode_text(parsed.get("From")),
                "to": decode_text(parsed.get("To")),
                "subject": decode_text(parsed.get("Subject")) or "(No subject)",
                "date": received_at,
                "preview": preview[:240],
                "html": html_body,
            })

        return messages
    finally:
        mailbox.close()
        mailbox.logout()


@app.route("/")
def index():
    return render_template("index.html", base_gmail=BASE_GMAIL)


@app.route("/api/new-email", methods=["POST"])
def api_new_email():
    label = (request.json or {}).get("label", "web")
    return jsonify({"email": make_alias(label)})


@app.route("/api/inbox")
def api_inbox():
    alias = request.args.get("email", "").strip()
    if not alias:
        return jsonify({"messages": []})
    return jsonify({"messages": fetch_alias_messages(alias)})


if __name__ == "__main__":
    app.run(debug=True)
