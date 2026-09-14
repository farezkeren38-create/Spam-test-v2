#!/usr/bin/env python3
# ════════════════════════════════════════════════════════════
#   FLOWKIRIM TOOLKIT  ·  v2.0
#   ────────────────────────────────────────────────────────
#   modes:
#     1. send       — outbound text to a target number
#     2. webhook    — Flask listener for inbound messages
#     3. discover   — probe endpoint paths against a test num
# ════════════════════════════════════════════════════════════

import os
import re
import sys
import time
import json
import hmac
import hashlib
import signal
import threading
import requests

# ── palette ─────────────────────────────────────────────────
RST   = "\033[0m"
BOLD  = "\033[1m"
DIM   = "\033[2m"
RED   = "\033[38;5;196m"
DRED  = "\033[38;5;88m"
WHITE = "\033[38;5;255m"
GRAY  = "\033[38;5;240m"
GREEN = "\033[38;5;46m"
YEL   = "\033[38;5;220m"
BLUE  = "\033[38;5;39m"
CYAN  = "\033[38;5;51m"
PURP  = "\033[38;5;135m"
ORNG  = "\033[38;5;208m"

# ── config ──────────────────────────────────────────────────
CONFIG_FILE = "flowkirim_config.json"
LOG_FILE    = "flowkirim_log.txt"
HOOK_FILE   = "flowkirim_inbound.log"

DEFAULT_CONFIG = {
    "api_key":     "ef462a122cb3068153bd544d365ed1a7caff0cbb9c379e3dfb6bc287d8ed1697",
    "api_base":    "https://panel.flowkirim.com",
    # {phone} placeholders: {phone} {phone_08} {phone_jid} {session} {key}
    "url_template": "/api/send/{phone}",
    "auth_header": "Authorization",
    "auth_prefix": "Bearer ",
    "session_id":  "",
    "sender_name": "",
    "text":        "Hi there",
    "delay":       1.5,
    "workers":     1,
    "total":       0,
    "save_log":    True,
}

# ── helpers ─────────────────────────────────────────────────
def clear():
    os.system("cls" if os.name == "nt" else "clear")

def term_width():
    try:
        return os.get_terminal_size().columns
    except Exception:
        return 60

def hr(ch="─", color=GRAY):
    print(color + (ch * min(term_width(), 70)) + RST)

def phone_digits(raw):
    d = re.sub(r"\D", "", raw or "")
    if d.startswith("0"):
        return "62" + d[1:]
    if d.startswith("8"):
        return "62" + d
    return d

def valid_phone(p):
    return p.startswith("62") and p.isdigit() and 10 <= len(p) <= 15

# ── config io ───────────────────────────────────────────────
def load_config():
    if not os.path.exists(CONFIG_FILE):
        return dict(DEFAULT_CONFIG)
    try:
        with open(CONFIG_FILE) as f:
            cfg = json.load(f)
        merged = dict(DEFAULT_CONFIG)
        merged.update(cfg)
        return merged
    except Exception:
        return dict(DEFAULT_CONFIG)

def save_config(cfg):
    try:
        with open(CONFIG_FILE, "w") as f:
            json.dump(cfg, f, indent=2)
        return True
    except Exception:
        return False

# ── banner ──────────────────────────────────────────────────
def banner():
    print(f"""{PURP}{BOLD}
   ███████╗██╗      ██████╗ ██╗    ██╗██╗  ██╗██╗██████╗ ██╗███╗   ███╗
   ██╔════╝██║     ██╔═══██╗██║    ██║██║ ██╔╝██║██╔══██╗██║████╗ ████║
   █████╗  ██║     ██║   ██║██║ █╗ ██║█████╔╝ ██║██████╔╝██║██╔████╔██║
   ██╔══╝  ██║     ██║   ██║██║███╗██║██╔═██╗ ██║██╔══██╗██║██║╚██╔╝██║
   ██║     ███████╗╚██████╔╝╚███╔███╔╝██║  ██╗██║██║  ██║██║██║ ╚═╝ ██║
   ╚═╝     ╚══════╝ ╚═════╝  ╚══╝╚══╝ ╚═╝  ╚═╝╚═╝╚═╝  ╚═╝╚═╝╚═╝     ╚═╝
{RST}{PURP}              ──  T O O L K I T   v2.0  ──{RST}
{GRAY}              send · webhook · discover
              rate limit: 1.5s / message{RST}
""")

