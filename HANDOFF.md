# Naam Jap — Handoff
_Last updated: 2026-10-01_

Speech-activated counter for the word "Radha". The whole app is one file: `index.html`.
User: Kuldeep. Has no coding background. Long-term goal: paid iOS/Android app (React Native). For now: the web app.

- **Live:** https://cachauhankuldeep.github.io/naam-jap (GitHub Pages, auto-deploys on push, ~1–2 min)
- **Repo:** https://github.com/cachauhankuldeep/naam-jap
- **Deploy:** `git add index.html && git commit -m "..." && git push`, then hard-refresh (Cmd+Shift+R)
- **Local preview:** `.claude/launch.json` → `npx serve -p 5500 .` (localStorage doesn't work over `file://`)

---

## Status
| Area | State |
|------|-------|
| Desktop Chrome (speech recognition) | ✅ Working. Main use: 30–50K counts/day |
| Mobile (audio-level detection) | ⚠️ Not confirmed working. `#debugBox` shows `rms/ratio/ctx` for tuning |

---

## How counting works

### Desktop Chrome: `webkitSpeechRecognition`
- `continuous`, `interimResults`, `lang: 'hi-IN'`. Matches `PATTERNS` (Devanagari + Roman spellings).
- `creditedPerIndex[i]` = words already counted for result `i`. Only the difference is added, so a phrase is never double-counted.
- **One instance, restarted 50ms after each `onend`.** Rebuilding the instance makes Chrome ask for the mic again.
- **Never call `rec.stop()` while chanting.** Measured live (Sep 25): each restart = ~2s deaf
  (stop→end 0.5s, start→audiostart 0.3s, →first result 1.1s). A "recycle every 30 words" experiment
  made lag much worse and was reverted. Remaining short pauses are Chrome ending its own sessions.
- **Mic permission re-ask (~2300 counts):** `onerror` `not-allowed` sets `permPending`, shows `#permModal`, and retries `rec.start()` every 800ms until the user clicks Allow. Permanent fix for the user: lock icon → Microphone → Always allow.

### Mobile: Web Audio burst detection (`startBurstDetection`)
- Two moving averages of mic loudness: fast `fastEMA` and slow `slowEMA` (the room's background noise). A count fires when fast/slow > `RISE_RATIO` (1.8). The background level is frozen while speaking.
- The AudioContext must be created inside the tap handler, connected to `destination`, and `micStream` is kept alive between Stop/Start (all iOS requirements).
- Tuning: no counts → lower `RISE_RATIO` to 1.4–1.5; counts from noise → raise it to 2.0–2.5; misses fast chanting → lower `MIN_ONSET_GAP`.

---

## Performance rules (keep these)
- **`addCount()` → `renderCount()` runs synchronously.** Don't batch with `requestAnimationFrame`:
  Chrome pauses rAF when its window is covered by another app, freezing the display.
- **Transcript:** `setTranscript(text)` takes plain text, renders only the last 120 characters.
- **Saving:** `flushUnsaved()` is the single way to save pending counts. Normal mode saves after 3s of silence; backfill mode saves at most once a second.
- **Caches:** `_backfillResult` (15s), grand-card finish-date estimate throttled to every 2s, DOM nodes cached in `_el`. `savedGrandTotal` holds the total so `localStorage` isn't re-read on every count.
- **No `backdrop-filter` on cards.** It was invisible on the white background but was redrawn on every count.
- **Count flash colour is dark teal `#0f4743`, not white.** White made the number invisible on the white card during fast chanting.

---

## Page layout
Three columns on desktop, stacked on mobile (≤900px):
- **Left:** 1K countdown (`GAME_10K = 1000`, counts down, "Round N"), chanting clock, Today timer with ✕ reset
- **Middle:** Radha Count, target + progress bar, transcript, history
- **Right:** Maha Lakshya card (target 1 crore), with total, %, daily average and estimated finish date; Manual Entries card below it

Style: white background, text `#0f4743`, accent `#4d9e8c`. `<html lang="en">` must stay `en`: `hi` would switch every element to the Devanagari serif font. Numbers are formatted by `fmtIN()` in Western style (1,500,000). Count cards are a fixed 320px wide with `tabular-nums`, so the layout doesn't shift as digits change.

---

## Timers
- `getElapsed()` = seconds of active chanting only. A burst ends after `CHANT_PAUSE` (4s) of silence.
- `dailyChantSecs` is stored in `radha_jap_chant_time_YYYY-MM-DD`. It survives Reset, rolls over at midnight (`checkDateRollover`, checked every 60s), and the ✕ button clears it (`resetDailyChantTime`).

---

## Data
| localStorage key | Content |
|---|---|
| `radha_jap_history` | `{ "YYYY-MM-DD": count }` |
| `radha_jap_targets` | `{ "YYYY-MM-DD": target }` for back-filling past days |
| `radha_jap_chant_time_YYYY-MM-DD` | seconds chanted that day |
| `radha_jap_manual` | `{ "YYYY-MM": count }` manual monthly entries (Sep 2026 →) |

- **Manual entries** (card under Maha Lakshya): one input per month from `MANUAL_START` (2026-09) to the current month. The value is the month's figure (it replaces, not adds). `manualSum` is added **only** to the 1 Crore total, daily average and finish date, never to history, today's count or the 1K countdown (`updateGrandCard`: `totalDone = chanted + manualSum`, `updateGame10k(chanted)`). It is included in the backup file and merged on restore, keeping the higher value per month.
- **Backfill:** past days with a target that isn't met get new counts first, oldest first (`saveToHistory`). The rest goes to today.
- **Auto-save file:** File System Access API. The file handle is stored in IndexedDB (`naam_jap_db`). Writes every 30s and when the tab is hidden. Format: `{history, targets, manual, savedAt, version: 1}`.
- **Restore:** merges a backup file, keeping the higher count per date. Shown automatically if localStorage is empty.
