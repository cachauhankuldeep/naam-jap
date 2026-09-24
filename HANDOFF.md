# Naam Jap — Handoff Document
_Last updated: 2026-09-24_

---

## What This App Does
A speech-activated counter that counts the word "Radha" when the user chants it. Single HTML file, runs in browser. User: Kuldeep, zero coding knowledge, wants it to eventually become a paid app on App Store + Google Play (Path B). For now finishing Path A (web app).

---

## Live URLs
- **GitHub Pages (iPhone-accessible):** https://cachauhankuldeep.github.io/naam-jap
- **GitHub repo:** https://github.com/cachauhankuldeep/naam-jap.git
- **Local file:** `/Users/kuldeepmac/App_Coding/Naam_Jap/index.html`
- **Git push command:** `git add index.html && git commit -m "..." && git push`

---

## Page Layout (Desktop — 3-column flex)

```
[ milestone-col auto ] [ main-col flex:1 ] [ grand-card 210px ]
  1K countdown           counter card         Maha Lakshya
  chanting clock         progress bar         (50% size)
  today timer            transcript           (sticky right)
  (sticky left)          history
```

```css
.page-cols     { display: flex; flex-direction: row; align-items: flex-start; max-width: 1160px; gap: 36px; }
.milestone-col { flex: 0 0 auto; min-width: 240px; display: flex; flex-direction: column; gap: 14px; position: sticky; top: 32px; align-self: flex-start; }
.main-col      { flex: 1; min-width: 0; display: flex; flex-direction: column; align-items: center; }
.grand-card    { flex: 0 0 210px; width: 210px; position: sticky; top: 32px; }
```

**Mobile (≤900px):** milestone-col becomes a horizontal flex strip below main content.
**Mobile (≤520px):** milestone-col stacks vertically again.

---

## Color Scheme