def ask(prompt, default=None):
    suffix = f" {GRAY}[{default}]{RST}" if default is not None else ""
    try:
        return input(f"{RED}  ▸ {RST}{prompt}{suffix}: ").strip()
    except EOFError:
        return ""

def ask_int(prompt, default=None, minimum=None, maximum=None):
    while True:
        raw = ask(prompt, default)
        if raw == "" and default is not None:
            return int(default)
        try:
            v = int(raw)
            if minimum is not None and v < minimum: raise ValueError
            if maximum is not None and v > maximum: raise ValueError
            return v
        except ValueError:
            print(f"{DRED}    ✗ enter int{RST}")

def ask_float(prompt, default=None, minimum=None):
    while True:
        raw = ask(prompt, default)
        if raw == "" and default is not None:
            return float(default)
        try:
            v = float(raw)
            if minimum is not None and v < minimum: raise ValueError
            return v
        except ValueError:
            print(f"{DRED}    ✗ enter number ≥ {minimum}{RST}")

def ask_yn(prompt, default="y"):
    while True:
        raw = ask(prompt + " (y/n)", default).lower()
        if raw in ("y", "yes", "1"): return True
        if raw in ("n", "no", "0"):  return False
        print(f"{DRED}    ✗ y or n{RST}")

def ask_multiline(prompt, default=None):
    print(f"{RED}  ▸ {RST}{prompt}{f' {GRAY}[{default}]{RST}' if default else ''}")
    print(f"{GRAY}    (empty line to finish · '.' to keep default){RST}")
    lines = []
    while True:
        try:
            ln = input(f"{GRAY}    | {RST}")
        except EOFError:
            break
        if ln == "" and lines:
            break
        if ln == "." and default:
            return default
        lines.append(ln)
    out = "\n".join(lines).strip()
    return out if out else (default or "")

# ── URL / payload builders ──────────────────────────────────
def build_url(cfg, phone_62):
    tpl = cfg["url_template"]
    phone_jid = f"{phone_62}@s.whatsapp.net"
    return cfg["api_base"].rstrip("/") + tpl.format(
        phone=phone_62,
        phone_08=("0" + phone_62[2:]) if phone_62.startswith("62") else phone_62,
        phone_jid=phone_jid,
        session=cfg.get("session_id", ""),
        key=cfg["api_key"],
    )

def build_headers(cfg):
    h = {
        "Content-Type": "application/json",
        "Accept": "application/json",
        "User-Agent": "flowkirim-py/2.0",
    }
    h[cfg.get("auth_header", "Authorization")] = f"{cfg.get('auth_prefix', '')}{cfg['api_key']}"
    return h

def build_payload(text):
    # exact shape FlowKirim expects
    return {
        "messageType": "text",
        "messageText": text,
    }

# ── logger ──────────────────────────────────────────────────
class Logger:
    def __init__(self, path, enabled=True, header=True):
        self.path = path
        self.enabled = enabled
        self.lock = threading.Lock()
        if self.enabled and header:
            try:
                with open(self.path, "a") as f:
                    f.write(f"\n=== session {time.strftime('%Y-%m-%d %H:%M:%S')} ===\n")
            except Exception:
                self.enabled = False

    def write(self, line):
        if not self.enabled:
            return
        with self.lock:
            try:
                with open(self.path, "a") as f:
                    f.write(line + "\n")
            except Exception:
                pass

# ── send ────────────────────────────────────────────────────
def send_one(cfg, phone_62, text):
    url = build_url(cfg, phone_62)
    headers = build_headers(cfg)
    payload = build_payload(text)

    t0 = time.time()
    try:
        r = requests.post(url, headers=headers, json=payload, timeout=20)
        ms = (time.time() - t0) * 1000
        body = (r.text or "").replace("\n", " ").replace("\r", " ")[:160]
        return r.status_code, ms, body
    except requests.exceptions.RequestException as e:
        return None, (time.time() - t0) * 1000, str(e)[:120]

