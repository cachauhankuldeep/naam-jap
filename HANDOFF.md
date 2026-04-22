# Naam Jap — Handoff Document
_Last updated: 2026-04-22_

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
  - `onend` keeps retrying `rec.start()` every 600ms via `permPending` flag
  - Once user clicks Allow → `onstart` fires → `permPending` cleared → counting resumes, dot goes green
  - User never needs to press the Start button again

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

## Key State Variables
```javascript
let recognition      = null;   // SR instance (reused on desktop)
let isListening      = false;
let permPending      = false;  // true while Chrome's permission bar is showing
let creditedPerIndex = {};     // tracks counts already credited per result index
let srCycleCount     = 0;      // counts since last SR rebuild (mobile only)
let srDoReset        = false;  // signals onend to rebuild SR instance (mobile only)
let micStream        = null;   // kept alive (iOS: avoids re-asking permission on Stop/Start)
let dynHigh, dynLow  = ...;    // burst detection thresholds, set during calibration
```

---

## UI Features
- Purple/gold spiritual theme, राधा title
- Big count display with gold flash on increment
- Target input + progress bar
- Session timer (only ticks while listening)
- Live transcript box + thin gold level bar (pulses with voice)
- Amber dot + "Click Allow" message when Chrome permission bar appears
- Date-wise history stored in `localStorage` (key: `radha_jap_history`)
- History panel with date/month view
- Reset button

---

## Known Issues / Status

### Desktop Chrome — Working well
- Counting is accurate and fast
- At ~2300 counts Chrome's internal session expires → amber dot + "Click Allow in browser bar ↑" → user clicks Allow once → counting resumes. No Start button needed.
- **Permanent fix:** In Chrome address bar → lock icon → Microphone → set to "Allow" (not "Ask"). This may allow Chrome to auto-approve and skip the bar entirely.

### Mobile Safari iOS — Partially working
- Uses SR with 7-count rebuild (repetition blindness fix)
- Counting speed slower than desktop (~10% of desktop speed per user report)
- Not thoroughly re-tested after recent changes

### Mobile Chrome iOS (CriOS) — Problematic
- Uses burst detection (no SR available in CriOS)
- Auto-calibration runs on Start (1.5s silence required)
- User reports it "not working at all" — root cause not identified
- Suspected issues: AudioContext state on CriOS, calibration threshold not adapting, or getUserMedia constraints not supported
- **Next step:** Add RMS debug display temporarily to measure what values are actually being seen vs dynHigh threshold

### Path B (future)
- User's end goal: paid app on App Store + Google Play
- Requires **React Native** with `SFSpeechRecognizer` (iOS) and Google Speech-to-Text (Android)
- Decision deferred until Path A is satisfactory

---

## Key Technical Decisions Made
1. **Desktop SR: same instance reuse** — rebuilding triggers Chrome session limit + permission re-ask; reuse delays it to ~2300 counts
2. **permPending flag** — keeps app alive during Chrome permission re-ask; user clicks Allow once, counting auto-resumes
3. **Mobile SR: 7-count rebuild** — resets engine memory before repetition blindness (same word stops being recognised at ~8–10 repeats)
4. **CriOS: burst detection instead of SR** — CriOS has no Web Speech API
5. **AudioContext in tap handler** — iOS/CriOS suspends AudioContext created in async callbacks
6. **Destination connection required** — iOS Web Audio graph doesn't run without a path to destination
7. **micStream kept alive** — releasing getUserMedia stream between Stop/Start triggers new permission dialog on iOS
8. **Auto-calibration on burst detection** — fixed hardcoded 0.005 threshold that was wrong for user's mic/room
9. **requestAnimationFrame over setInterval** — setInterval gets throttled by mobile browsers

---

## File Structure
```
/Users/kuldeepmac/App_Coding/Naam_Jap/
├── index.html      ← entire app (single file, ~900 lines)
└── HANDOFF.md      ← this file
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