| Role | Value |
|------|-------|
| Body background | `#ffffff` |
| Body text | `#0f4743` (dark teal) |
| Accent / count numbers | `#4d9e8c` (muted teal — matched to user's reference image) |
| Card borders | `rgba(26,94,82,0.2)` dark green |
| Card backgrounds | `rgba(26,94,82,0.04)` very light green tint |
| Box shadows | `rgba(26,94,82,0.08)` |

---

## Typography

```html
<html lang="en">  <!-- MUST be "en" — "hi" makes :lang(hi) apply Devanagari serif to ALL elements -->
<link href="https://fonts.googleapis.com/css2?family=DM+Sans:wght@300;400;500;600;700&family=Inter:wght@300;400;500;600;700&family=Tiro+Devanagari+Hindi:ital@0;1&display=swap" rel="stylesheet">
```

```css
body {
    font-family: 'Aptos', 'Aptos Display', 'Inter', 'DM Sans', 'Segoe UI', system-ui, sans-serif;
    color: #0f4743;
}
.deity-name, [lang="hi"], :lang(hi) {
    font-family: 'Tiro Devanagari Hindi', 'Noto Sans Devanagari', serif;
}
```

---

## Number Formatting

```javascript
// Western format: 1500000 → "1,500,000" (NOT Indian "15,00,000")
function fmtIN(n) {
    n = Math.round(n);
    if (n < 0) return '-' + fmtIN(-n);
    return String(n).replace(/\B(?=(\d{3})+(?!\d))/g, ',');
}
```
All number displays use `fmtIN()`. No `toLocaleString()` calls remain in the codebase.

---

## Counter Cards (Left Column)

### Radha Count card
```css
.counter-card {
    background: rgba(26,94,82,0.04);
    border: 1px solid rgba(26,94,82,0.2);
    border-radius: 24px;
    padding: 28px 52px 24px;
    width: 320px;       /* fixed — prevents shake as digits change */
    overflow: hidden;
}
.count-number {
    font-size: 6rem;
    font-weight: 700;
    color: #4d9e8c;
    font-variant-numeric: tabular-nums;  /* prevents layout shift */
}
```

### Dynamic font scaling (prevents overflow for large numbers)
```javascript
const _FONT_SIZES = [6, 6, 6, 6, 4.5, 3.8, 3.0, 2.5, 2.0, 1.7, 1.5]; // indexed by digit count
function fitCountFont() {
    const digits = count > 0 ? (Math.floor(Math.log10(count)) + 1) : 1;
    el.style.fontSize = (_FONT_SIZES[Math.min(digits, _FONT_SIZES.length - 1)] || 1.5) + 'rem';
}
function fitGame10kFont(rem) {
    const digits = rem > 0 ? (Math.floor(Math.log10(rem)) + 1) : 1;
    el.style.fontSize = (_FONT_SIZES[Math.min(digits, 4)] || 4.5) + 'rem';
}
```

### 1K Countdown card (`GAME_10K = 1000`)
```css
.game10k-card {
    border-color: rgba(26,94,82,0.5);
    padding: 28px 52px 24px;
    width: 320px;       /* same size as Radha Count card */
    overflow: hidden;
}
.game10k-card .block-mini-num { font-size: 6rem; font-variant-numeric: tabular-nums; }
```
- Counts down from 1000 to 0, then resets (Round N++)
- `GAME_10K = 1000` constant

---

## Chanting Clock + Today Timer (Left Column, below 1K card)

### HTML
```html
<div class="stats" id="stats">
  <div class="clock-ring" id="clockRing">
    <span class="clock-label">Chanting</span>
    <span class="clock-time" id="clockTime">--:--</span>
  </div>
  <div class="today-summary">
    <div class="today-summary-label" style="display:flex;align-items:center;justify-content:center;gap:6px;">
      Today
      <button onclick="resetDailyChantTime()" title="Reset today's timer" ...>✕</button>
    </div>
    <div class="today-summary-row">
      <span class="today-val" id="dailyChantTime">–</span>
    </div>
  </div>
</div>
```

### Daily chanting time state
```javascript
let dailyChantSecs     = 0;   // total chanting seconds for today (persists across session resets)
let sessionFlushedSecs = 0;   // how much of this session is already added to dailyChantSecs
let dailyChantDate     = '';  // YYYY-MM-DD key for current day (set on init)
```

### Midnight rollover
```javascript
function checkDateRollover() {
    const today = todayKey();
    if (today === dailyChantDate) return;
    // Flush remaining seconds to old day's key
    const secs = getElapsed();
    const delta = secs - sessionFlushedSecs;
    if (delta > 0) {
        dailyChantSecs += delta;
        localStorage.setItem('radha_jap_chant_time_' + dailyChantDate, dailyChantSecs);
    }
    // Reset for new day
    dailyChantDate     = today;
    sessionFlushedSecs = secs;
    dailyChantSecs     = parseInt(localStorage.getItem('radha_jap_chant_time_' + today) || '0', 10);
    updateTodaySummary();
}
// Called at top of flushDailyChantTime() AND via setInterval every 60s (catches idle pages at midnight)
setInterval(checkDateRollover, 60000);
```

### Manual reset button
```javascript
function resetDailyChantTime() {
    dailyChantSecs     = 0;
    sessionFlushedSecs = getElapsed();
    localStorage.removeItem('radha_jap_chant_time_' + dailyChantDate);
    updateTodaySummary();
}
```
The ✕ button resets both in-memory value AND localStorage atomically (unlike external deletion, which gets overwritten by beforeunload).

### How the timer works
- `chantElapsed` (ms) — accumulated from completed bursts
- `chantSegStart` — timestamp when current burst began
- `chantLastHit` — timestamp of last count
- `CHANT_PAUSE = 4000ms` — silence threshold; after 4s of no counts, burst ends
- `getElapsed()` → seconds of active chanting (not idle time)
- `sessionFlushedSecs` resets on `resetAll()` but `dailyChantSecs` persists

---

## Performance Caches (Critical — do not remove)

```javascript
let _backfillExpiry = 0;   // timestamp when _backfillResult expires
let _backfillResult = null; // cached getActiveBackfillDate() result
let _grandEtaExpiry = 0;   // timestamp when grand card ETA was last updated
```

**Why these exist:**
- `addCount()` was calling `getActiveBackfillDate()` on every count → 2x `localStorage.getItem` + `JSON.parse` per count
- `updateGrandCard()` was calling `toLocaleDateString()` + `innerHTML =` on every count → heavy Intl + DOM reparse
- With rapid speech recognition, this caused visible counting lag
- Fix: backfill cached for 15s (invalidated on save); ETA throttled to max once per 2s
- Both caches invalidated in `saveToHistory()` via `_backfillExpiry = 0; _grandEtaExpiry = 0;`

---

## Maha Lakshya (1 Crore Grand Target) — Right Column
- **Target:** 10,000,000 (1 crore) — displayed as `of 10,000,000`
- **Card size: 210px wide**
- Shows: total done, percentage, progress bar, daily avg, estimated completion date
- `updateGrandCard()` fast path (count/pct/bar) runs on every `addCount()`
- ETA section (`toLocaleDateString` + `innerHTML`) throttled to max every 2s

---

## Architecture — Platform Detection

```javascript
const hasSR    = !!(window.SpeechRecognition || window.webkitSpeechRecognition);
const isMobile = /Android|iPad|iPhone|iPod/i.test(navigator.userAgent) && !window.MSStream;
const useSR    = hasSR && !isMobile;   // desktop Chrome only
```

### Path 1 — Desktop Chrome (`useSR = true`)
- Uses **Web Speech Recognition API** (`webkitSpeechRecognition`)
- `continuous: true`, `interimResults: true`, `lang: 'hi-IN'`
- Counts by matching transcript against `PATTERNS` (Devanagari + Roman variants)
- Same `rec` instance reused on every `onend` restart — 50ms gap between sessions
- **Chrome permission re-ask at ~2300 counts:** Chrome shows browser-level permission bar
  - App does NOT stop; status shows **"⚠️ Click Allow in browser bar ↑"**
  - Full-screen modal (`#permModal`) appears with instructions
  - Independent `permRetryInterval` (setInterval 800ms) retries `rec.start()`
  - Once user clicks Allow → `onstart` fires → counting resumes automatically
  - **Permanent fix:** Chrome lock icon → Microphone → Always Allow

### Path 2 — ALL Mobile (`useSR = false`)
- Uses **Web Audio API** burst detection (VAD) — see VAD section below
- `startListening()` creates AudioContext + calls `startBurstDetection(stream, actx)`
- `stopListening()` calls `stopBurstDetection()`
- `micStream` kept alive between Stop/Start to avoid repeated permission dialogs

---

## Mobile VAD Algorithm — Current State (Iteration 4)

**Status: NOT YET CONFIRMED WORKING. User has not tested iteration 4.**

### Constants:
```javascript
const RISE_RATIO    = 1.8;
const FALL_RATIO    = 1.4;
const MIN_ABS       = 0.003;
const MAX_SPEECH_MS = 500;
const MIN_ONSET_GAP = 150;
```

### Algorithm:
```javascript
// WARMUP: time-based (400ms grace)
lastCountTime = Date.now() + 400;

let fastEMA = 0, slowEMA = 0;
let state = 'IDLE', speechEntry = 0;

function tick() {
    if (!burstCtx) return;
    if (burstCtx.state !== 'running') burstCtx.resume();  // KEY: no early return on suspended

    analyser.getByteTimeDomainData(buf);
    let sum = 0;
    for (let i = 0; i < buf.length; i++) { const v = (buf[i]-128)/128; sum += v*v; }
    const rms = Math.sqrt(sum / buf.length);
    const now = Date.now();

    fastEMA = fastEMA * 0.25 + rms * 0.75;

    if (state === 'IDLE') {
        slowEMA = slowEMA * 0.85 + rms * 0.15;
        const ratio = slowEMA > 0.001 ? fastEMA / slowEMA : 1;
        if (ratio > RISE_RATIO && fastEMA > MIN_ABS && (now - lastCountTime) > MIN_ONSET_GAP) {
            state = 'ACTIVE'; speechEntry = now; addCount(1); lastCountTime = now;
        }
    } else {
        if (fastEMA < slowEMA * FALL_RATIO || (now - speechEntry) > MAX_SPEECH_MS) state = 'IDLE';
    }
    burstTimer = requestAnimationFrame(tick);
}
```

### Tuning guide:
| Symptom | Fix |
|---------|-----|
| Zero counts, `ctx: suspended` throughout | AudioContext not resuming |
| `ratio` never exceeds 1.8 when chanting | Lower `RISE_RATIO` to 1.4–1.5 |
| Over-counting ambient noise | Raise `RISE_RATIO` to 2.0–2.5 or raise `MIN_ABS` |
| Under-counting fast chanting | Lower `MIN_ONSET_GAP` to 100ms |

---

## Desktop Speech Recognition — Word Patterns
```javascript
const PATTERNS = [
    /राधा/g, /राधे/g, /राधी/g, /रधा/g, /रहा/g, /रहे/g,
    /\b(?:radha|radhaa|radhae|radhe|radhi|radhai|radhey|radho)\b/gi,
    /\b(?:rada|raada|rade|radi|rado)\b/gi,
    /\b(?:raha|rahaa|rahe|rahi|raho|rahay)\b/gi,
    /\b(?:ratha|rathe|rathi|ratho)\b/gi,
];
```

---

## localStorage Keys
| Key | Purpose |
|-----|---------|
| `radha_jap_history` | `{ "YYYY-MM-DD": count }` — daily chant history |
| `radha_jap_targets` | `{ "YYYY-MM-DD": target }` — manually set per-date targets |
| `radha_jap_chant_time_YYYY-MM-DD` | seconds of active chanting for that date (resets at midnight) |

---

## Key State Variables
```javascript
let count              = 0;
let recognition        = null;    // SR instance (reused on desktop)
let isListening        = false;
let permPending        = false;
let permRetryInterval  = null;
let creditedPerIndex   = {};
let micStream          = null;
let savedGrandTotal    = 0;
let unsavedCount       = 0;
let saveDebounce       = null;
let chantElapsed       = 0;       // accumulated ms from completed chant bursts
let chantSegStart      = null;    // Date.now() when current burst began
let chantLastHit       = null;    // Date.now() of most recent count
const CHANT_PAUSE      = 4000;    // ms of silence before pausing timer
let dailyChantSecs     = 0;
let sessionFlushedSecs = 0;
let dailyChantDate     = '';
let _backfillExpiry    = 0;       // performance cache for getActiveBackfillDate()
let _backfillResult    = null;
let _grandEtaExpiry    = 0;       // throttle for grand card ETA updates
const GAME_10K         = 1000;    // 1K countdown size
const GRAND_TARGET     = 10000000; // 1 crore
```

---

## Past-Date Target Backfill System
- In history panel, each past date has a target input (saved on Enter or blur)
- `saveTarget(dateKey, rawVal)` → stores in `radha_jap_targets`
- `getActiveBackfillDate()` → returns oldest unfulfilled past-date target
- `saveToHistory(n)` → routes counts to oldest unfulfilled target first, remainder to today
- When backfill active → save immediately; normal mode → debounce 3s

---

## Data Safety — Auto-Save + Restore System

### Auto-Save (File System Access API)
- **💾 Set Auto-Save File** button at bottom of app
- `FileSystemFileHandle` stored in IndexedDB (`naam_jap_db` / `handles` / key `auto_save_handle`)
- Saves every 30 seconds + on `visibilitychange` + on `beforeunload`
- Format: `{ history: {...}, targets: {...}, savedAt: "ISO string", version: 1 }`

### Restore from Backup
- **📂 Restore from Backup** always visible
- Auto-shown when localStorage is empty on load
- Smart merge: keeps whichever count is higher per date

---

## Recent Commit History (this session)
| Commit | What changed |
|--------|-------------|
| `fd13ca5` | Fix number overflow: Western format, dynamic font scaling |
| `9a74ba7` | Midnight rollover: reset daily chanting time at 12 AM |
| `782ade7` | Fix hardcoded Indian format `of 1,00,00,000` → `of 10,000,000` |
| `35e8ce7` | Add ✕ reset button for Today chanting timer |
| `065fe89` | Fix counting lag: cache backfill check, throttle ETA updates |

---

## Known Issues / Status

### Desktop Chrome — Working well ✅
- Counting accurate; lag fixed (backfill cache + ETA throttle)
- Chrome permission re-ask at ~2300 counts handled gracefully

### Mobile — NOT WORKING ⚠️
- All mobile uses VAD (iteration 4 pushed, not yet tested by user)
- **Next step:** User tests at https://cachauhankuldeep.github.io/naam-jap (hard refresh: hold reload on iPhone)
- Look at `#debugBox` while chanting and report `rms`, `ratio`, `ctx` values
- After mobile working: remove `#debugBox` and its population code from `startBurstDetection()`

### Path B (future)
- User's end goal: paid app on App Store + Google Play
- Requires React Native with `SFSpeechRecognizer` (iOS) + Google Speech-to-Text (Android)

---

## Key Technical Decisions
1. **Desktop SR: same instance reuse** — rebuilding triggers Chrome permission re-ask earlier
2. **permPending + independent setInterval** — keeps app alive during Chrome permission bar
3. **ALL mobile → VAD** — SR has 0.5–2s cloud round-trip lag on mobile
4. **AudioContext in tap handler** — iOS/CriOS suspends AudioContext from async callbacks
5. **Destination connection required** — iOS Web Audio graph doesn't run without destination
6. **micStream kept alive** — releasing getUserMedia triggers new permission dialog on iOS
7. **No early return on suspended AudioContext** — iter 3 bug caused zero counts on iOS
8. **Time-based warmup (400ms)** — replaced frame-count WARMUP which broke on suspended ctx
9. **slowEMA frozen in ACTIVE** — prevents chant energy from polluting ambient baseline
10. **`font-variant-numeric: tabular-nums`** — prevents card width changing as digits update
11. **Fixed 320px card width** — both Radha Count and 1K card; prevents layout shake
12. **`<html lang="en">`** — must NOT be `"hi"`: `:lang(hi)` would apply Tiro Devanagari (serif) to ALL elements
13. **Performance caches** — `_backfillResult` (15s) and `_grandEtaExpiry` (2s) prevent main-thread blocking on every count
14. **Western number format** — `fmtIN()` uses `/\B(?=(\d{3})+(?!\d))/g` regex; no `toLocaleString` calls
15. **Midnight rollover via `checkDateRollover()`** — called every tick + every 60s via setInterval; saves to old date's key, resets for new day
16. **resetDailyChantTime() for Today ✕ button** — resets both in-memory and localStorage atomically; external localStorage deletion alone doesn't work because `beforeunload` re-writes the in-memory value

---

## File Structure
```
/Users/kuldeepmac/App_Coding/Naam_Jap/
├── index.html              ← entire app (single file, ~2200 lines)
├── HANDOFF.md              ← this file
└── backup/
    └── index_backup_23-Apr-2026.html
```

---

## How to Deploy
```bash
cd /Users/kuldeepmac/App_Coding/Naam_Jap
git add index.html
git commit -m "description"
git push
# GitHub Pages auto-deploys in ~1-2 minutes
# Hard refresh: Cmd+Shift+R (Mac) or hold reload button (iPhone)
```
