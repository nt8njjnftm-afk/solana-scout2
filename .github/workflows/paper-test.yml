#!/usr/bin/env python3
"""Solana Scout AutoTrader — conservative starter with paper mode by default.

IMPORTANT:
- PAPER_MODE defaults to true. Do not switch to live trading until paper-tested.
- Live mode requires a dedicated low-balance wallet, Jupiter API key, RPC URL,
  and a private key stored only in the hosting provider's secret/environment store.
- Never put a private key or .env file in GitHub, a website, or client-side code.
- Token checks reduce risk but cannot make meme coins safe or guarantee exits.

Install:
    python -m pip install requests solders
Run:
    python solana_scout_autotrader.py

Environment variables (all optional unless live mode is enabled):
    PAPER_MODE=true                 # default; set false only after testing
    TRADE_SIZE_USD=5                # intended position size
    MAX_OPEN_BUDGET_USD=25          # max cost basis across open positions
    TAKE_PROFIT_PCT=20              # exit trigger relative to entry
    STOP_LOSS_PCT=10                # exit trigger relative to entry
    MAX_SLIPPAGE_BPS=150            # 1.50% max slippage requested from Jupiter
    MIN_LIQUIDITY_USD=8000
    MIN_VOLUME_5M_USD=500
    MIN_VOLUME_1H_USD=3000
    MIN_AGE_MINUTES=2
    MAX_AGE_HOURS=6
    ALERT_SCORE=55
    SCAN_INTERVAL_SECONDS=45
    MAX_OPEN_POSITIONS=5
    MIN_SOL_RESERVE=0.02
    JUPITER_API_KEY=...             # required live
    SOLANA_RPC_URL=...              # required live; use a trusted RPC provider
    SOLANA_PRIVATE_KEY=...          # JSON array or base58; live only; NEVER commit
    PAUSE_BUYS=true                 # emergency pause for new entries; exits continue
    KILL_SWITCH_FILE=STOP_TRADING   # create this file to pause new entries

State is saved locally in solana_scout_autotrader_state.json. Back up the state
file privately; do not commit it if it contains wallet/position details.
"""

import base64
import json
import os
import time
from datetime import datetime
from pathlib import Path
from typing import Any, Dict, List, Optional, Tuple

import requests

DEX = "https://api.dexscreener.com"
JUP = "https://api.jup.ag/swap/v2"
RUGCHECK = "https://api.rugcheck.xyz/v1/tokens"
SOL_MINT = "So11111111111111111111111111111111111111112"
LAMPORTS = 1_000_000_000

PAPER_MODE = os.getenv("PAPER_MODE", "true").strip().lower() not in {"0", "false", "no"}
TRADE_SIZE_USD = max(1.0, float(os.getenv("TRADE_SIZE_USD", "5")))
MAX_OPEN_BUDGET_USD = max(TRADE_SIZE_USD, float(os.getenv("MAX_OPEN_BUDGET_USD", "25")))
TAKE_PROFIT_PCT = max(1.0, float(os.getenv("TAKE_PROFIT_PCT", "20")))
STOP_LOSS_PCT = max(1.0, float(os.getenv("STOP_LOSS_PCT", "10")))
MAX_SLIPPAGE_BPS = min(300, max(10, int(os.getenv("MAX_SLIPPAGE_BPS", "150"))))
MIN_LIQUIDITY = float(os.getenv("MIN_LIQUIDITY_USD", "8000"))
MIN_VOL5 = float(os.getenv("MIN_VOLUME_5M_USD", "500"))
MIN_VOL1H = float(os.getenv("MIN_VOLUME_1H_USD", "3000"))
MIN_AGE_MIN = float(os.getenv("MIN_AGE_MINUTES", "2"))
MAX_AGE_HOURS = float(os.getenv("MAX_AGE_HOURS", "6"))
ALERT_SCORE = int(os.getenv("ALERT_SCORE", "55"))
INTERVAL = max(30, int(os.getenv("SCAN_INTERVAL_SECONDS", "45")))
MAX_OPEN_POSITIONS = max(1, int(os.getenv("MAX_OPEN_POSITIONS", "5")))
MIN_SOL_RESERVE = max(0.01, float(os.getenv("MIN_SOL_RESERVE", "0.02")))
PAUSE_BUYS = os.getenv("PAUSE_BUYS", "false").strip().lower() in {"1", "true", "yes"}
KILL_SWITCH_FILE = Path(os.getenv("KILL_SWITCH_FILE", "STOP_TRADING"))
STATE_FILE = Path(os.getenv("STATE_FILE", "solana_scout_autotrader_state.json"))

