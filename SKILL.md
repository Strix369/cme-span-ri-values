---
name: cme-span-ri-values
description: Calculate and record NQ/MNQ and ES/MES Risk Interval candidates from official CME margin evidence. Use for CME SPAN RI values, daily RI requests, and observed-value comparisons.
---

# CME SPAN RI VALUES

Author: RANX. This is an independent research workflow, not an official CME service. Its margin-to-RI equation is inferred and has known misses. Do not describe the method as verified, guaranteed or exact for future openings.

This file contains the complete skill. Work in the user's selected project. Store generated records in `research/`; no private history is required to install or use the skill. Do not execute orders, connect a broker or configure any external platform. This skill reads data, calculates candidates and records comparisons.

## 1. Resolve the request and session

Read the actual clock with its UTC offset. Identify the requested trading date in `America/New_York`; do not select it from the user's Indian calendar date alone.

- A normal Globex session starts at 18:00 New York on Sunday through Thursday. Sunday evening belongs to Monday's trading date; other evening starts belong to the next trading date.
- A daytime request before 17:00 New York on a trading weekday refers to the current session. The 17:00 to 18:00 break refers to the upcoming session. At or after 18:00, current Globex means the session that just opened. A late automation must not silently advance to another day's opening.
- Friday's market-close capture prepares Sunday's opening and uses Monday's filename. Saturday and pre-open Sunday requests refer to the same Monday trading date.
- The normal capture window is 17:00 through 18:00 New York on the capture day. For Monday's trading date, that is Friday's window. It does not extend over the weekend. A later Saturday or Sunday recovery read remains provisional.
- Verify holiday exceptions against the official CME calendar. Never infer a holiday opening from the normal hours alone. Retain the calendar evidence and mark an uncertain opening provisional.

Scheduling is optional and separate from installation. If requested, the RANX default is fixed at **04:00 Asia/Kolkata, Tuesday through Saturday**, with no Sunday or Monday run. During U.S. standard time this corresponds to 17:30 New York on the preceding calendar day. Do not shift that fixed Indian time automatically for daylight saving changes. Let the user request a change. Use New York time to assess sessions and timeliness. Log actual trigger time and capture time separately; a configured schedule does not establish that a run occurred.

## 2. Choose the answer's authority

Check the selected date's existing records before fetching:

1. A confirmed user observation for that date is authoritative. A tentative or "likely" observation must stay tentative.
2. Otherwise use the latest eligible saved forecast captured before both opening and actual reveal. Review its raw evidence and preserved revisions; do not trust an old eligibility flag alone.
3. If neither exists, fetch live CME rows. Missing history does not mean the browser failed.
4. When the opening has passed, save a new calculation with null `predictions` and numeric `diagnostic_candidates`. Show it with a post-open label. It cannot establish the original opening RI or receive a pre-open score.

Never reuse another date's actual, forecast or cached margin as a new numeric prediction. Keep an eligible forecast authoritative over later ineligible reads. Freeze the prediction once actuals appear; put later diagnostics into the separate result record.

## 3. Read official CME sources

Use the agent's built-in browser in background tabs. Do not read or change the user's Chrome tabs. Read the documentation for the browser tool actually available in the host; do not assume a specific tool namespace exists.

| Symbol | Official margin page | USD per index point |
| --- | --- | ---: |
| NQ | https://www.cmegroup.com/markets/equities/nasdaq/e-mini-nasdaq-100.margins.html | 20 |
| MNQ | https://www.cmegroup.com/markets/equities/nasdaq/micro-e-mini-nasdaq-100.margins.html | 2 |
| ES | https://www.cmegroup.com/markets/equities/sp/e-mini-sandp500.margins.html | 50 |
| MES | https://www.cmegroup.com/markets/equities/sp/micro-e-mini-sandp-500.margins.html | 5 |

Read all four pages for a new capture. Save each complete rendered table, exact source URL, actual UTC read time and visible contract header. A documented table locator such as `getByRole('table').innerText()` may provide the rendered text. Do not call a hidden endpoint or bypass access restrictions.

