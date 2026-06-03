# Naam Jap — Handoff Document
_Last updated: 2026-06-03_

---

## What This App Does
A speech-activated counter that counts the word "Radha" when the user chants it. Single HTML file, runs in browser. User: Kuldeep, zero coding knowledge, wants it to eventually become a paid app on App Store + Google Play (Path B). For now finishing Path A (web app).

---

## Live URLs
- **GitHub Pages (iPhone-accessible):** https://cachauhankuldeep.github.io/naam-jap
- **GitHub repo:** https://github.com/cachauhankuldeep/naam-jap.git
- **Local file:** `/Users/kuldeepmac/App_Coding/Naam_Jap/index.html`
- **Git push command:** `cd /Users/kuldeepmac/App_Coding/Naam_Jap && git add index.html && git commit -m "..." && git push`

---

## Page Layout (Desktop)

Two-column flex layout (`.page-cols`):
- **Left column** (`.main-col`): counter card, progress bar, target, controls, transcript, history, auto-save buttons
- **Right column** (`.grand-card`): Maha Lakshya panel — 420px wide, `position: sticky; top: 32px` so it stays visible while scrolling

On mobile (≤900px): grand-card switches to `position: fixed` bottom strip, main-col takes full width.

```css
.page-cols { display: flex; flex-direction: row; align-items: flex-start; max-width: 1160px; gap: 36px; }
.main-col  { flex: 1; min-width: 0; display: flex; flex-direction: column; align-items: center; }
.grand-card { flex: 0 0 420px; position: sticky; top: 32px; }
@media (max-width: 900px) { .page-cols { flex-direction: column; } .grand-card { position: fixed; bottom: 0; ... } }
```

---

## Architecture — Three Paths Based on Browser Detection

```javascript
const hasSR    = !!(window.SpeechRecognition || window.webkitSpeechRecognition);
const isMobile = /Android|iPad|iPhone|iPod/i.test(navigator.userAgent) && !window.MSStream;
```

### Path 1 — Desktop Chrome (hasSR=true, isMobile=false)
- Uses **Web Speech Recognition API** (`webkitSpeechRecognition`)
- `continuous: true`, `interimResults: true`, `lang: 'hi-IN'`
- Counts by matching transcript against `PATTERNS` (Devanagari + Roman variants)
- Same `rec` instance reused on every `onend` restart — 50ms gap between sessions
- **Chrome permission re-ask at ~2300 counts:** Chrome's internal session expires and shows a browser-level permission bar. This is a Chrome security feature — cannot be suppressed from JS. Handled gracefully:
  - App does NOT stop (green dot turns amber)
  - Status shows **"⚠️ Click Allow in browser bar ↑"**
  - Full-screen modal (`#permModal`) appears with instructions
  - Independent `permRetryInterval` (setInterval 800ms) retries `rec.start()` — does not rely on `onend`
  - Once user clicks Allow → `onstart` fires → `permPending` cleared → counting resumes, dot goes green
  - User never needs to press the Start button again
  - **Permanent fix tip shown in modal:** Chrome lock icon → Microphone → Always Allow

### Path 2 — Safari iOS / Android Chrome (hasSR=true, isMobile=true)
- Uses **Web Speech Recognition API** (same as desktop)
- **Repetition blindness fix:** rebuilds recognition instance every 7 counts (`srCycleCount` + `srDoReset` flags)
  - Speech engines stop recognising the same repeated word after ~8–10 repeats
  - Rebuild resets engine memory before blindness kicks in
  - 80ms gap on each rebuild
- Status: works but not thoroughly tested recently

### Path 3 — Chrome on iOS / CriOS (hasSR=false)
- Chrome on iPhone has NO `webkitSpeechRecognition` → falls to burst detection
- Uses **Web Audio API** (`AudioContext` + `AnalyserNode`) — counts by detecting speech onset (rising edge of mic volume)
- `getUserMedia` with `echoCancellation:false, noiseSuppression:false, autoGainControl:false`
- AudioContext created AND `resume()`d **synchronously in the tap handler** — CriOS suspends it if created async
- Audio graph: `source → analyser → gain(0) → destination` — destination connection required for iOS to run the graph
- `micStream` kept **alive between Stop/Start** — releasing it triggers repeated permission dialogs
- Status: **problematic** — user reports unreliable counting. Root cause not resolved.

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

