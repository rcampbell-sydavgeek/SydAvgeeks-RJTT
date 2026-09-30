# RJTT (Tokyo Haneda) Special Livery Arrival Watcher — Setup Guide

This is a replica of your YSSY arrivals watcher, retargeted at Tokyo
Haneda (RJTT). Same watchlist, same detection pipeline (destination
lookup → bearing sanity check → AeroDataBox cross-check → two-poll
confirmation for anything AeroDataBox can't confirm) — just a
different target airport.

## 1. Reuse your existing Google Sheet

No new sheet needed. This reads the exact same published
`registration`/`notes` CSV as your YSSY and departure watchers. Use
the same `GOOGLE_SHEET_CSV_URL` secret value.

## 2. ntfy

Decide whether you want Haneda arrivals in the same ntfy topic as
your other alerts, or a separate one so you can tell at a glance
which airport triggered it (the notification text names the airport
either way, since `TARGET_AIRPORT_ICAO` is baked into the message).
A separate topic per airport is probably easier to scan.

## 3. AeroDataBox — a shared budget, worth thinking about

Your AeroDataBox free tier is capped at 600 API units / 2400 requests
per month, and that's a **per-account** limit, not per-repo. If this
watcher uses the same `AERODATABOX_API_KEY` as your YSSY watcher, all
three repos (YSSY arrivals, this one, and RJAA) are drawing from the
same monthly pool — running all three on a 5-minute schedule could
burn through the quota noticeably faster than one repo alone.

Options:
- Share the key across all three and keep an eye on RapidAPI's usage
  dashboard.
- Leave `AERODATABOX_API_KEY` unset here — the watcher still works
  fine without it, just falls back to bearing-check-only plus the
  two-poll confirmation requirement for every match (i.e. everything
  behaves like the "unknown" verdict case).
- Get a second free RapidAPI account/key just for this watcher, if
  you want independent budgets.

## 4. Create the GitHub repository

1. New **private** repo (e.g. `rjtt-watcher`).
2. **Add file > Upload files** — drag in `watcher.py`,
   `requirements.txt`, `icao24_cache.json`, `notified.json`,
   `pending_confirmation.json`, and the whole `.github` folder.
   Double check `watch.yml` lands at exactly
   `.github/workflows/watch.yml` (not `.guthub`, not nested any
   deeper).
3. **Settings > Secrets and variables > Actions** (not "Agents" —
   that page's secrets aren't visible to workflows), add:
   - `GOOGLE_SHEET_CSV_URL` — same value as your other watchers
   - `NTFY_TOPIC` — same or a new topic, your choice (see above)
   - `AERODATABOX_API_KEY` — optional, see above
4. **Actions** tab, run the workflow manually once (Run workflow)
   to confirm it works before relying on the schedule.

## 5. Reliable triggering (cron-job.org)

GitHub's native `schedule:` trigger is unreliable under load, so use
an external pinger the same way as your other watchers:

1. GitHub Personal Access Token (fine-grained), scoped to
   `Actions: Read and write`, with this repo added to its
   repository access list. If you're reusing the same PAT as your
   other watchers, remember to add `rjtt-watcher` to its repo access
   list too — a 403 on the cron-job.org test usually means the token
   doesn't have this repo yet.
2. cron-job.org job, every 5 minutes, POST to:
   `https://api.github.com/repos/YOUR_USERNAME/rjtt-watcher/actions/workflows/watch.yml/dispatches`
   Headers: `Authorization: Bearer TOKEN`,
   `Accept: application/vnd.github+json`,
   `Content-Type: application/json`.
   Body: `{"ref":"main"}` (just the JSON — don't type the word
   "JSON" into the body field).
   Expect `204 No Content` on a test run.

## How it works

Same pipeline as the YSSY watcher:

1. Pulls your watchlist from the Google Sheet, resolves each
   registration to an ICAO24 hex (cached after first lookup).
2. Checks OpenSky for any of those aircraft currently airborne.
3. Looks up each matching callsign's destination. If it resolves to
   RJTT, runs a bearing sanity check (does the aircraft's actual
   heading roughly agree with the great-circle bearing to Haneda?).
4. If the bearing checks out, cross-checks against AeroDataBox's real
   schedule/live-tracking data (when the API key is set):
   - **Confirmed** → alerts immediately, with a real ETA if available.
   - **Contradicted** (AeroDataBox has data for this aircraft today,
     but none of it goes to RJTT) → suppressed, no alert.
   - **Unknown** (no AeroDataBox data — common for ad-hoc
     cargo/charter operators) → held pending; only alerts if the same
     match repeats on a later poll (at least ~3 minutes later), to
     filter out one-off coincidental bearing matches.
5. Records what's been notified so the same arrival doesn't re-alert
   for 20 hours.

State (`icao24_cache.json`, `notified.json`,
`pending_confirmation.json`) is committed back to the repo by the
workflow after each run.

## Known limitation

Haneda is a much busier, more complex arrival environment than YSSY,
with tighter approach spacing and holding patterns closer to the
airport — expect the bearing check to occasionally need the two-poll
confirmation path more often here than at YSSY, particularly for
aircraft still in a holding pattern when first detected (a holding
turn can transiently point away from the airport).
