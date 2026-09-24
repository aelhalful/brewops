# Implementation plan: CSV export of raw brew data

Ticket: `tickets/002-csv-export.md`. Goal: let finance download the raw brew data
(not the aggregated dashboard numbers) as a CSV file, so they stop screenshotting
the dashboard for the quarterly office-cost review.

Read this whole file before changing anything. It names every file and function you
need to touch, in the order to touch them.

## Contract

- **Scope**: raw, per-event rows from `brew_events` — one row per brew, not the
  aggregated `per_drink`/`per_day` summaries the dashboard uses. Finance wants to
  pivot this themselves in Excel, so give them the underlying data, not a
  pre-summarized view.
- **Endpoint**: `GET /api/export.csv`, optional `start`/`end` query params in the
  same `YYYY-MM-DD` format used by `/api/stats` (see `docs/plan-date-filter.md` if
  present, or `src/brewops/api/main.py`'s existing `validate_date_range` helper).
  Reuse that helper — don't write new date validation.
  - Both inclusive, both optional, same semantics as `/api/stats`: omit both for
    all-time, omit one for an open-ended bound on that side.
- **Response**: `Content-Type: text/csv`, with a `Content-Disposition:
  attachment; filename="brewops-export.csv"` header so the browser downloads the
  file directly instead of navigating to it. This means a plain `<a href=...>`
  link works client-side — no JS fetch/blob handling needed.
- **Columns** (one row per brew event, in this order):
  `timestamp, machine, drink_type, drink_label, duration_s, temp_c, source`
  - `timestamp`: as stored, `YYYY-MM-DD HH:MM:SS` (naive local time — see the
    module docstring in `src/brewops/db/queries.py`). Do not reformat it; finance
    can format dates themselves in Excel, and reformatting risks a timezone bug.
  - `machine`: the machine's human-readable `name` (e.g. `"Bertha (3rd floor)"`),
    not its numeric `id`. Join against the `machines` table.
  - `drink_type`: the machine-readable key (e.g. `espresso`).
  - `drink_label`: the human-readable label (e.g. `Espresso`). Join against
    `drink_types`.
  - `duration_s`, `temp_c`: pass through as-is; both are nullable (`Old Faithful`'s
    manually-logged brews will have `NULL`/empty values here since it has no
    telemetry — see `has_telemetry` on the `machines` table). Empty CSV cell for
    `NULL`, not the string `"None"` or `"null"`.
  - `source`: `'csv'` or `'manual'`, as stored.
  - Rows ordered by `timestamp` ascending.
- **No pagination**: this is a small office-fleet dataset (see the existing
  `MACHINES`/`DRINK_TYPES` seed sizes in `src/brewops/db/schema.py`). Return the
  full result set in one response. Don't add a `limit`/`offset` — out of scope and
  unneeded at this data volume.

## Backend changes

### `src/brewops/db/queries.py`

Add a new function, near `get_stats`/`get_machine_health` (currently around line
68-189): `get_brew_events_for_export(conn, start=None, end=None)`.

- Build the same `WHERE DATE(be.timestamp) ...` filter fragment used by
  `get_stats`/`get_machine_health` in this file — copy the existing
  `if start and end / elif start / elif end` pattern for consistency (it's
  already duplicated between those two functions; don't take this as license to
  add a third copy carelessly — keep the SQL fragment identical in spirit so a
  future refactor into a shared helper stays easy).
- Query:
  ```sql
  SELECT be.timestamp, m.name AS machine, dt.name AS drink_type, dt.label AS drink_label,
         be.duration_s, be.temp_c, be.source
  FROM brew_events be
  JOIN machines m ON m.id = be.machine_id
  JOIN drink_types dt ON dt.name = be.drink_type
  [WHERE ...]
  ORDER BY be.timestamp
  ```
- Use `JOIN`, not `LEFT JOIN` — every `brew_events` row has a valid `machine_id`
  and `drink_type` by foreign-key/CHECK constraint (see `schema.py`), so there's
  no "unmatched row" case to preserve here, unlike `get_stats`'s deliberate
  `LEFT JOIN` for `per_drink`.
- Return `list[dict[str, Any]]`, same shape convention as the rest of the file
  (`[dict(r) for r in conn.execute(...)]`).

### `src/brewops/api/main.py`

Add a new route near the other `GET` routes (currently `/api/stats` at line ~89,
`/api/machines/{machine_id}` at line ~104):

```python
from fastapi.responses import StreamingResponse
import csv
import io

@app.get("/api/export.csv")
def export_csv(
    start: str | None = None,
    end: str | None = None,
    conn: sqlite3.Connection = Depends(get_db),
):
    validate_date_range(start, end)
    rows = queries.get_brew_events_for_export(conn, start=start, end=end)

    buffer = io.StringIO()
    writer = csv.DictWriter(
        buffer,
        fieldnames=["timestamp", "machine", "drink_type", "drink_label", "duration_s", "temp_c", "source"],
    )
    writer.writeheader()
    writer.writerows(rows)
    buffer.seek(0)

    return StreamingResponse(
        buffer,
        media_type="text/csv",
        headers={"Content-Disposition": 'attachment; filename="brewops-export.csv"'},
    )
```

- `csv.DictWriter` writes `None` values as empty strings by default — this is
  exactly the "empty cell for NULL" behavior the contract above asks for. Don't
  add manual `None`-to-`""` conversion; it's redundant.
- Reuse `validate_date_range` exactly as `/api/stats` and `/api/machines/{id}` do
  — same 400 behavior for a bad date or `start > end`, no new validation logic.
- Import `csv` and `io` from the standard library — no new dependency. Both
  `fastapi` and `StreamingResponse` are already available (`fastapi` is an
  existing dependency; `StreamingResponse` lives in `fastapi.responses`, add that
  import alongside the existing `fastapi` imports at the top of the file).

## Frontend changes

### `src/brewops/frontend/index.html`

Inside `<section id="dashboard">`, next to the existing `.date-filter` block
(currently lines 18-26), add a download link:

```html
<a id="export-link" href="/api/export.csv" download>Download CSV</a>
```

- Use a real `<a>` tag with `download`, not a button with a JS click handler —
  the `Content-Disposition` header on the response already triggers a save
  dialog / direct download, and a plain link lets the browser handle it natively
  with no fetch/blob code.
- Place it near the filter bar (same visual area as `#filter-start`/`#filter-end`)
  so it's clear the export respects whatever range is currently applied.

### `src/brewops/frontend/app.js`

- The export link's `href` needs to track the currently-applied `start`/`end`,
  the same query-string values `loadDashboard(start, end)` already builds
  (currently lines 82-86). Update `#export-link`'s `href` in the same place
  `loadDashboard` builds `queryString` — set
  `document.getElementById("export-link").href = `/api/export.csv${queryString}`;`
  right after computing `queryString`.
- This means every place that calls `loadDashboard` (initial page load at the
  bottom of the file, the `#filter-apply` handler, the `#filter-reset` handler,
  and `submitForm`'s post-submit refresh) automatically keeps the export link in
  sync — no separate wiring needed at each call site.
- Do not build the query string a second time in a separate function; reuse the
  one already computed inside `loadDashboard` for this request.

## Edge cases checklist

1. **No range selected (all params omitted)** — export contains every brew event
   ever logged, oldest first.
2. **Range with zero brews in it** — response is a CSV with just the header row,
   not an empty file and not an error. Confirm `csv.DictWriter.writerows([])`
   still writes the header.
3. **`start` after `end`** — API returns `400` (via `validate_date_range`), same
   as `/api/stats`. The browser will show its normal failed-navigation behavior
   for a direct link; that's acceptable since this mirrors existing API error
   behavior elsewhere in the app — no special client-side handling required.
4. **`Old Faithful` brews (`has_telemetry = false`, manually logged)** — these
   rows have `source = 'manual'` and likely `NULL` `duration_s`/`temp_c`. Confirm
   they appear in the export with empty (not `"None"`) cells for those columns.
5. **Values containing commas or quotes** — none of the current columns are
   free-text except `machine` name and `drink_label`, which come from seeded
   reference data (`schema.py`) and don't contain commas today. Still, rely on
   `csv.writer`'s built-in quoting (via `DictWriter`) rather than hand-building
   CSV strings — this is handled automatically by the stdlib `csv` module, not
   something to special-case.

## How to verify

Run the app: `uv run start`, open `http://localhost:8123`.

1. **Manual export via curl**, to isolate backend from frontend:
   - `curl "http://localhost:8123/api/export.csv" -o all.csv` — confirm the file
     has a header row plus one row per brew event; row count should equal
     `total_brews` from `curl "http://localhost:8123/api/stats"`.
   - `curl "http://localhost:8123/api/export.csv?start=2026-09-01&end=2026-09-07" -o range.csv`
     — confirm row count matches
     `sqlite3 <db file> "SELECT COUNT(*) FROM brew_events WHERE DATE(timestamp) BETWEEN '2026-09-01' AND '2026-09-07'"`
     (check `src/brewops/db/connection.py` for the actual db file path).
   - `curl -i "http://localhost:8123/api/export.csv?start=2026-09-07&end=2026-09-01"`
     — confirm HTTP 400, not a 200 with an empty or malformed CSV.
   - `curl -i "http://localhost:8123/api/export.csv"` — confirm the response
     headers include `content-type: text/csv` and
     `content-disposition: attachment; filename="brewops-export.csv"`.
2. **Open the CSV in a spreadsheet tool** (Excel or similar) to confirm it opens
   cleanly with correct column headers and no encoding/quoting artifacts.
3. **Click the download link in the browser** with a date range applied via the
   existing filter UI; confirm the downloaded file's row count matches the
   filtered range, not the full dataset, and that the file actually downloads
   (not just navigates the tab).
4. **Click the download link with no filter applied** (fresh page load, before
   touching the date inputs) — confirm it downloads all-time data.