## Mobile Burst Detection Algorithm (Path 3 only)
```
CALIB_MS      = 1500   // 1.5s ambient noise measurement on each Start
MAX_SPEECH_MS = 300    // force-exit speech state after 300ms (handles continuous chanting)
MIN_ONSET_GAP = 150    // min ms between counts

// Auto-calibration (runs for first 1.5s after Start — user must stay silent):
avg = mean of ambient RMS samples
dynHigh = clamp(avg * 3 + 0.002, 0.003, 0.015)   // entry threshold
dynLow  = dynHigh * 0.35                           // exit threshold

// Tick loop (requestAnimationFrame — more reliable than setInterval on mobile):
if (AudioContext.state !== 'running') → resume and skip frame
smoothRMS = smoothRMS * 0.35 + rms * 0.65

if (!inSpeech && smoothRMS > dynHigh && gap > MIN_ONSET_GAP):
    inSpeech = true; addCount(1)
elif inSpeech:
    if smoothRMS < dynLow OR elapsed > MAX_SPEECH_MS:
        inSpeech = false
```

**Tuning guide (if burst detection is under/over counting):**
- Under-counting → lower `dynHigh` floor: change `Math.max(0.003, ...)` to `0.002`
- Over-counting / noise triggers → raise multiplier from `avg * 3` to `avg * 4`
- Double-counting → increase `MIN_ONSET_GAP` to 200–250ms
- Missing fast chanting → decrease `MAX_SPEECH_MS` to 200ms and `MIN_ONSET_GAP` to 100ms

---

## localStorage Keys
| Key | Purpose |
|-----|---------|
| `radha_jap_history` | `{ "YYYY-MM-DD": count }` — daily chant history |
| `radha_jap_targets` | `{ "YYYY-MM-DD": target }` — manually set per-date targets |

**Current data:** 4,00,000 counts seeded across Apr 18 – May 26, 2026 (39 days × ~10,256/day)

---

## Key State Variables
```javascript
let recognition       = null;   // SR instance (reused on desktop)
let isListening       = false;
let permPending       = false;  // true while Chrome's mic permission bar is showing
let permRetryInterval = null;   // independent setInterval retrying rec.start() during re-ask
let creditedPerIndex  = {};     // tracks counts already credited per result index
let srCycleCount      = 0;      // counts since last SR rebuild (mobile only)
let srDoReset         = false;  // signals onend to rebuild SR instance (mobile only)
let micStream         = null;   // kept alive (iOS: avoids re-asking permission on Stop/Start)
let dynHigh, dynLow   = ...;    // burst detection thresholds, set during calibration
let savedGrandTotal   = 0;      // cached total from localStorage, updated in saveToHistory()
let unsavedCount      = 0;      // counts since last flush to localStorage
let saveDebounce      = null;   // 3s debounce timer for normal-mode saves
```

---

## Maha Lakshya (1 Crore Grand Target)
- **Target:** 1,00,00,000 (1 crore)
- **Pace start date:** `PACE_START = '2026-04-18'`
- Shows: total done, percentage, thick progress bar, daily avg, estimated completion date + days remaining
- Updated live on every `addCount()` call via `updateGrandCard(savedGrandTotal + unsavedCount)`
- Uses earliest history key or PACE_START (whichever is earlier) for daily average calculation

---

## Past-Date Target Backfill System
- In history panel, each past date has a `<input>` target field (saved on Enter or blur)
- `saveTarget(dateKey, rawVal)` → stores in `radha_jap_targets`
- `getActiveBackfillDate()` → returns oldest unfulfilled past-date target
- `saveToHistory(n)` → routes counts to oldest unfulfilled target first, remainder to today
- Backfill banner (`#backfillBanner`) shows which date is being filled + progress
- When backfill active → save immediately; normal mode → debounce 3s

---

## Data Safety — Auto-Save + Restore System

### Problem
Chrome clears `localStorage` and `IndexedDB` when user re-logs into Chrome profile. Both the history data and the file handle are lost.

### Auto-Save (File System Access API)
- **💾 Set Auto-Save File** button at bottom of app
- User picks a file location once → Chrome shows save picker → file created
- `FileSystemFileHandle` stored in IndexedDB (`naam_jap_db` / `handles` store / key `auto_save_handle`)
- Saves every 30 seconds via `setInterval` + on `visibilitychange` (hidden) + on `beforeunload`
- File format: `{ history: {...}, targets: {...}, savedAt: "ISO string", version: 1 }`
- On startup: `initAutoSave()` checks IndexedDB for stored handle; if found + permission granted → resumes silently
- Button label changes to "💾 Saving to file" when active
- Status shows last save time: "✓ Saved at 10:32 PM"