def render(phone, code, ms, body, worker=None, logger=None, tag=""):
    ts = time.strftime("%H:%M:%S")
    wtag = f"{BLUE}w{worker:<2}{RST} " if worker is not None else ""
    ttag = f"{PURP}{tag:<10}{RST} " if tag else ""

    if code is None:
        color, ct = DRED + BOLD, "ERR"
    elif 200 <= code < 300:
        color, ct = GREEN + BOLD, str(code)
    elif code == 429:
        color, ct = YEL + BOLD, "429"
    elif 400 <= code < 500:
        color, ct = YEL + BOLD, str(code)
    else:
        color, ct = RED + BOLD, str(code)

    print(f"{GRAY}[{ts}]{RST} {wtag}{ttag}{WHITE}{phone}{RST}  "
          f"{color}{ct:>4}{RST}  {GRAY}{ms:>6.0f}ms{RST}  "
          f"{GRAY}·{RST} {DIM}{body}{RST}")
    if logger:
        logger.write(f"{ts} worker={worker} phone={phone} code={code} ms={ms:.0f} body={body}")

class Stats:
    def __init__(self):
        self.lock = threading.Lock()
        self.total = 0; self.ok = 0; self.fail = 0; self.err = 0
        self.t0 = time.time()

    def hit(self, code):
        with self.lock:
            self.total += 1
            if code is None: self.err += 1
            elif 200 <= code < 300: self.ok += 1
            else: self.fail += 1

    def snapshot(self):
        with self.lock:
            el = max(time.time() - self.t0, 0.001)
            return {"total": self.total, "ok": self.ok, "fail": self.fail,
                    "err": self.err, "rate": self.total / el, "elapsed": el}

    def line(self):
        s = self.snapshot()
        return (f"{GRAY}SENT {WHITE}{s['total']}{GRAY}  ·  "
                f"OK {GREEN}{s['ok']}{GRAY}  ·  "
                f"FAIL {RED}{s['fail']}{GRAY}  ·  "
                f"ERR {DRED}{s['err']}{GRAY}  ·  "
                f"{s['rate']:.1f}/s{RST}")

def run_single(cfg, phone, text, logger):
    print(f"\n{PURP}{BOLD}  ▸▸ FIRING ONCE{RST}")
    print(f"{GRAY}     url: {WHITE}{build_url(cfg, phone)}{RST}\n")
    code, ms, body = send_one(cfg, phone, text)
    render(phone, code, ms, body, worker=1, logger=logger)

def run_loop(cfg, phone, text, delay, workers, total, logger):
    label = "TURBO" if delay == 1 else ("NO DELAY" if delay == 0 else f"{delay}s gap")
    print(f"\n{PURP}{BOLD}  ▸▸ LOOP — {label} · {workers} worker(s){RST}")
    print(f"{GRAY}     url: {WHITE}{build_url(cfg, phone)}{RST}")
    print(f"{GRAY}     ctrl+C to stop{RST}\n")

    stats = Stats()
    stop_flag = threading.Event()
    sent = [0]
    sent_lock = threading.Lock()

    def worker(wid):
        while not stop_flag.is_set():
            if total > 0:
                with sent_lock:
                    if sent[0] >= total: return
                    sent[0] += 1
            code, ms, body = send_one(cfg, phone, text)
            stats.hit(code)
            render(phone, code, ms, body, worker=wid, logger=logger)
            s = stats.snapshot()
            if s["total"] % 10 == 0 and s["total"] > 0:
                print("  " + stats.line())
            if delay not in (0.0, 1.0) and delay > 0:
                time.sleep(delay)

    threads = [threading.Thread(target=worker, args=(i + 1,), daemon=True) for i in range(workers)]
    for t in threads: t.start()
    try:
        while any(t.is_alive() for t in threads):
            time.sleep(0.2)
    except KeyboardInterrupt:
        stop_flag.set()
        print(f"\n\n{RED}{BOLD}  ■ interrupt — draining...{RST}\n")
        for t in threads: t.join(timeout=3)

    s = stats.snapshot()
    print()
    hr()
    print(f"{WHITE}{BOLD}  SESSION SUMMARY{RST}")
    hr()
    print(f"{GRAY}  elapsed      {WHITE}{s['elapsed']:.1f}s{RST}")
    print(f"{GRAY}  total sent   {WHITE}{s['total']}{RST}")
    print(f"{GRAY}  delivered    {GREEN}{s['ok']}{RST}")
    print(f"{GRAY}  failed       {RED}{s['fail']}{RST}")
    print(f"{GRAY}  errors       {DRED}{s['err']}{RST}")
    print(f"{GRAY}  throughput   {WHITE}{s['rate']:.2f} req/s{RST}")
    hr()
    print()

