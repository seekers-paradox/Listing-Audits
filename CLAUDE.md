# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A NAP (Name, Address, Phone) audit tool: it reads a CSV of business listings, looks each one up via the Google Places API, scores how well the local data matches what Google has, and optionally asks OpenAI to adjudicate ambiguous cases. Results are written to a new CSV.

This is a small, flat-file Python script — no package structure, no tests, no CI, no build step.

## Setup and running

```bash
pip install -r requirements.txt
```

Requires a `.env` file (gitignored, not present in the repo) with:
```
GOOGLE_PLACES_API_KEY=...
OPENAI_API_KEY=...        # optional — without it, AI verification is skipped
```

Run the audit:
```bash
python main.py
```

There are no automated tests, linter, or build/CI configuration in this repo. Verifying a change means running `main.py` against a CSV and inspecting the output/console log.

Note: `requirements.txt` is UTF-16 encoded (unusual for this ecosystem) — read/edit tools must handle that encoding, and `pip install -r requirements.txt` still works fine since pip handles it. The pinned `openai==0.28` version matters: `ai_matcher.py` uses the pre-1.0 `openai.ChatCompletion.create(...)` API, which is a breaking-incompatible surface from `openai>=1.0`.

## Configuration

All tunable behavior lives in `config.py` (`Config` dataclass), not scattered through the code:
- `INPUT_CSV` / `OUTPUT_CSV` — file paths (currently hardcoded to `MT15_data_export.csv` / `nap_audit_results_cleaned.csv`, overriding the README's generic example names)
- `NAME_MATCH_THRESHOLD` / `ADDRESS_MATCH_THRESHOLD` — similarity cutoffs (0-1) for declaring a match
- `REQUEST_DELAY` / `REQUEST_TIMEOUT` — Google Places API rate limiting/timeout

## Architecture / data flow

The pipeline is a straight chain, one module per stage:

```
main.py
  -> NAPAuditProcessor.process_csv()          [nap_audit_processor.py]
       reads INPUT_CSV row by row
       -> BusinessData.from_csv_row(row)        [models.py]   parses/normalizes one input row
       -> GooglePlacesClient.search_and_get_details(query)   [google_places_client.py]
            Text Search API -> take first result -> Place Details API -> PlaceData
       -> NAPMatcher.match_business_to_place(business, place)  [nap_matcher.py]
            delegates field comparisons to TextMatcher          [text_utils.py]
            combines name/address/phone match+similarity into an overall_status string
       -> if overall_status is PARTIAL or FAIL:
            AIBusinessMatcher.is_match(...)     [ai_matcher.py]  — OpenAI GPT-4 tiebreaker,
            logged to console but NOT currently folded back into overall_status or the output CSV
  -> NAPAuditProcessor.save_results(results)   writes one row per MatchResult to OUTPUT_CSV
```

Key structural points:
- `models.py` defines the three data shapes that flow through the whole pipeline: `BusinessData` (input row), `PlaceData` (Google API result), `MatchResult` (final verdict, with `to_dict()` defining the output CSV's columns).
- Matching is two-layered: `text_utils.TextMatcher` has the low-level string/phone similarity primitives (difflib ratio, phone digit normalization, component-based address matching); `nap_matcher.NAPMatcher` applies thresholds from `Config` and turns the three booleans/similarities into one of the fixed `overall_status` strings (`SUCCESS - ...`, `PARTIAL - ...`, `FAIL - ...`, `ERROR - ...`).
- Google Places lookup always takes only the top text-search result (`search_and_get_details` in `google_places_client.py`) — there's no candidate re-ranking.
- Errors during a single row's processing are caught per-record in `process_csv` so one bad row doesn't abort the whole run; it becomes an `ERROR - <message>` result instead.
- `MT15_data_export.csv` and `nap_audit_results_cleaned.csv` in the repo root are real data files (sample input/output), and `Logs.txt` is a captured run log — treat them as data/artifacts, not code to modify.