### Restore from Backup
- **📂 Restore from Backup** button always visible (doesn't require empty localStorage)
- Also: full-screen restore modal (`#restoreModal`) shown automatically when localStorage is empty on load
- `pickAndRestoreFile()` → opens Chrome file picker (or falls back to `<input type=file>`)
- Smart merge: keeps whichever count is higher per date (protects new counts added since last backup)
- After restore: shows alert with total count + prompts to re-enable auto-save
- `startFresh()` button in modal: seeds 4,00,000 base data if user has no backup

### Restore Modal (shown when localStorage empty)
```html
<div class="restore-modal" id="restoreModal"> ... </div>
<input type="file" id="restoreFileInput" accept=".json" style="display:none">
```
Two options:
1. **📂 Pick Backup File & Restore** → loads data from file
2. **No backup — start fresh** → seeds 4,00,000 base data

---

## UI Features
- Purple/gold spiritual theme, राधा title
- **Two-column desktop layout:** counting app left | Maha Lakshya right (sticky)
- Big count display with gold flash on increment
- Target input + progress bar (session target)
- Session timer (only ticks while listening)
- Live transcript box + thin gold level bar (pulses with voice)
- Amber dot + "Click Allow" message + full-screen modal when Chrome permission bar appears
- Date-wise history stored in `localStorage`
- History panel with month accordion (preserves open/closed state on re-render)
- Past-date target inputs in history rows (Enter or blur to save)
- Backfill banner while filling a past-date target
- Reset button (resets session counter only, not history)
- Auto-save status row at bottom

---

## Known Issues / Status

### Desktop Chrome — Working well
- Counting is accurate and fast
- At ~2300 counts Chrome's internal session expires → full-screen modal appears → user clicks Allow once in browser bar → counting resumes automatically
- **Permanent fix:** Chrome address bar → lock icon → Microphone → Always Allow

### Mobile Safari iOS — Partially working
- Uses SR with 7-count rebuild (repetition blindness fix)
- Counting speed slower than desktop
- Not thoroughly re-tested after recent changes

### Mobile Chrome iOS (CriOS) — Problematic
- Uses burst detection (no SR available in CriOS)
- Auto-calibration runs on Start (1.5s silence required)
- User reports it "not working at all" — root cause not identified
- **Next step if revisiting:** Add RMS debug display temporarily to measure actual values vs dynHigh threshold

### Path B (future)
- User's end goal: paid app on App Store + Google Play
- Requires **React Native** with `SFSpeechRecognizer` (iOS) and Google Speech-to-Text (Android)
- Decision deferred until Path A is satisfactory

---

## Key Technical Decisions Made
1. **Desktop SR: same instance reuse** — rebuilding triggers Chrome session limit + permission re-ask; reuse delays it to ~2300 counts
2. **permPending + independent setInterval** — keeps app alive during Chrome permission re-ask; interval retries rec.start() every 800ms without relying on onend
3. **Mobile SR: 7-count rebuild** — resets engine memory before repetition blindness (same word stops being recognised at ~8–10 repeats)
4. **CriOS: burst detection instead of SR** — CriOS has no Web Speech API
5. **AudioContext in tap handler** — iOS/CriOS suspends AudioContext created in async callbacks
6. **Destination connection required** — iOS Web Audio graph doesn't run without a path to destination
7. **micStream kept alive** — releasing getUserMedia stream between Stop/Start triggers new permission dialog on iOS
8. **Auto-calibration on burst detection** — fixed hardcoded 0.005 threshold that was wrong for user's mic/room
9. **requestAnimationFrame over setInterval** — setInterval gets throttled by mobile browsers
10. **File System Access API for auto-save** — saves JSON to a user-chosen file on disk every 30s; handle stored in IndexedDB
11. **Smart merge on restore** — takes max(file count, localStorage count) per date; never loses new chants
12. **Restore modal on empty localStorage** — guards against Chrome clearing site data on profile re-login
13. **Two-column layout** — grand-card is now a proper flex sibling (sticky right column) not `position:fixed`, so no overlap with main content

---

## File Structure
```
/Users/kuldeepmac/App_Coding/Naam_Jap/
├── index.html              ← entire app (single file, ~1600 lines)
├── HANDOFF.md              ← this file
└── backup/
    └── index_backup_23-Apr-2026.html   ← backup before backfill feature
```

---

## How to Deploy
```bash
cd /Users/kuldeepmac/App_Coding/Naam_Jap
git add index.html
git commit -m "description"
git push
# GitHub Pages auto-deploys in ~30 seconds
```