If the browser cannot connect, inspect its provider inventory. An empty inventory is a connection failure before CME is reached, not a CME access-denied response. Wait no more than 30 seconds, refresh the inventory once, and retry a fresh built-in tab. A page showing `Loading...` needs another observation after other source work; if still unreadable, retry once in a fresh tab. Record the actual failed URL, time and visible error. Never switch to Chrome to evade a failure. Stop after the bounded retry and keep unavailable values explicit.

Return `Unavailable` for a family only if its primary E-mini read or validation fails after that retry and no authoritative observation or eligible forecast exists. Missing Micro data weakens the check but does not replace or automatically invalidate a valid E-mini input. If an authorised retry later succeeds, preserve the failed record as a revision and use newly read evidence. Recovery does not prove why access failed or guarantee the next scheduled connection.

### Contract, field and freshness checks

- Require named columns `EXCHANGE`, `ASSET CLASS`, `PRODUCT`, `PRODUCT CODE`, `PERIOD`, `MAINTENANCE LONG`, `MAINTENANCE SHORT`. Bind LONG and SHORT by those headers, not column guesses.
- Select exactly one row for CME, EQUITY INDEX, the requested symbol and verified expiry. Check the product name, Micro size and USD currency.
- Verify the live contract header against the selected row. Quarter codes are H for March, M for June, U for September and Z for December. Around a roll, inspect the [official roll calendar](https://www.cmegroup.com/trading/equity-index/rolldates.html), contract calendar and volume context. Never choose an expiry because it fits a revealed actual.
- Use MAINTENANCE LONG for the fixed candidate. Retain SHORT for review. Initial margin, scan range, portfolio margin and maintenance margin are not interchangeable.
- A newly validated capture must contain rows read in that capture, normally within ten minutes of validation. Do not relabel an earlier collection timestamp. A saved forecast can be replayed later at its original time for audit; that is not a fresh capture.
- Quote update times date quotes, not the margin rows. Keep `margin_effective_business_date` and `margin_cycle` null unless the source explicitly states them. An unchanged live row does not prove staleness; a fresh quote does not prove a new margin cycle.

Read the NQ and ES settlement pages by replacing `.margins.html` with `.settlements.html` in their official URLs. Record the displayed settlement trade date, update time and selected-expiry row. Settlement prices provide context and do not enter the equation. Delayed settlements must not block otherwise valid margin calculations. Check [official trading hours](https://www.cmegroup.com/trading-hours.html) for the selected opening and holiday exceptions.

## 4. Calculate the fixed candidate

Let `M` be the selected maintenance-long margin in USD and `V` be USD per index point:

```text
raw_RI = 11 * M / (80 * V)
RI = floor(11 * M / (20 * V)) / 4
```

Floor to a quarter point. Do not round to the nearest quarter. NQ defines NQ/MNQ; ES defines ES/MES. Calculate Micro outputs as checks. Record each Micro margin's difference from one tenth of the E-mini margin. A difference up to one Micro dollar can arise from whole-dollar display precision and can cross a quarter-floor boundary. Keep E-mini authority in that case and record both values. Recheck a larger discrepancy; leave that family unavailable if it remains unresolved.

The following code is self-contained. Run it in memory or a temporary location. Python needs `fractions` and `zoneinfo` with working IANA timezone data. Call `session_plan` with the actual time; provide `trading_date` only when a specific date is requested. Official holiday exceptions require an explicit trading date and verified opening.

```python
from datetime import date, datetime, time, timedelta, timezone
from fractions import Fraction
from zoneinfo import ZoneInfo

NY = ZoneInfo("America/New_York")
IST = ZoneInfo("Asia/Kolkata")
POINT_VALUES = {"NQ": 20, "MNQ": 2, "ES": 50, "MES": 5}

def stamp(value):
    result = datetime.fromisoformat(str(value).replace("Z", "+00:00"))
    if result.tzinfo is None:
        raise ValueError("Timestamp needs a UTC offset")
    return result

def ri(symbol, margin_usd):
    margin = Fraction(str(margin_usd).replace(",", ""))
    if margin <= 0:
        raise ValueError("Margin must be positive")
    raw = Fraction(11, 80) * margin / POINT_VALUES[symbol]
    floored = (raw * 4).__floor__() * Fraction(1, 4)
    return {"raw_exact": str(raw), "raw_ri_points": float(raw),
            "quarter_floor_ri_points": float(floored)}

def session_plan(at, trading_date=None, official_open=None):
    now = stamp(at).astimezone(NY)
    if trading_date:
        trading = date.fromisoformat(trading_date)
        if trading.weekday() > 4:
            raise ValueError("Normal trading date must be Monday through Friday")
        opened = datetime.combine(trading - timedelta(days=1), time(18), NY)
    elif now.weekday() in (0, 1, 2, 3, 6) and now.time() >= time(18):
        opened = datetime.combine(now.date(), time(18), NY)
        trading = (opened + timedelta(days=1)).date()
    elif now.weekday() < 5 and now.time() < time(17):
        trading = now.date()
        opened = datetime.combine(trading - timedelta(days=1), time(18), NY)
    else:
        for offset in range(8):
            candidate = datetime.combine(now.date() + timedelta(days=offset), time(18), NY)
            if candidate.weekday() in (0, 1, 2, 3, 6) and candidate > now:
                opened = candidate
                trading = (opened + timedelta(days=1)).date()
                break
    if official_open:
        if not trading_date:
            raise ValueError("Holiday exception needs an explicit trading date")
        opened = stamp(official_open).astimezone(NY)
        if opened.date() != trading - timedelta(days=1):
            raise ValueError("Review exceptional session date against official hours")
    capture_day = trading - timedelta(days=3 if trading.weekday() == 0 else 1)
    close = datetime.combine(capture_day, time(17), NY)
    nominal = datetime.combine(capture_day, time(17, 30), NY)
    capture_end = min(opened, datetime.combine(capture_day, time(18), NY))
    in_window = close <= now < capture_end
    phase = ("post_open" if now >= opened else "pre_open" if in_window
             else "provisional_outside_window" if now >= close else "provisional_early")
    return {"target_trading_date": trading.isoformat(), "session_open": opened.isoformat(),
            "session_open_utc": opened.astimezone(timezone.utc).isoformat(),
            "capture_utc": now.astimezone(timezone.utc).isoformat(),
            "capture_new_york": now.isoformat(), "capture_asia_calcutta": now.astimezone(IST).isoformat(),
            "requested_capture_new_york": nominal.isoformat(),
            "requested_capture_asia_calcutta": nominal.astimezone(IST).isoformat(),
            "captured_before_session_open": now < opened,
            "captured_in_requested_close_to_open_window": in_window,
            "scheduled_delay_seconds": (now - nominal).total_seconds(),
            "holiday_override_applied": bool(official_open), "phase": phase}

def compare(candidate, actual):
    keys = ("NQ_MNQ", "ES_MES")
    errors = {key: Fraction(str(candidate[key])) - Fraction(str(actual[key])) for key in keys}
    return {"signed_error_prediction_minus_actual": {k: float(v) for k, v in errors.items()},
            "absolute_error": {k: float(abs(v)) for k, v in errors.items()},
            "mean_absolute_error_points": float(sum(abs(v) for v in errors.values()) / 2),
            "exact_match_count": sum(v == 0 for v in errors.values()), "family_count": 2}
```

Only compare numeric pairs with this helper. Keep a missing family null and explain the missing comparison instead of converting it to zero.

## 5. Save the evidence and results

Use the trading date, with these exact canonical names:

- `research/RI_YYYY-MM-DD_prediction.json` and `.md`
- `research/RI_YYYY-MM-DD_result.json` and `.md`

Update a date's existing pair in place. Do not create duplicate forecast names or `data/forecast_log.json`. Preserve existing observed images, input files, check notes and older historical versions. Never fabricate an observed screenshot.

### Prediction record

Save schema version and revision, target trading date, opening with offset and UTC, actual trigger and capture times, capture UTC/New York/India, phase/status, before-open and capture-window flags, eligibility and reason, whether the actual was already seen, fixed method version, `predictions`, optional `diagnostic_candidates`, selected margin rows, point values, raw and floored RI values, Micro checks, and source/calendar/settlement checks. Retain limitations and actual source errors.

Retain raw browser evidence using this shape, populated only with observed content:

```json
{
  "browser": "Codex built-in in-app browser",
  "period": "verified MM/YYYY",
  "captured_at": "actual timestamp with UTC offset",
  "rows": [
    {
      "symbol": "NQ",
      "url": "exact official source URL",
      "captured_at": "actual page read timestamp",
      "header_context": "visible contract header and quote context",
      "table_text": "complete rendered table",
      "margin_effective_business_date": null
    }
  ]
}
```

The example is a schema, not usable market input. Include all four row attempts in a real capture. For a failed read keep empty `table_text`, an actual `read_error` and its attempt times. Hash each retained table using SHA-256 of its UTF-8 bytes. An unavailable family is null, not zero. A new post-open record keeps family predictions null and numeric calculations under `diagnostic_candidates`.

Before any pre-reveal update, store a flat `revisions` entry containing the previous capture time, reason, complete previous JSON snapshot without nested revisions, previous Markdown text, and SHA-256 hashes of both original raw files. State that the hashes refer to raw file bytes. Do not overwrite an eligible forecast with an ineligible later read. Once actuals appear, freeze that prediction pair byte-for-byte.

### Result record

Record the user's original statement, observed family values, confirmation status and recording time. Save exact receipt time only if known; do not invent it from the recording time. Keep tentative observations tentative. Attach the latest eligible pre-open/pre-reveal forecast's revision and hashes, or `latest_eligible_prediction: null` if none qualifies.

Use `forecast - actual` for signed errors, absolute errors, exact count out of two families and two-family MAE. Compare only available numeric values. Mark `comparison.scored: false` if there was no eligible capture; retain any late/provisional comparison as a separate diagnostic. Tentative actuals do not count toward confirmed accuracy. Preserve corrections to actuals in their own result revision history. Put later source investigations in the result, not into the frozen forecast.

### Diagnose a miss without fitting it away

Replay the retained source extraction, field selection, expiry and exact arithmetic. Compare adjacent same-expiry captures only as evidence. Under the unchanged rule, observed RI `R` requires:

```text
80 * V * R / 11 <= M < 80 * V * (R + 0.25) / 11
```

This interval is conditional on the candidate equation, not a recovered historical margin. A matching old margin or alternate expiry after reveal does not prove the provider used it. Do not change the coefficient, rounding, field or expiry to force an exact result. Separate a proven calculation or timing fault from an unknown numerical cause.

For example, a retained NQ input of $41,718 gives 286.75, while an observed 288.50 implies a different input under this rule. An older December row and a later-expiry row can both match that observation. This ambiguity leaves input selection unresolved. Do not reuse these illustrative amounts as future predictions. Historical discovery matches are retrospective, not independent forward tests.

## 6. Validate and reply

Reopen both saved files. Recompute arithmetic from the retained raw table and named columns. Verify contract, USD multiplier, Micro scaling, source read times, session flags, previous snapshot hashes and observed-first answer authority. On a repair, review affected historical snapshots without rewriting original evidence. A browser failure, late run, miss or unavailable family must remain visible in the records.

Reply briefly:

```text
NQ / MNQ: value
ES / MES: value
```

Add only a material note about date, inferred forecast, post-open status, tentative actuals or source failure. Keep calculations and diagnostics in local records. A user confirmation needs a short acknowledgement after saving the comparison. A repeated live row or passing software check does not prove the RI model or provider cutoff. Only future captures and observed comparisons can assess predictive accuracy.