# ── discovery ───────────────────────────────────────────────
DISCOVERY_PATHS = [
    "/api/send/{phone}",
    "/api/send/{phone_jid}",
    "/api/send-message/{phone}",
    "/api/sendMessage/{phone}",
    "/api/sendText/{phone}",
    "/api/send-text/{phone}",
    "/api/message/send/{phone}",
    "/api/messages/send/{phone}",
    "/api/v1/send/{phone}",
    "/api/v1/message/send/{phone}",
    "/api/v1/sendText/{phone}",
    "/api/v2/send/{phone}",
    "/api/whatsapp/send/{phone}",
    "/api/whatsapp/sendMessage/{phone}",
    "/api/wa/send/{phone}",
    "/api/chats/send/{phone}",
    "/api/{session}/send/{phone}",
    "/api/sendMessage?to={phone}",
    "/api/send?to={phone}",
    "/api/send?target={phone}",
    "/send",
    "/sendMessage",
    "/api/send",
    "/api/message",
]

def run_discovery(cfg, phone_62, text, logger):
    print(f"\n{PURP}{BOLD}  ▸▸ DISCOVERY MODE{RST}")
    print(f"{GRAY}     target : {WHITE}{phone_62}{RST}")
    print(f"{GRAY}     base   : {WHITE}{cfg['api_base']}{RST}")
    print(f"{GRAY}     auth   : {WHITE}{cfg['auth_header']}: {cfg['auth_prefix']}{cfg['api_key'][:10]}…{RST}")
    print(f"{GRAY}     body   : {WHITE}{json.dumps(build_payload(text))}{RST}\n")
    print(f"{GRAY}     testing {len(DISCOVERY_PATHS)} path templates…{RST}\n")
    hr()

    wins = []
    headers = build_headers(cfg)
    payload = build_payload(text)
    base = cfg["api_base"].rstrip("/")
    phone_jid = f"{phone_62}@s.whatsapp.net"

    for i, tpl in enumerate(DISCOVERY_PATHS, 1):
        path = tpl.format(
            phone=phone_62, phone_08=phone_62, phone_jid=phone_jid,
            session=cfg.get("session_id", "") or "SESSION", key=cfg["api_key"],
        )
        url = base + path
        short = tpl[:50]

        try:
            t0 = time.time()
            r = requests.post(url, headers=headers, json=payload, timeout=12)
            ms = (time.time() - t0) * 1000
            code = r.status_code
            body = (r.text or "").replace("\n", " ").replace("\r", " ")[:110]

            if 200 <= code < 300:
                color = GREEN + BOLD
                mark = "✓"
                wins.append((tpl, code, body))
            elif code == 404:
                color = GRAY
                mark = "·"
            elif code == 401 or code == 403:
                color = ORNG + BOLD
                mark = "!"
            elif code == 429:
                color = YEL + BOLD
                mark = "~"
            elif 400 <= code < 500:
                color = CYAN
                mark = "?"
            else:
                color = RED + BOLD
                mark = "x"

            print(f"  {color}{mark}{RST} "
                  f"{WHITE}[{i:>2}]{RST} "
                  f"{GRAY}{short:<52}{RST} "
                  f"{color}{code:>4}{RST} "
                  f"{GRAY}{ms:>5.0f}ms{RST} "
                  f"{DIM}{body}{RST}")
            if logger:
                logger.write(f"DISCOVERY {tpl} -> {code} ms={ms:.0f} body={body}")
        except requests.exceptions.RequestException as e:
            print(f"  {DRED}x{RST} {WHITE}[{i:>2}]{RST} "
                  f"{GRAY}{short:<52}{RST} {DRED}ERR{RST} "
                  f"{DIM}{str(e)[:60]}{RST}")

        time.sleep(0.4)

    hr()
    if wins:
        print(f"\n{GREEN}{BOLD}  ✓ {len(wins)} WORKING PATH(S):{RST}\n")
        for tpl, code, body in wins:
            print(f"  {GREEN}●{RST} {WHITE}{tpl}{RST}  "
                  f"{GREEN}{code}{RST}  {DIM}{body[:80]}{RST}")
        print(f"\n{GRAY}  recommended default → url_template = {WHITE}{wins[0][0]}{RST}\n")
        cfg["url_template"] = wins[0][0]
        save_config(cfg)
        print(f"{GREEN}  ✓ saved to config as new default{RST}\n")
    else:
        print(f"\n{DRED}{BOLD}  ✗ no working path found{RST}")
        print(f"{GRAY}     try: check panel → API tab for exact endpoint{RST}")
        print(f"{GRAY}     try: swap auth header to X-API-Key with empty prefix{RST}")
        print(f"{GRAY}     try: add a session id in the config screen{RST}\n")