S = requests.Session()
S.headers.update({"User-Agent": "SolanaScoutAutoTrader/1.0", "Accept": "application/json"})


def log(message: str) -> None:
    print(f"[{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}] {message}", flush=True)


def request_json(method: str, url: str, **kwargs: Any) -> Optional[Dict[str, Any]]:
    try:
        response = S.request(method, url, timeout=20, **kwargs)
        if response.status_code == 429:
            log(f"Rate limited by {url.split('/')[2]}; skipping this request.")
            return None
        response.raise_for_status()
        data = response.json()
        return data if isinstance(data, dict) else {"_list": data} if isinstance(data, list) else None
    except (requests.RequestException, ValueError) as exc:
        log(f"Request failed ({url.split('/')[2]}): {exc}")
        return None


def default_state() -> Dict[str, Any]:
    return {"open_positions": {}, "closed_trades": [], "realized_pnl_usd": 0.0, "updated_at": None}


def load_state() -> Dict[str, Any]:
    try:
        data = json.loads(STATE_FILE.read_text(encoding="utf-8"))
        if not isinstance(data, dict):
            raise ValueError("state is not an object")
        data.setdefault("open_positions", {})
        data.setdefault("closed_trades", [])
        data.setdefault("realized_pnl_usd", 0.0)
        return data
    except (OSError, ValueError, TypeError):
        return default_state()


def save_state(state: Dict[str, Any]) -> None:
    state["updated_at"] = datetime.now().isoformat(timespec="seconds")
    tmp = STATE_FILE.with_suffix(STATE_FILE.suffix + ".tmp")
    tmp.write_text(json.dumps(state, indent=2), encoding="utf-8")
    tmp.replace(STATE_FILE)


def as_float(value: Any) -> float:
    try:
        return float(value or 0)
    except (TypeError, ValueError):
        return 0.0


def get_list(url: str) -> List[Dict[str, Any]]:
    try:
        response = S.get(url, timeout=15)
        if response.status_code == 429:
            log("DexScreener rate limit; waiting until next scan.")
            return []
        response.raise_for_status()
        data = response.json()
        return data if isinstance(data, list) else []
    except (requests.RequestException, ValueError) as exc:
        log(f"Market-data request failed: {exc}")
        return []


def token_candidates() -> List[str]:
    found = set()
    for endpoint in ("token-profiles/latest/v1", "token-boosts/latest/v1"):
        for item in get_list(f"{DEX}/{endpoint}"):
            if item.get("chainId") == "solana" and item.get("tokenAddress"):
                found.add(str(item["tokenAddress"]))
    return list(found)


def get_pairs(mint: str) -> List[Dict[str, Any]]:
    try:
        response = S.get(f"{DEX}/token-pairs/v1/solana/{mint}", timeout=15)
        if response.status_code == 429:
            return []
        response.raise_for_status()
        data = response.json()
        return data if isinstance(data, list) else []
    except (requests.RequestException, ValueError) as exc:
        log(f"Could not load pairs for {mint[:6]}…: {exc}")
        return []


def age_minutes(pair: Dict[str, Any]) -> Optional[float]:
    created = pair.get("pairCreatedAt")
    try:
        return max(0.0, (time.time() * 1000 - float(created)) / 60000) if created else None
    except (TypeError, ValueError):
        return None


