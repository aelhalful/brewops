# Implementation plan: dashboard date-range filter

Ticket: `tickets/005-date-filter.md`. Goal: let the dashboard's numbers, charts, and
machine cards be scoped to a date range instead of always showing all-time totals.

Read this whole file before changing anything. It names every file and function you
need to touch, in the order to touch them.

## Contract

- Date format on the wire: plain calendar dates, `YYYY-MM-DD` (e.g. `2026-09-16`).
  No time-of-day component. This matches an HTML `<input type="date">` value exactly,
  so the frontend can pass it straight through with no formatting.
- Query params: `start` and `end`, both optional, both `YYYY-MM-DD`.
  - `start` is inclusive: brews on that calendar day count.
  - `end` is inclusive: brews on that calendar day count.
  - If both are omitted, behave exactly as today (all-time). This keeps any other
    caller of `/api/stats` working unchanged.
  - If only one of `start`/`end` is given, treat the other side as open-ended
    (no lower/upper bound).
- Storage note: `brew_events.timestamp` is stored as `YYYY-MM-DD HH:MM:SS` (see
  `src/brewops/db/queries.py` module docstring). Comparing a plain date string like
  `'2026-09-16'` against a full timestamp with `>=` works correctly for the lower
  bound, but for the upper bound you must compare against `DATE(timestamp) <= ?`
  (or bump `end` to `end + 1 day` and use `<`) — otherwise brews on the `end` day
  after midnight get excluded. Use `DATE(timestamp) BETWEEN ? AND ?` for both bounds;
  it is simplest and correct for inclusive start/end on calendar dates.

## Backend changes

### `src/brewops/db/queries.py`

**`get_stats`** (currently line ~68): add two optional parameters,
`start: str | None = None` and `end: str | None = None`.

- Build a `WHERE` clause fragment and parameter tuple once, shared by all three
  sub-queries:
  - No filter: no `WHERE`.
  - `start` only: `WHERE DATE(timestamp) >= ?`
  - `end` only: `WHERE DATE(timestamp) <= ?`
  - Both: `WHERE DATE(timestamp) BETWEEN ? AND ?`
- `total`: apply the filter to the `COUNT(*)` query directly.
- `per_drink`: this query uses `LEFT JOIN brew_events be ON be.drink_type = dt.name`
  specifically so that drinks with zero brews still appear in the result (see
  existing comment-free but deliberate structure). **Do not** add the date filter as
  a top-level `WHERE` — that would turn unmatched left-join rows into exclusions
  incorrectly in some SQLite edge cases and is easy to get backwards. Instead move
  the filter into the `ON` clause: `LEFT JOIN brew_events be ON be.drink_type =
  dt.name AND DATE(be.timestamp) BETWEEN ? AND ?` (or the start-only/end-only
  variant). This preserves "every drink type appears, count 0 if none in range."
- `per_day`: apply the filter as a normal `WHERE` before the `GROUP BY`.
- Return value shape is unchanged: `{"total_brews": ..., "per_drink": [...], "per_day": [...]}`.

**`get_machine_health`** (currently line ~97): add the same `start`/`end`
parameters. Apply the same filter fragment to:
- the `brews` query (`COUNT(*)`, `MAX(timestamp)`) — filtered.
- the `top_drinks` CTE's `WHERE be.machine_id = ?` — add the date condition there
  too.
- **Do not** filter `last_maintenance` or `recent_errors`. Maintenance and error
  history reflect the machine's current real-world state and should stay all-time
  regardless of the dashboard's selected range. This is a deliberate product
  decision, not an oversight — leave those two queries exactly as they are.
- If a machine has zero brews in the selected range, the existing code already
  degrades correctly with no changes needed: `MAX(timestamp)` over zero rows is
  `NULL`, and the `top_drinks` CTE's `WHERE count = (SELECT MAX(count) FROM
  counts)` is never true when `counts` is empty (comparing to `NULL` is never
  true), so `top_drinks` comes back as `[]`. Verify this stays true after your
  edit — don't add a special case for it.

### `src/brewops/api/main.py`

**`/api/stats`** (currently line ~73):

```python
@app.get("/api/stats")
def stats(
    start: str | None = None,
    end: str | None = None,
    conn: sqlite3.Connection = Depends(get_db),
):
    validate_date_range(start, end)
    return queries.get_stats(conn, start=start, end=end)