# ── webhook (flask) ─────────────────────────────────────────
def run_webhook(cfg):
    try:
        from flask import Flask, request, abort
    except ImportError:
        print(f"\n{DRED}  ✗ flask not installed{RST}")
        print(f"{GRAY}    pip install flask{RST}\n")
        return

    port = ask_int("webhook port", default=8080, minimum=1, maximum=65535)
    path = ask("webhook path", "/webhook/flowkirim")
    secret = ask("webhook secret (HMAC)", "changeme")

    app = Flask(__name__)
    logger = Logger(HOOK_FILE, enabled=True)

    @app.post(path)
    def webhook():
        raw = request.get_data()
        received = request.headers.get("X-Flowkirim-Signature", "")
        expected = "sha256=" + hmac.new(
            secret.encode(), raw, hashlib.sha256
        ).hexdigest()

        if secret and not hmac.compare_digest(received, expected):
            print(f"{DRED}[{time.strftime('%H:%M:%S')}] ✗ bad signature{RST}")
            abort(401)

        try:
            payload = request.get_json(force=True, silent=True) or {}
        except Exception:
            payload = {}

        ts = time.strftime("%H:%M:%S")
        phone = payload.get("senderNumber", "?").split("@")[0]
        name  = payload.get("senderName", "?")
        msg   = payload.get("messageText", "")
        sess  = payload.get("sessionId", "?")
        dev   = payload.get("devicePhoneNumber", "?")

        print(f"\n{ORNG}{BOLD}  ◀◀ INBOUND{RST} {GRAY}[{ts}]{RST}")
        print(f"{GRAY}     session  {WHITE}{sess}{RST}")
        print(f"{GRAY}     device   {WHITE}{dev}{RST}")
        print(f"{GRAY}     from     {WHITE}{phone}{RST} {GRAY}({name}){RST}")
        print(f"{GRAY}     text     {WHITE}{msg}{RST}")
        print(f"{GRAY}     msid     {WHITE}{payload.get('messageId', '?')}{RST}")

        logger.write(json.dumps(payload, ensure_ascii=False))
        return "", 200

    print(f"\n{PURP}{BOLD}  ▸▸ WEBHOOK LISTENING{RST}")
    print(f"{GRAY}     url    : {WHITE}http://0.0.0.0:{port}{path}{RST}")
    print(f"{GRAY}     secret : {WHITE}{secret[:6]}…{RST}")
    print(f"{GRAY}     log    : {WHITE}{HOOK_FILE}{RST}")
    print(f"{GRAY}     ctrl+C to stop{RST}\n")
    hr()

    try:
        app.run(host="0.0.0.0", port=port, debug=False, use_reloader=False)
    except KeyboardInterrupt:
        print(f"\n{RED}  webhook stopped.{RST}\n")