def evaluate_pair(pair: Dict[str, Any]) -> Tuple[int, List[str], Dict[str, float]]:
    liquidity = as_float((pair.get("liquidity") or {}).get("usd"))
    volume = pair.get("volume") or {}
    vol5, vol1h = as_float(volume.get("m5")), as_float(volume.get("h1"))
    tx5 = ((pair.get("txns") or {}).get("m5") or {})
    buys, sells = as_float(tx5.get("buys")), as_float(tx5.get("sells"))
    change5 = as_float((pair.get("priceChange") or {}).get("m5"))
    change1h = as_float((pair.get("priceChange") or {}).get("h1"))
    age = age_minutes(pair)
    score, reasons = 0, []
    if liquidity >= MIN_LIQUIDITY:
        score += 25; reasons.append(f"liquidity ${liquidity:,.0f}")
    if vol5 >= MIN_VOL5:
        score += 15; reasons.append(f"5m volume ${vol5:,.0f}")
    if vol1h >= MIN_VOL1H:
        score += 10; reasons.append(f"1h volume ${vol1h:,.0f}")
    trades = buys + sells
    if trades >= 10:
        score += 8; reasons.append(f"{int(trades)} trades/5m")
    if trades and buys / trades >= 0.60:
        score += 12; reasons.append(f"buy pressure {buys / trades:.0%}")
    if 0 < change5 <= 8:
        score += 15; reasons.append(f"early 5m momentum +{change5:.1f}%")
    elif -2 <= change5 <= 0:
        score += 5; reasons.append(f"5m consolidation {change5:.1f}%")
    if 0 < change1h <= 35:
        score += 8; reasons.append(f"1h trend +{change1h:.1f}%")
    if age is not None and MIN_AGE_MIN <= age <= MAX_AGE_HOURS * 60:
        score += 7; reasons.append(f"pair age {age:.0f}m")
    metrics = {"liquidity": liquidity, "vol5": vol5, "vol1h": vol1h,
               "buys": buys, "sells": sells, "change5": change5,
               "change1h": change1h, "age": age if age is not None else -1,
               "price": as_float(pair.get("priceUsd"))}
    return min(score, 100), reasons, metrics


def qualifies(pair: Dict[str, Any], score: int, m: Dict[str, float]) -> bool:
    base = pair.get("baseToken") or {}
    # Only buy the mint represented by the pair's base token so priceUsd is for that mint.
    if not base.get("address") or not pair.get("pairAddress"):
        return False
    if m["price"] <= 0 or m["liquidity"] < MIN_LIQUIDITY or m["vol5"] < MIN_VOL5 or m["vol1h"] < MIN_VOL1H:
        return False
    if m["age"] < MIN_AGE_MIN or m["age"] > MAX_AGE_HOURS * 60:
        return False
    if m["change5"] > 12 or m["change1h"] > 60 or m["buys"] + m["sells"] < 5:
        return False
    return score >= ALERT_SCORE


def live_config() -> Tuple[Any, str, str]:
    api_key = os.getenv("JUPITER_API_KEY", "").strip()
    rpc_url = os.getenv("SOLANA_RPC_URL", "").strip()
    private_key = os.getenv("SOLANA_PRIVATE_KEY", "").strip()
    if not (api_key and rpc_url and private_key):
        raise RuntimeError("Live mode needs JUPITER_API_KEY, SOLANA_RPC_URL, and SOLANA_PRIVATE_KEY set as private environment secrets.")
    try:
        from solders.keypair import Keypair
    except ImportError as exc:
        raise RuntimeError("Install solders with: python -m pip install solders") from exc
    try:
        if private_key.startswith("["):
            keypair = Keypair.from_bytes(bytes(json.loads(private_key)))
        else:
            keypair = Keypair.from_base58_string(private_key)
    except Exception as exc:
        raise RuntimeError("SOLANA_PRIVATE_KEY must be a valid base58 key or JSON byte-array keypair.") from exc
    return keypair, api_key, rpc_url


def rpc_call(rpc_url: str, method: str, params: list) -> Optional[Dict[str, Any]]:
    data = request_json("POST", rpc_url, json={"jsonrpc": "2.0", "id": 1, "method": method, "params": params})
    if not data or data.get("error"):
        return None
    result = data.get("result")
    return result if isinstance(result, dict) else None