```

**`/api/machines/{machine_id}`** (currently line ~84, the `machine_health`
function): add the same `start`/`end` query params and pass them through to
`queries.get_machine_health`.

**New helper `validate_date_range(start, end)`** in `main.py`, near
`parse_timestamp`:
- If a value is given, it must match `YYYY-MM-DD` — validate with
  `datetime.strptime(value, "%Y-%m-%d")`; raise `HTTPException(400, f"unparsable
  date {value!r}")` on failure. Do not reuse `parse_timestamp` as-is — it expects
  datetime formats and rejects future dates, neither of which applies here (a
  future `end` date is harmless and should just return an empty range, not error).
- If both `start` and `end` are given and `start > end` (string comparison is safe
  here since the format is zero-padded `YYYY-MM-DD`), raise
  `HTTPException(400, "start date is after end date")`. This is the "from-after-to"
  edge case — it must be a 400, not a silently-empty result.

## Frontend changes

### `src/brewops/frontend/index.html`

Inside `<section id="dashboard">`, before `.stat-tiles`, add a small filter bar:

```html
<div class="date-filter">
  <label for="filter-start">From</label>
  <input type="date" id="filter-start">
  <label for="filter-end">To</label>
  <input type="date" id="filter-end">
  <button type="button" id="filter-apply">Apply</button>
  <button type="button" id="filter-reset">This week</button>
</div>
```

No `<form>` wrapper needed — plain buttons with click handlers, since there's
nothing to submit to a server via a redirect.

### `src/brewops/frontend/app.js`

- Add a function `defaultRange()` that returns `{start, end}` for "one working
  week": Monday through Friday of the current week, as `YYYY-MM-DD` strings.
  Use local time (same pattern as the existing `localNow()` helper, which already
  handles the timezone-offset trick — reuse that trick, don't call `toISOString()`
  on a `Date` directly without correcting for offset, or the date can roll to the
  wrong day near midnight).
- On page load: set `#filter-start` / `#filter-end` to `defaultRange()`, then call
  `loadDashboard()`.
- `loadDashboard()` (currently line ~82): change its signature to
  `loadDashboard(start, end)`. Build the query string only with params that are
  non-empty: `start ? `start=${start}` : ""` etc., joined with `&`. Append to both
  `/api/stats` and every `/api/machines/${m.id}` call (the machine cards must use
  the same range as the stats tiles — this was the ticket's requirement that
  machine cards be in scope).
- Add a click handler on `#filter-apply` that reads both input values and calls
  `loadDashboard(start, end)`. Add one on `#filter-reset` that recomputes
  `defaultRange()`, writes it back into the two inputs, and reloads.
- **Empty range handling**: if the user clears both date inputs and clicks Apply,
  treat that as "all time" — pass no `start`/`end` params at all (matches the
  backend's documented no-param behavior). Don't send empty-string query params
  (`start=`) since FastAPI will treat that as a present-but-empty string, not
  `None`, and validation will reject it. Only append a param to the query string
  when its input value is non-empty.
- **From-after-to**: don't validate this client-side beyond what's natural; let the
  400 from the API surface. Reuse the existing error-handling path: wrap the
  `loadDashboard` calls from the filter buttons in a try/catch that writes
  `error.message` somewhere visible (there's no existing dashboard-level message
  element — add a `<p id="filter-message" class="message" role="status"></p>`
  next to the filter bar in `index.html`, and set its text/class the same way
  `submitForm` does it in `app.js`).
- `renderDrinkBars` and `renderTimeline` (both unchanged) already handle an empty
  or all-zero `per_day`/`per_drink` array — `renderTimeline` already returns early
  on `perDay.length === 0`, and `Math.max(1, ...)` in `renderDrinkBars` already
  guards against an empty array producing `-Infinity`. No changes needed there,
  but re-check this after wiring up the filter, since it's the "days with no
  brews in range" case and it's cheap to confirm.

## Edge cases checklist

1. **No range selected (all params omitted)** — dashboard matches current
   behavior exactly. This must not regress; it's the default for any direct API
   caller that doesn't send `start`/`end`.
2. **Empty range after user clears both inputs** — treated as all-time, per above.
3. **`start` after `end`** — API returns `400`, frontend shows the error message
   near the filter bar, does not clear the previously-rendered dashboard.
4. **Range with zero brews in it entirely** (e.g. a week before the coffee
   machines existed) — `total_brews: 0`, `per_drink` shows every drink type at
   count 0 (because of the `LEFT JOIN ... ON` fix above), `per_day: []`, all
   machine cards show "no brews yet" / "never". Nothing should throw.
5. **A machine with brews overall but none in the selected range** — see the
   `get_machine_health` note above; already handled by existing null/empty
   semantics, verify it stays that way.
6. **Range that only partially overlaps data** (e.g. `end` is in the future) —
   should just return whatever real brews fall before "now"; no special-casing,
   the date comparison naturally handles it.
7. **Single-day range** (`start == end`) — must include that day's brews, not
   return empty. This is why both bounds are inclusive (`BETWEEN`), not
   half-open.

## How to verify

Run the app: `uv run start`, open `http://localhost:8123`.

1. **Regression check (no filter):** load the page fresh. Confirm total brews,
   per-drink bars, timeline, and machine cards all show the same numbers as
   before your change (compare against `git stash`-ed behavior or just eyeball
   that nothing looks newly zeroed-out).
2. **Default range:** confirm on load the two date inputs are pre-filled with
   this week's Monday and Friday (check against today's actual date).
3. **Manual range via curl**, to isolate backend from frontend:
   - `curl "http://localhost:8123/api/stats?start=2026-09-01&end=2026-09-07"` —
     confirm `total_brews` matches a manual count you can verify with
     `sqlite3 <db file> "SELECT COUNT(*) FROM brew_events WHERE DATE(timestamp) BETWEEN '2026-09-01' AND '2026-09-07'"`
     (check `src/brewops/db/connection.py` for the actual db file path).
   - `curl "http://localhost:8123/api/stats?start=2026-09-07&end=2026-09-01"` —
     confirm this returns HTTP 400, not a 200 with empty data.
   - `curl "http://localhost:8123/api/stats?start=2099-01-01&end=2099-01-07"` —
     confirm this returns `total_brews: 0` and `per_drink` still lists every
     drink type at count 0, not an empty list.
   - `curl "http://localhost:8123/api/stats"` (no params) — confirm it matches
     step 1's all-time numbers exactly.
4. **Machine cards in range:** pick a machine you know has brews outside the
   default week (check `recent_errors`/timestamps from step 3's exploration or
   the `brew_events` table directly). Set the date filter to a range excluding
   all its brews and click Apply. Confirm that machine's card shows "no brews
   yet" and "never" while its last-maintenance info (if any) is still shown —
   confirming maintenance stayed unfiltered per the deliberate decision above.
5. **Single-day range:** set `start` and `end` to the same date that you know has
   at least one brew. Confirm it's not excluded.
6. **UI error path:** manually type an end-date input before the start-date
   input (`from` after `to`) and click Apply. Confirm the error message renders
   near the filter bar and the previous dashboard state is not wiped out.