# ── screens ─────────────────────────────────────────────────
def screen_main_menu():
    clear(); banner()
    print(f"{WHITE}{BOLD}  SELECT MODE{RST}\n")
    print(f"  {CYAN}[1]{RST} {WHITE}send{RST}       {GRAY}· outbound text to a number{RST}")
    print(f"  {CYAN}[2]{RST} {WHITE}webhook{RST}    {GRAY}· flask listener for inbound{RST}")
    print(f"  {CYAN}[3]{RST} {WHITE}discover{RST}   {GRAY}· probe endpoints (uses your own number){RST}")
    print(f"  {CYAN}[4]{RST} {WHITE}config{RST}     {GRAY}· edit credentials & url template{RST}")
    print(f"  {CYAN}[5]{RST} {WHITE}quit{RST}\n")
    return ask_int("choice", default=1, minimum=1, maximum=5)

def screen_key(cfg):
    clear(); banner()
    print(f"{WHITE}{BOLD}  CONFIG{RST}\n")
    print(f"{GRAY}  key       : {WHITE}{cfg['api_key'][:12]}…{cfg['api_key'][-6:]}{RST}")
    print(f"{GRAY}  base      : {WHITE}{cfg['api_base']}{RST}")
    print(f"{GRAY}  template  : {WHITE}{cfg['url_template']}{RST}")
    print(f"{GRAY}  auth      : {WHITE}{cfg['auth_header']}: {cfg['auth_prefix']}…{RST}")
    print(f"{GRAY}  session   : {WHITE}{cfg['session_id'] or '(none)'}{RST}\n")

    if ask_yn("change API key", "n"):
        k = ask("API key", cfg["api_key"]); cfg["api_key"] = k or cfg["api_key"]
    if ask_yn("change base url", "n"):
        b = ask("base url", cfg["api_base"]); cfg["api_base"] = b or cfg["api_base"]
    if ask_yn("change url template", "n"):
        print(f"{GRAY}    placeholders: {{phone}} {{phone_08}} {{phone_jid}} {{session}} {{key}}{RST}")
        t = ask("template", cfg["url_template"]); cfg["url_template"] = t or cfg["url_template"]
    if ask_yn("change auth header", "n"):
        h = ask("header name", cfg["auth_header"])
        p = ask("header prefix", cfg["auth_prefix"])
        cfg["auth_header"] = h or cfg["auth_header"]
        cfg["auth_prefix"] = p
    if ask_yn("set session id", "y" if cfg["session_id"] else "n"):
        s = ask("session id", cfg["session_id"] or ""); cfg["session_id"] = s
    if ask_yn("set sender name", "y" if cfg["sender_name"] else "n"):
        sn = ask("sender name", cfg["sender_name"] or ""); cfg["sender_name"] = sn

    save_config(cfg)
    print(f"\n{GREEN}  ✓ saved{RST}\n")

def screen_phone(cfg):
    clear(); banner()
    print(f"{WHITE}{BOLD}  TARGET NUMBER{RST}\n")
    print(f"{GRAY}  08xxxxxxxxxx · 62xxxxxxxxxx · +62xxxxxxxxxx{RST}\n")
    while True:
        p = phone_digits(ask("phone"))
        if valid_phone(p):
            print(f"{GREEN}    ✓ {p}{RST}\n"); return p
        print(f"{DRED}    ✗ invalid{RST}\n")

def screen_text(cfg):
    clear(); banner()
    print(f"{WHITE}{BOLD}  MESSAGE BODY{RST}\n")
    print(f"{GRAY}  current: {WHITE}{cfg['text'][:60]}{RST}\n")
    if ask_yn("keep previous", "y"): return cfg["text"]
    t = ask_multiline("new body", cfg["text"])
    return t or cfg["text"]

def screen_delay(cfg):
    clear(); banner()
    print(f"{WHITE}{BOLD}  RATE CONTROL{RST}\n")
    print(f"  {YEL}FlowKirim limits below 1.5s per send{RST}\n")
    return ask_float("delay", default=cfg.get("delay", 1.5), minimum=0)

def screen_workers(cfg):
    clear(); banner()
    print(f"{WHITE}{BOLD}  CONCURRENCY{RST}\n")
    print(f"{GRAY}  >1 worker may 429 fast on single-device gateways{RST}\n")
    return ask_int("worker threads", default=cfg.get("workers", 1), minimum=1, maximum=8)

