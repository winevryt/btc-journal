#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
공장장의 비트코인일지 — 22:00 KST 스냅샷 수집기  v1.0 (2026-08)

설계 원칙
  1) 모든 값은 '나중에 같은 요청으로 같은 답이 나오는' 소스에서만 받는다.
  2) 실패한 항목은 절대 직전값을 이월하지 않는다. null + error 로 남긴다.
  3) 가격과 30주선은 반드시 같은 소스(Coinbase BTC-USD)에서 나온다.
     → 베이시스 문제가 구조적으로 발생할 수 없다.

사용법
    python3 collect.py                 # 오늘 22:00 KST 기준
    python3 collect.py 2026-08-14      # 특정 날짜 소급 수집
    python3 collect.py --selftest      # 네트워크 없이 계산 로직만 검증

출력
    snapshots/YYYY-MM-DDT2200KST.json
"""

import sys, os, json, math, csv, io, re
import datetime as dt
import urllib.request, urllib.error

KST = dt.timezone(dt.timedelta(hours=9))
UTC = dt.timezone.utc
SNAP_DIR = os.path.join(os.path.dirname(os.path.abspath(__file__)), "snapshots")
UA = {"User-Agent": "macrofactory-btc-journal/1.0"}
TIMEOUT = 30

# 주 경계 정의(명시적 고정): 월요일 00:00 UTC ~ 일요일 23:59 UTC
# 주봉 종가 = 그 주 마지막 일봉(UTC) 종가
WEEK_START_WEEKDAY = 0  # Monday
SMA_WEEKS = 30


# ────────────────────────────── 공통 ──────────────────────────────

def _get(url, as_json=False, timeout=TIMEOUT, retries=3):
    """일시적 오류(타임아웃·5xx)는 재시도. 4xx는 즉시 포기(요청이 잘못된 것)."""
    last = None
    for i in range(retries):
        try:
            req = urllib.request.Request(url, headers=UA)
            with urllib.request.urlopen(req, timeout=timeout) as r:
                raw = r.read()
            return json.loads(raw.decode("utf-8")) if as_json else raw.decode("utf-8", "replace")
        except urllib.error.HTTPError as e:
            if 400 <= e.code < 500:
                raise
            last = e
        except Exception as e:
            last = e
        if i < retries - 1:
            import time
            time.sleep(2 * (i + 1))
    raise last


def field(value=None, source=None, asof=None, method=None, error=None):
    """모든 항목의 공통 봉투. verified=False면 카드에서 ★ 또는 [미측정]."""
    return {
        "value": value,
        "source": source,
        "asof": asof,
        "method": method,
        "error": error,
        "verified": value is not None and error is None,
    }


def safe(fn, *a, **kw):
    try:
        return fn(*a, **kw)
    except Exception as e:
        return field(error=f"{type(e).__name__}: {e}")


# ───────────────────────── 1. 가격 (Coinbase) ─────────────────────────

CB = "https://api.exchange.coinbase.com"


def cb_candles(granularity, start, end):
    """Coinbase Exchange 캔들. 반환 원소: [time, low, high, open, close, volume]"""
    url = (f"{CB}/products/BTC-USD/candles?granularity={granularity}"
           f"&start={start.astimezone(UTC).strftime('%Y-%m-%dT%H:%M:%SZ')}"
           f"&end={end.astimezone(UTC).strftime('%Y-%m-%dT%H:%M:%SZ')}")
    return _get(url, as_json=True)


def resolve_ref_date(ref_date):
    """22:00 KST가 아직 지나지 않았으면 전일로 물린다.
    낮에 수동 실행해도 미래 캔들을 요청하지 않게 하는 안전장치."""
    cutoff = dt.datetime.combine(ref_date, dt.time(22, 0), tzinfo=KST)
    now = dt.datetime.now(KST)
    if now < cutoff + dt.timedelta(minutes=2):
        newd = ref_date - dt.timedelta(days=1)
        print(f"[안내] {ref_date} 22:00 KST 미도래(현재 {now:%H:%M} KST) → {newd} 기준으로 수집한다.")
        return newd
    return ref_date


def fetch_price_2200kst(ref_date):
    """22:00 KST = 13:00 UTC. 그 시각을 종료로 하는 1시간봉 종가."""
    end = dt.datetime.combine(ref_date, dt.time(22, 0), tzinfo=KST)
    start = end - dt.timedelta(hours=1)
    rows = cb_candles(3600, start, end)
    if not rows:
        raise RuntimeError("no hourly candle returned")
    # 21:00~22:00 KST 봉의 시작 timestamp = 12:00 UTC
    want = int((end - dt.timedelta(hours=1)).timestamp())
    row = next((r for r in rows if int(r[0]) == want), None)
    if row is None:
        row = sorted(rows, key=lambda r: int(r[0]))[-1]
    return field(
        value=round(float(row[4]), 2),
        source="Coinbase Exchange /products/BTC-USD/candles granularity=3600",
        asof=dt.datetime.fromtimestamp(int(row[0]), UTC).isoformat(),
        method="21:00–22:00 KST(12:00–13:00 UTC) 1시간봉 종가",
    )


def fetch_weekly_closes(ref_date, weeks=SMA_WEEKS + 2):
    """완료된 주봉 종가 목록(오래된 것 → 최근). 현재 진행 중인 주는 제외."""
    end = dt.datetime.combine(ref_date, dt.time(23, 59), tzinfo=KST)
    start = end - dt.timedelta(days=weeks * 7 + 10)
    rows = cb_candles(86400, start, end)      # 최대 300개 → 약 300일, 충분
    if not rows:
        raise RuntimeError("no daily candles returned")
    rows.sort(key=lambda r: int(r[0]))

    buckets = {}                               # 주 시작일(UTC date) -> (마지막 일자, 종가)
    for r in rows:
        d = dt.datetime.fromtimestamp(int(r[0]), UTC).date()
        wk = d - dt.timedelta(days=(d.weekday() - WEEK_START_WEEKDAY) % 7)
        prev = buckets.get(wk)
        if prev is None or d > prev[0]:
            buckets[wk] = (d, float(r[4]))

    ref_utc = dt.datetime.combine(ref_date, dt.time(13, 0), tzinfo=KST).astimezone(UTC).date()
    cur_wk = ref_utc - dt.timedelta(days=(ref_utc.weekday() - WEEK_START_WEEKDAY) % 7)

    done = [(wk, v[1]) for wk, v in sorted(buckets.items()) if wk < cur_wk]
    return field(
        value=[{"week_start": wk.isoformat(), "close": round(c, 2)} for wk, c in done[-(SMA_WEEKS + 1):]],
        source="Coinbase Exchange /products/BTC-USD/candles granularity=86400",
        asof=ref_utc.isoformat(),
        method=f"주 경계 월 00:00 UTC~일 23:59 UTC · 주봉 종가=그 주 마지막 일봉 종가 · 진행 주({cur_wk}) 제외",
    )


def compute_sma30(weekly_closes, price_now):
    """SMA30 = (완료 29주 종가 합 + 현재가) / 30  — 기존 정의와 동일."""
    closes = [w["close"] for w in weekly_closes][-(SMA_WEEKS - 1):]
    if len(closes) < SMA_WEEKS - 1:
        raise RuntimeError(f"주봉 부족: {len(closes)}/{SMA_WEEKS-1}")
    s = sum(closes)
    sma = (s + price_now) / SMA_WEEKS
    return {
        "sma30": round(sma, 2),
        "completed_sum_29w": round(s, 2),
        "divergence_pct": round((price_now / sma - 1) * 100, 3),
        # 이번 주 30주선 교차에 필요한 가격 (P = S/(N-1), 항등식)
        "cross_price": round(s / (SMA_WEEKS - 1), 2),
    }


def compute_sma_4w_ago(weekly_closes):
    """v1.1 기울기 조건: 4주 전 SMA30(완료 주봉만으로 산출) 대비 현재 방향."""
    closes = [w["close"] for w in weekly_closes]
    if len(closes) < SMA_WEEKS + 4:
        return None
    win = closes[-(SMA_WEEKS + 4):-4]
    return round(sum(win) / len(win), 2)


# ───────────────────────── 2. TIPS 실질금리 ─────────────────────────

TREAS = ("https://home.treasury.gov/resource-center/data-chart-center/interest-rates/"
         "daily-treasury-rates.csv/{y}/all?type=daily_treasury_real_yield_curve"
         "&field_tdr_date_value={y}&page&_format=csv")


def _treasury_year(y):
    out = []
    for row in csv.DictReader(io.StringIO(_get(TREAS.format(y=y)))):
        key = next((k for k in row if k and k.strip().upper().startswith("10")), None)
        if not key:
            continue
        v = (row[key] or "").strip()
        if not v or v.upper() == "N/A":
            continue
        d = dt.datetime.strptime(row["Date"].strip(), "%m/%d/%Y").date()
        out.append((d, float(v)))
    return out


def fetch_tips(ref_date, half=10):
    """현재값 = 직전 미 정규장 마감 확정치. 앵커 = 6개월 전 전후 (2*half+1)영업일 평균."""
    series = sorted(set(_treasury_year(ref_date.year) + _treasury_year(ref_date.year - 1)))
    if not series:
        raise RuntimeError("treasury series empty")

    past = [x for x in series if x[0] < ref_date]        # 발행일 기준 1영업일 래그
    if not past:
        raise RuntimeError("no confirmed observation before ref_date")
    cur_date, cur_val = past[-1]

    # 6개월 전 목표일에 가장 가까운 영업일
    m = cur_date.month - 6
    y = cur_date.year + (m - 1) // 12
    m = (m - 1) % 12 + 1
    day = min(cur_date.day, [31, 29 if y % 4 == 0 and (y % 100 or y % 400 == 0) else 28,
                             31, 30, 31, 30, 31, 31, 30, 31, 30, 31][m - 1])
    target = dt.date(y, m, day)
    idx = min(range(len(series)), key=lambda i: abs((series[i][0] - target).days))

    lo, hi = max(0, idx - half), min(len(series), idx + half + 1)
    win = series[lo:hi]
    if len(win) < 2 * half + 1:
        raise RuntimeError(f"앵커 창 부족: {len(win)}/{2*half+1}")
    anchor = sum(v for _, v in win) / len(win)

    return field(
        value={
            "current": cur_val,
            "current_date": cur_date.isoformat(),
            "anchor_smoothed": round(anchor, 4),
            "anchor_center_date": series[idx][0].isoformat(),
            "anchor_point_value": series[idx][1],
            "anchor_window": [[d.isoformat(), v] for d, v in win],
            "gap_bp_v11": round((cur_val - anchor) * 100, 1),
            "gap_bp_pointwise": round((cur_val - series[idx][1]) * 100, 1),
        },
        source="U.S. Treasury Daily Par Real Yield Curve (CSV)",
        asof=cur_date.isoformat(),
        method=f"현재=직전 영업일 확정치 · 앵커=6개월 전 중심 {2*half+1}영업일 평균 (규칙 v1.1)",
    )


# ───────────────────────── 3. 달러지수 (FRED) ─────────────────────────

FRED = "https://fred.stlouisfed.org/graph/fredgraph.csv?id={sid}"


def fetch_dollar(ref_date, series_id="DTWEXBGS", half=10):
    rows = []
    for row in csv.reader(io.StringIO(_get(FRED.format(sid=series_id), timeout=90))):
        if len(row) < 2 or row[0].lower().startswith(("date", "observation")):
            continue
        v = row[1].strip()
        if v in (".", ""):
            continue
        rows.append((dt.datetime.strptime(row[0].strip(), "%Y-%m-%d").date(), float(v)))
    rows.sort()
    past = [x for x in rows if x[0] < ref_date]
    if not past:
        raise RuntimeError("no observation before ref_date")
    cur_date, cur_val = past[-1]

    target = cur_date - dt.timedelta(days=182)
    idx = min(range(len(rows)), key=lambda i: abs((rows[i][0] - target).days))
    win = rows[max(0, idx - half): idx + half + 1]
    anchor = sum(v for _, v in win) / len(win) if len(win) >= 2 * half + 1 else None

    return field(
        value={"current": cur_val, "current_date": cur_date.isoformat(),
               "anchor_smoothed": round(anchor, 4) if anchor else None,
               "change_pct": round((cur_val / anchor - 1) * 100, 2) if anchor else None},
        source=f"FRED {series_id} (Nominal Broad U.S. Dollar Index, 연준 공식)",
        asof=cur_date.isoformat(),
        method="DXY(ICE 독점·재조회 불가) 대체. 신호등 없는 참고 지표.",
    )


# ───────────────────────── 4. ETF (Farside) ─────────────────────────

def fetch_etf(ref_date, days=5):
    html = _get("https://farside.co.uk/btc/")
    pat = re.compile(r"(\d{2}\s+\w{3}\s+\d{4})(.*?)</tr>", re.S)
    out = []
    for m in pat.finditer(html):
        try:
            d = dt.datetime.strptime(m.group(1), "%d %b %Y").date()
        except ValueError:
            continue
        cells = re.findall(r"<td[^>]*>(.*?)</td>", m.group(2), re.S)
        if not cells:
            continue
        raw = re.sub(r"<[^>]+>", "", cells[-1]).strip().replace(",", "")
        neg = raw.startswith("(") and raw.endswith(")")
        raw = raw.strip("()")
        try:
            v = float(raw)
        except ValueError:
            continue
        out.append((d, -v if neg else v))
    out = sorted(set(out))
    conf = [x for x in out if x[0] < ref_date]        # 당일 부분값 제외
    if not conf:
        raise RuntimeError("no confirmed ETF rows")
    sel = conf[-days:]
    return field(
        value={"daily": [[d.isoformat(), v] for d, v in sel],
               "sum_musd": round(sum(v for _, v in sel), 1),
               "window": f"{sel[0][0].isoformat()}~{sel[-1][0].isoformat()}"},
        source="Farside Investors",
        asof=sel[-1][0].isoformat(),
        method=f"직전 확정 {len(sel)}거래일 누적 · 확인지표(진입 신호 아님)",
    )


# ───────────────────────── 5. V-Lab 변동성 ─────────────────────────

VLAB = "https://vlab.stern.nyu.edu/volatility/VOL.BTCUSD%3AFOREX-R.{model}"


def fetch_vlab(model="GJR-GARCH"):
    html = _get(VLAB.format(model=model))
    txt = re.sub(r"<[^>]+>", " ", html)
    txt = re.sub(r"\s+", " ", txt)

    def horizon(label):
        m = re.search(label + r"\s*([\d.]+)%\s*(increased|decreased)\s*by\s*([\d.]+)%", txt)
        if not m:
            return None, None
        v = float(m.group(1))
        delta = float(m.group(3)) * (1 if m.group(2) == "increased" else -1)
        return v, round(v - delta, 2)     # 두 번째 = 현재 조건부 변동성(역산)

    d1, base1 = horizon(r"1\s*Day")
    w1, basew = horizon(r"1\s*Week")
    m1, basem = horizon(r"1\s*Month")
    upd = re.search(r"Analysis last updated:\s*([A-Za-z]+,\s*[A-Za-z]+ \d+, \d{4} at [\d:]+ [AP]M UTC)", txt)
    est = re.search(r"([A-Z][a-z]{2} \d{1,2}, \d{4}) to ([A-Z][a-z]{2} \d{1,2}, \d{4})", txt)

    # 세 지평의 역산 기준선이 일치해야 파싱이 옳다 (6식 교차검증)
    bases = [b for b in (base1, basew, basem) if b is not None]
    consistent = len(bases) == 3 and max(bases) - min(bases) < 0.02

    return field(
        value={"h1d": d1, "h1w": w1, "h1m": m1,
               "spot_conditional": round(sum(bases) / len(bases), 2) if consistent else None,
               "spot_consistent": consistent,
               "updated_at": upd.group(1) if upd else None,
               "estimation_period": f"{est.group(1)} to {est.group(2)}" if est else None},
        source=f"V-Lab {model}",
        asof=upd.group(1) if upd else None,
        method="h1m=현 운용 정의(C1). spot_conditional=표시 델타 역산, 세 지평 교차검증.",
    )


# ───────────────────────── 6. 김치프리미엄 ─────────────────────────

def fetch_kimp(ref_date, usd_price):
    end_utc = dt.datetime.combine(ref_date, dt.time(22, 0), tzinfo=KST).astimezone(UTC)
    url = ("https://api.upbit.com/v1/candles/minutes/60?market=KRW-BTC&count=1&to="
           + end_utc.strftime("%Y-%m-%dT%H:%M:%S"))
    krw = float(_get(url, as_json=True)[0]["trade_price"])

    fx_url = f"https://api.frankfurter.app/{ref_date.isoformat()}?from=USD&to=KRW"
    fx_js = _get(fx_url, as_json=True)
    fx = float(fx_js["rates"]["KRW"])

    prem = (krw / (usd_price * fx) - 1) * 100
    return field(
        value={"upbit_krw": krw, "coinbase_usd": usd_price, "usdkrw": fx,
               "premium_pct": round(prem, 3)},
        source="Upbit KRW-BTC · Coinbase BTC-USD · Frankfurter(ECB) USD/KRW",
        asof=f"{ref_date.isoformat()} 22:00 KST",
        method=("(업비트 원화가) / (코인베이스 달러가 × 환율) − 1. "
                "환율은 ECB 일일 기준(16:00 CET)이라 22:00 KST와 시점이 다르다. "
                "확인지표 — 신호등 없음."),
    )


# ───────────────────────── 스냅샷 ─────────────────────────

def previous_snapshot(ref_date):
    if not os.path.isdir(SNAP_DIR):
        return None
    files = sorted(f for f in os.listdir(SNAP_DIR) if f.endswith(".json"))
    files = [f for f in files if f[:10] < ref_date.isoformat()]
    for f in reversed(files):                      # 실패 스냅샷은 직전값으로 쓰지 않는다
        with open(os.path.join(SNAP_DIR, f), encoding="utf-8") as fh:
            js = json.load(fh)
        if js.get("derived", {}).get("divergence_pct") is not None:
            return js
    return None


def derive(snap, prev):
    """카드에 바로 쓰는 파생값. 기여 분해는 의무 항목."""
    d = {}
    px = snap["price"]["value"]
    wk = snap["weekly_closes"]["value"]
    if px and wk:
        d.update(compute_sma30(wk, px))
        d["price_gap_to_cross_pct"] = round((d["cross_price"] / px - 1) * 100, 2)
        s4 = compute_sma_4w_ago(wk)
        if s4:
            d["sma30_4w_ago"] = s4
            d["sma30_slope_4w"] = round(d["sma30"] - s4, 2)
            d["c3_slope_ok"] = d["sma30"] >= s4
        d["c3_position_ok"] = px > d["sma30"]
        d["c3_ok"] = bool(d.get("c3_position_ok")) and bool(d.get("c3_slope_ok"))

    t = snap["tips"]["value"]
    if t:
        d["c2_gap_bp"] = t["gap_bp_v11"]
        d["c2_ok"] = t["gap_bp_v11"] < 0
        d["c2_distance_bp"] = round(-t["gap_bp_v11"], 1)

    v = snap["vlab_gjr"]["value"]
    if v and v.get("h1m") is not None:
        d["c1_value"] = v["h1m"]
        d["c1_ok"] = v["h1m"] <= 40.0
        d["c1_distance_pp"] = round(40.0 - v["h1m"], 2)

    # 이격률 기여 분해 (직전 스냅샷 대비) — 문구 규약상 필수
    if prev and prev.get("derived", {}).get("divergence_pct") is not None and "divergence_pct" in d:
        p0 = prev["price"]["value"]
        s0 = prev["derived"]["sma30"]
        dv0 = prev["derived"]["divergence_pct"]
        price_only = (px / s0 - 1) * 100
        d["divergence_delta_pp"] = round(d["divergence_pct"] - dv0, 3)
        d["contrib_price_pp"] = round(price_only - dv0, 3)
        d["contrib_centerline_pp"] = round(d["divergence_pct"] - price_only, 3)
        d["prev"] = {"date": prev["ref_date"], "price": p0, "sma30": s0, "divergence_pct": dv0}

    ok = [d.get("c1_ok"), d.get("c2_ok"), d.get("c3_ok")]
    d["conditions_met"] = sum(1 for x in ok if x is True)
    d["conditions_known"] = sum(1 for x in ok if x is not None)
    return d


def collect(ref_date):
    ref_date = resolve_ref_date(ref_date)
    snap = {
        "schema": "btc-journal-snapshot/1.0",
        "rule_version": "v1.1",
        "ref_date": ref_date.isoformat(),
        "ref_time_kst": "22:00",
        "collected_at": dt.datetime.now(UTC).isoformat(),
    }
    snap["price"] = safe(fetch_price_2200kst, ref_date)
    snap["weekly_closes"] = safe(fetch_weekly_closes, ref_date)
    snap["tips"] = safe(fetch_tips, ref_date)
    snap["dollar"] = safe(fetch_dollar, ref_date)
    snap["etf"] = safe(fetch_etf, ref_date)
    snap["vlab_gjr"] = safe(fetch_vlab, "GJR-GARCH")
    snap["vlab_egarch"] = safe(fetch_vlab, "EGARCH")
    snap["kimp"] = (safe(fetch_kimp, ref_date, snap["price"]["value"])
                    if snap["price"]["value"] else field(error="price unavailable"))

    prev = previous_snapshot(ref_date)
    snap["derived"] = derive(snap, prev)
    snap["unverified"] = [k for k, v in snap.items()
                          if isinstance(v, dict) and v.get("verified") is False]
    return snap


def report(snap):
    d = snap["derived"]
    print(f"\n═══ {snap['ref_date']} 22:00 KST · 규칙 {snap['rule_version']} ═══")
    if d.get("conditions_known") == 3:
        print(f"판정 {d['conditions_met']}/3")
    else:
        print(f"판정 보류 — 3조건 중 {d.get('conditions_known')}개만 산출됨. "
              f"발행 금지(누락 항목 확인 후 재수집).")
    if "divergence_pct" in d:
        print(f"가격 ${snap['price']['value']:,.2f}  SMA30 ${d['sma30']:,.2f}  이격률 {d['divergence_pct']:+.2f}%")
        print(f"교차가 ${d['cross_price']:,.2f}  (30주선까지 {d['price_gap_to_cross_pct']:+.2f}%)")
        if "sma30_slope_4w" in d:
            print(f"기울기 4주 {d['sma30_slope_4w']:+,.2f}  → 기울기 조건 {'충족' if d['c3_slope_ok'] else '미충족'}")
    if "contrib_price_pp" in d:
        print(f"이격률 변화 {d['divergence_delta_pp']:+.2f}%p"
              f"  = 가격 {d['contrib_price_pp']:+.2f}%p + 중심선 {d['contrib_centerline_pp']:+.2f}%p")
    if d.get("c1_value") is not None:
        print(f"C1 변동폭 {d['c1_value']}%  (충족선까지 {d['c1_distance_pp']:+.2f}%p)")
    if d.get("c2_gap_bp") is not None:
        print(f"C2 실질금리 갭 {d['c2_gap_bp']:+.1f}bp  (충족선까지 {-d['c2_distance_bp']:+.1f}bp)")
    if snap["unverified"]:
        print(f"\n⚠ 미검증 → 카드에 [미측정] 표기 필수: {', '.join(snap['unverified'])}")
        for k in snap["unverified"]:
            print(f"   - {k}: {snap[k]['error']}")
    else:
        print("\n✓ 전 항목 검증됨 — ★ 없음")


# ───────────────────────── 셀프테스트 ─────────────────────────

def selftest():
    print("── 셀프테스트 (네트워크 없음) ──")
    base = dt.date(2026, 2, 2)
    wk = [{"week_start": (base + dt.timedelta(weeks=i)).isoformat(),
           "close": 70000.0} for i in range(33)]
    r = compute_sma30(wk, 62721.5)
    assert abs(r["sma30"] - (70000 * 29 + 62721.5) / 30) < 0.01
    assert abs(r["cross_price"] - 70000.0) < 1e-6, r
    # 교차가 항등식: 그 가격이면 SMA와 정확히 같아야 한다
    chk = compute_sma30(wk, r["cross_price"])
    assert abs(chk["divergence_pct"]) < 1e-6, chk
    print(f"  SMA30 항등식 OK  (교차가 ${r['cross_price']:,.2f}, 이격률 {chk['divergence_pct']:.9f}%)")

    # 기여 분해: 8/11 → 8/14 실제 수치로 검증
    prev = {"ref_date": "2026-08-11", "price": {"value": 64226.5},
            "derived": {"sma30": 69940.5, "divergence_pct": -8.170}}
    snap = {"price": {"value": 62721.5},
            "weekly_closes": {"value": [{"week_start": "x", "close": 2033989.5 / 29}] * 29},
            "tips": {"value": None}, "vlab_gjr": {"value": None}}
    d = derive(snap, prev)
    assert abs(d["divergence_pct"] - (-10.257)) < 0.01, d["divergence_pct"]
    assert abs(d["contrib_price_pp"] - (-2.152)) < 0.01, d["contrib_price_pp"]
    assert abs(d["contrib_centerline_pp"] - 0.064) < 0.01, d["contrib_centerline_pp"]
    assert abs(d["divergence_delta_pp"] - (d["contrib_price_pp"] + d["contrib_centerline_pp"])) < 1e-6
    print(f"  기여 분해 OK  ({d['divergence_delta_pp']:+.2f}%p = 가격 {d['contrib_price_pp']:+.2f} "
          f"+ 중심선 {d['contrib_centerline_pp']:+.2f})")

    # 실패 시 이월 금지
    f = safe(lambda: (_ for _ in ()).throw(RuntimeError("boom")))
    assert f["value"] is None and f["verified"] is False
    print("  실패 항목 null 처리 OK")

    # 앵커 평활화 산술 (8/14 회차 실제 값)
    win = [1.90,1.94,1.92,1.94,1.89,1.88,1.87,1.84,1.86,1.80,1.77,
           1.79,1.80,1.79,1.80,1.77,1.78,1.77,1.74,1.72,1.76]
    anchor = sum(win) / len(win)
    assert abs(anchor - 1.8252) < 1e-3, anchor
    assert abs((2.39 - anchor) * 100 - 56.5) < 0.6
    print(f"  TIPS 앵커 평활화 OK  (앵커 {anchor:.4f}% · 갭 {(2.39-anchor)*100:+.1f}bp)")
    print("── 전부 통과 ──")


def main():
    args = [a for a in sys.argv[1:]]
    if "--selftest" in args:
        selftest()
        return
    ref = dt.date.fromisoformat(args[0]) if args else dt.datetime.now(KST).date()
    snap = collect(ref)
    ref = dt.date.fromisoformat(snap["ref_date"])
    os.makedirs(SNAP_DIR, exist_ok=True)
    path = os.path.join(SNAP_DIR, f"{ref.isoformat()}T2200KST.json")

    # 이미 더 완전한 스냅샷이 있으면 덮어쓰지 않는다
    if os.path.exists(path):
        try:
            with open(path, encoding="utf-8") as fh:
                old = json.load(fh)
            if (old.get("derived", {}).get("conditions_known", 0)
                    > snap["derived"].get("conditions_known", 0)):
                path = path.replace(".json", ".partial.json")
                print(f"[안내] 기존 스냅샷이 더 완전하다. 덮어쓰지 않고 {os.path.basename(path)} 로 저장한다.")
        except Exception:
            pass

    with open(path, "w", encoding="utf-8") as fh:
        json.dump(snap, fh, ensure_ascii=False, indent=2)
    report(snap)
    print(f"\n저장: {path}")

    # 3조건이 다 산출되지 않으면 종료 코드 2 → GitHub Actions가 빨간불 + 알림 메일
    if snap["derived"].get("conditions_known") != 3:
        sys.exit(2)


if __name__ == "__main__":
    main()