def sol_price_usd() -> float:
    data = request_json("GET", f"{DEX}/latest/dex/tokens/{SOL_MINT}")
    pairs = (data or {}).get("pairs") or []
    prices = [as_float(p.get("priceUsd")) for p in pairs if p.get("chainId") == "solana" and as_float(p.get("priceUsd")) > 0]
    return sorted(prices)[len(prices)//2] if prices else 0.0


def rugcheck_passes(mint: str) -> Tuple[bool, str]:
    """Fail closed: if the risk endpoint is unavailable or unclear, do not buy."""
    report = request_json("GET", f"{RUGCHECK}/{mint}/report/summary")
    if not report or "risks" not in report or not isinstance(report["risks"], list):
        return False, "RugCheck report unavailable or unrecognized; fail-closed"
    risks = report["risks"]
    severe = []
    for risk in risks:
        if not isinstance(risk, dict):
            return False, "Unrecognized risk report; fail-closed"
        level = str(risk.get("level", risk.get("severity", ""))).lower()
        if level in {"danger", "critical", "high", "warning", "warn"}:
            severe.append(str(risk.get("name") or risk.get("description") or level))
    if severe:
        return False, "RugCheck risk flags: " + ", ".join(severe[:4])
    return True, f"RugCheck report passed basic severity filter (score={report.get('score', 'n/a')})"


def get_token_decimals(rpc_url: str, mint: str) -> Optional[int]:
    result = rpc_call(rpc_url, "getTokenSupply", [mint, {"commitment": "confirmed"}])
    try:
        return int(result["value"]["decimals"]) if result else None
    except (KeyError, TypeError, ValueError):
        return None


def wallet_sol(rpc_url: str, pubkey: str) -> Optional[float]:
    result = rpc_call(rpc_url, "getBalance", [pubkey, {"commitment": "confirmed"}])
    try:
        return int(result["value"]) / LAMPORTS if result else None
    except (KeyError, TypeError, ValueError):
        return None


def jupiter_swap(keypair: Any, api_key: str, input_mint: str, output_mint: str, amount: int) -> Dict[str, Any]:
    """Request, sign, and execute a Jupiter v2 order. Signs only our wallet's slot."""
    from solders.message import to_bytes_versioned
    from solders.transaction import VersionedTransaction

    params = {"inputMint": input_mint, "outputMint": output_mint, "amount": str(amount),
              "taker": str(keypair.pubkey()), "slippageBps": str(MAX_SLIPPAGE_BPS)}
    order = request_json("GET", f"{JUP}/order", params=params, headers={"x-api-key": api_key})
    if not order or not order.get("transaction") or not order.get("requestId"):
        raise RuntimeError(f"Jupiter did not return an executable order: {(order or {}).get('errorMessage', 'no response')}")
    tx = VersionedTransaction.from_bytes(base64.b64decode(order["transaction"]))
    keys = list(tx.message.account_keys)
    try:
        signer_index = keys.index(keypair.pubkey())
    except ValueError as exc:
        raise RuntimeError("Wallet is not a required signer in Jupiter transaction; refusing to sign.") from exc
    signatures = list(tx.signatures)
    signatures[signer_index] = keypair.sign_message(to_bytes_versioned(tx.message))
    signed_tx = VersionedTransaction.populate(tx.message, signatures)
    body = {"signedTransaction": base64.b64encode(bytes(signed_tx)).decode("ascii"), "requestId": order["requestId"]}
    result = request_json("POST", f"{JUP}/execute", json=body, headers={"x-api-key": api_key, "Content-Type": "application/json"})
    if not result or result.get("status") != "Success" or int(result.get("code", -1)) != 0:
        raise RuntimeError(f"Jupiter execution failed: {(result or {}).get('error', 'unknown error')} code={(result or {}).get('code')}")
    return result


def open_budget(state: Dict[str, Any]) -> float:
    return sum(as_float(p.get("cost_usd")) for p in state["open_positions"].values())


def manage_positions(state: Dict[str, Any], live: bool, live_parts: Optional[Tuple[Any, str, str]]) -> None:
    for mint, pos in list(state["open_positions"].items()):
        pairs = get_pairs(mint)
        matching = [p for p in pairs if str((p.get("baseToken") or {}).get("address", "")) == mint]
        if not matching:
            log(f"Cannot price open position {mint[:6]}…; leaving it open (no blind sell).")
            continue
        matching.sort(key=lambda p: as_float((p.get("liquidity") or {}).get("usd")), reverse=True)
        pair = matching[0]
        price = as_float(pair.get("priceUsd"))
        if price <= 0:
            continue
        entry = as_float(pos.get("entry_price"))
        if entry <= 0:
            continue
        change_pct = (price / entry - 1.0) * 100
        trigger = "take-profit" if change_pct >= TAKE_PROFIT_PCT else "stop-loss" if change_pct <= -STOP_LOSS_PCT else ""
        log(f"Position {mint[:6]}… mark {change_pct:+.1f}%" + (f" -> {trigger}" if trigger else ""))
        if not trigger:
            continue
        if not live:
            pnl = as_float(pos.get("cost_usd")) * change_pct / 100
            state["realized_pnl_usd"] = as_float(state.get("realized_pnl_usd")) + pnl
            state["closed_trades"].append({"mint": mint, "reason": trigger, "pnl_usd": pnl, "change_pct": change_pct, "closed_at": datetime.now().isoformat(timespec="seconds"), "mode": "paper"})
            del state["open_positions"][mint]
            save_state(state)
            log(f"PAPER EXIT {trigger}: {mint[:6]}… approximate P/L ${pnl:+.2f}")
            continue
        keypair, api_key, rpc_url = live_parts  # type: ignore[misc]
        try:
            result = jupiter_swap(keypair, api_key, mint, SOL_MINT, int(pos["token_amount_raw"]))
            # Approximate P/L based on price trigger and original cost; realized SOL proceeds can differ.
            pnl = as_float(pos.get("cost_usd")) * change_pct / 100
            state["realized_pnl_usd"] = as_float(state.get("realized_pnl_usd")) + pnl
            state["closed_trades"].append({"mint": mint, "reason": trigger, "pnl_usd_estimate": pnl, "change_pct": change_pct, "signature": result.get("signature"), "closed_at": datetime.now().isoformat(timespec="seconds"), "mode": "live"})
            del state["open_positions"][mint]
            save_state(state)
            log(f"LIVE EXIT sent/confirmed: {trigger}; tx https://solscan.io/tx/{result.get('signature', '')}")
        except Exception as exc:
            log(f"EXIT FAILED for {mint[:6]}…; position kept for retry: {exc}")


def try_entry(mint: str, pair: Dict[str, Any], score: int, reasons: List[str], m: Dict[str, float], state: Dict[str, Any], live_parts: Optional[Tuple[Any, str, str]]) -> bool:
    if mint in state["open_positions"]:
        return False
    if len(state["open_positions"]) >= MAX_OPEN_POSITIONS:
        return False
    if open_budget(state) + TRADE_SIZE_USD > MAX_OPEN_BUDGET_USD + 1e-6:
        return False
    if PAUSE_BUYS or KILL_SWITCH_FILE.exists():
        log("New entries paused (PAUSE_BUYS or kill-switch file). Protective exits remain enabled.")
        return False

    entry_price = m["price"]
    if PAPER_MODE:
        token_units = TRADE_SIZE_USD / entry_price
        state["open_positions"][mint] = {"mint": mint, "symbol": (pair.get("baseToken") or {}).get("symbol", "?"),
            "entry_price": entry_price, "token_amount_raw": None, "token_amount": token_units,
            "cost_usd": TRADE_SIZE_USD, "opened_at": datetime.now().isoformat(timespec="seconds"),
            "score": score, "reasons": reasons, "mode": "paper"}
        save_state(state)
        log(f"PAPER BUY ${TRADE_SIZE_USD:.2f} {state['open_positions'][mint]['symbol']} score={score}; reasons: {', '.join(reasons)}")
        return True

    keypair, api_key, rpc_url = live_parts  # type: ignore[misc]
    ok, safety_message = rugcheck_passes(mint)
    if not ok:
        log(f"LIVE BUY BLOCKED {mint[:6]}…: {safety_message}")
        return False
    sol_usd = sol_price_usd()
    if sol_usd <= 0:
        log("LIVE BUY BLOCKED: cannot establish SOL/USD price.")
        return False
    balance = wallet_sol(rpc_url, str(keypair.pubkey()))
    if balance is None or balance < MIN_SOL_RESERVE + (TRADE_SIZE_USD / sol_usd):
        log(f"LIVE BUY BLOCKED: wallet balance unavailable/too low; preserve at least {MIN_SOL_RESERVE:.3f} SOL for fees.")
        return False
    decimals = get_token_decimals(rpc_url, mint)
    if decimals is None:
        log("LIVE BUY BLOCKED: token decimals could not be verified.")
        return False
    lamports = int((TRADE_SIZE_USD / sol_usd) * LAMPORTS)
    try:
        result = jupiter_swap(keypair, api_key, SOL_MINT, mint, lamports)
        out_raw = int(result.get("totalOutputAmount") or 0)
        if out_raw <= 0:
            raise RuntimeError("Jupiter reported zero output token amount.")
        token_units = out_raw / (10 ** decimals)
        actual_entry = TRADE_SIZE_USD / token_units if token_units > 0 else entry_price
        state["open_positions"][mint] = {"mint": mint, "symbol": (pair.get("baseToken") or {}).get("symbol", "?"),
            "entry_price": actual_entry, "token_amount_raw": str(out_raw), "token_amount": token_units,
            "token_decimals": decimals, "cost_usd": TRADE_SIZE_USD,
            "opened_at": datetime.now().isoformat(timespec="seconds"), "score": score,
            "reasons": reasons, "mode": "live", "buy_signature": result.get("signature")}
        save_state(state)
        log(f"LIVE BUY success: {state['open_positions'][mint]['symbol']} tx https://solscan.io/tx/{result.get('signature', '')}")
        return True
    except Exception as exc:
        log(f"LIVE BUY failed for {mint[:6]}…: {exc}")
        return False


def main() -> None:
    live_parts = None
    if not PAPER_MODE:
        live_parts = live_config()
        keypair, _, _ = live_parts
        log(f"LIVE MODE ENABLED for wallet {keypair.pubkey()} — use only a dedicated low-balance wallet.")
    else:
        log("PAPER MODE: no wallet is loaded and no real transactions will be signed.")
    log(f"Settings: ${TRADE_SIZE_USD:.2f}/entry, max ${MAX_OPEN_BUDGET_USD:.2f} open exposure, TP +{TAKE_PROFIT_PCT:.1f}%, SL -{STOP_LOSS_PCT:.1f}%, slippage cap {MAX_SLIPPAGE_BPS} bps.")
    log("Safety note: market filters and third-party risk reports are imperfect; token sells can fail or have no liquidity.")
    state = load_state()
    while True:
        try:
            manage_positions(state, not PAPER_MODE, live_parts)
            candidates = token_candidates()
            ranked = []
            for mint in candidates:
                if mint in state["open_positions"]:
                    continue
                pairs = get_pairs(mint)
                pairs = [p for p in pairs if str((p.get("baseToken") or {}).get("address", "")) == mint]
                pairs.sort(key=lambda p: as_float((p.get("liquidity") or {}).get("usd")), reverse=True)
                for pair in pairs[:2]:
                    score, reasons, metrics = evaluate_pair(pair)
                    if qualifies(pair, score, metrics):
                        ranked.append((score, mint, pair, reasons, metrics))
            ranked.sort(key=lambda item: item[0], reverse=True)
            log(f"Scanned {len(candidates)} Solana tokens; {len(ranked)} met entry filters; {len(state['open_positions'])} positions open; exposure ${open_budget(state):.2f}.")
            for score, mint, pair, reasons, metrics in ranked:
                if len(state["open_positions"]) >= MAX_OPEN_POSITIONS or open_budget(state) + TRADE_SIZE_USD > MAX_OPEN_BUDGET_USD + 1e-6:
                    break
                try_entry(mint, pair, score, reasons, metrics, state, live_parts)
                time.sleep(1.0)
            save_state(state)
        except KeyboardInterrupt:
            log("Stopped by user.")
            break
        except Exception as exc:
            log(f"Scan loop error (will retry next cycle): {exc}")
        time.sleep(INTERVAL)


if __name__ == "__main__":
    main()