def screen_total(cfg):
    clear(); banner()
    print(f"{WHITE}{BOLD}  BURST SIZE{RST}\n")
    return ask_int("total sends", default=100, minimum=1)

def screen_send_mode():
    clear(); banner()
    print(f"{WHITE}{BOLD}  SEND MODE{RST}\n")
    print(f"  {CYAN}[1]{RST} single")
    print(f"  {CYAN}[2]{RST} loop  {GRAY}(until stopped){RST}")
    print(f"  {CYAN}[3]{RST} burst {GRAY}(N then exit){RST}\n")
    return {1: "single", 2: "loop", 3: "burst"}[ask_int("choice", default=2, minimum=1, maximum=3)]

def screen_confirm_send(cfg, mode, phone, text, delay, workers, total):
    clear(); banner()
    print(f"{WHITE}{BOLD}  LAUNCH CONFIG{RST}\n")
    hr()
    print(f"{GRAY}  url         {WHITE}{build_url(cfg, phone)}{RST}")
    print(f"{GRAY}  auth        {WHITE}{cfg['auth_header']}: {cfg['auth_prefix']}{cfg['api_key'][:10]}…{RST}")
    print(f"{GRAY}  target      {WHITE}{phone}{RST}")
    print(f"{GRAY}  mode        {WHITE}{mode}{RST}")
    print(f"{GRAY}  delay       {WHITE}{delay}{RST}")
    print(f"{GRAY}  workers     {WHITE}{workers}{RST}")
    if mode == "burst":
        print(f"{GRAY}  total       {WHITE}{total}{RST}")
    hr()
    print(f"{WHITE}{BOLD}  PAYLOAD{RST}")
    print(f"{GRAY}  │ {WHITE}{json.dumps(build_payload(text))}{RST}")
    hr()
    print()
    return ask_yn("confirm launch", "y")

# ── main ────────────────────────────────────────────────────
def main():
    cfg = load_config()

    while True:
        choice = screen_main_menu()

        if choice == 1:
            mode = screen_send_mode()
            phone = screen_phone(cfg)
            text = screen_text(cfg)

            delay, workers, total = cfg.get("delay", 1.5), 1, 0
            if mode in ("loop", "burst"):
                delay = screen_delay(cfg)
                workers = screen_workers(cfg)
                if mode == "burst":
                    total = screen_total(cfg)

            cfg["save_log"] = ask_yn("write session log", "y" if cfg["save_log"] else "n")
            if not screen_confirm_send(cfg, mode, phone, text, delay, workers, total):
                print(f"{DRED}  cancelled.{RST}\n"); time.sleep(0.6); continue

            save_config(cfg)
            logger = Logger(LOG_FILE, enabled=cfg["save_log"])
            if mode == "single": run_single(cfg, phone, text, logger)
            else: run_loop(cfg, phone, text, delay, workers, total, logger)
            input(f"{GRAY}  press enter to continue…{RST}")

        elif choice == 2:
            run_webhook(cfg)

        elif choice == 3:
            clear(); banner()
            print(f"{WHITE}{BOLD}  DISCOVERY{RST}\n")
            print(f"{GRAY}  uses your own number as the test target — do not point this")
            print(f"  at a stranger's phone, it will send real messages if any")
            print(f"  endpoint succeeds.{RST}\n")
            if not ask_yn("continue", "y"): continue

            phone = screen_phone(cfg)
            text = ask("test message", "flowkirim discovery test")
            logger = Logger(LOG_FILE, enabled=True)
            run_discovery(cfg, phone, text, logger)
            input(f"{GRAY}  press enter to continue…{RST}")

        elif choice == 4:
            screen_key(cfg)

        else:
            print(f"\n{RED}  exiting.{RST}\n")
            sys.exit(0)

def _sigint(sig, frame):
    raise KeyboardInterrupt

if __name__ == "__main__":
    signal.signal(signal.SIGINT, _sigint)
    try:
        main()
    except KeyboardInterrupt:
        print(f"\n{RED}  aborted.{RST}\n")
        sys.exit(0)
