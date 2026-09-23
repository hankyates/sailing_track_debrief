# AGENTS.md — sailing_race_debrief

## Overview
Single-file offline web app (`index.html`) that overlays two GPX sailing tracks, scrubs their shared timeline per-second, calculates Velocity Made Good (VMG) to a windward mark, and can trim the analysis window (cut start/end). Sessions are selectable by day from a dropdown.

- **Sessions** (one dropdown entry per day):
  - **Mon 14 Sep 2026** — Henry `henry-2026-09-14-234229.gpx` (Open GPX Tracker, 3199 pts, dense 1–4s) vs Hank `hank-2026-09-14-234827.gpx` (Garmin Connect, 1523 pts, sparse 1–15s, HR extensions ignored). Overlap `N=7541`.
  - **Mon 21 Sep 2026** — Henry `henry-2026-09-21-233222.gpx` (3502 pts) vs Hank `hank-2026-09-21-234108.gpx` (1739 pts). Overlap `N=6379`.
- `activity_*` files are **Hank**; other uploaded GPX are **Henry**. Blue = Henry, Orange = Hank.
- Map via **Leaflet 1.9.4** (CDN `unpkg.com`) + **Esri World Imagery** satellite tiles (`server.arcgisonline.com`). No layer control — Esri only.
- All track data is **embedded** as JSON arrays in `index.html` — no runtime GPX fetch; `file://` works. GPX files are build inputs only.

## Files
```
index.html                  — app (self-contained except Leaflet + Esri CDN)
activities/
  2026-09-14/
    henry-2026-09-14-234229.gpx   — Henry (14 Sep)
    hank-2026-09-14-234827.gpx    — Hank  (14 Sep)
  2026-09-21/
    henry-2026-09-21-233222.gpx   — Henry (21 Sep)
    hank-2026-09-21-234108.gpx    — Hank  (21 Sep)
AGENTS.md                   — this file
```
Filename scheme: `<sailor>-<YYYY-MM-DD>-<HHMMSS>.gpx` where `HHMMSS` is the UTC activity start (first `trkpt` time). Directory per day: `activities/<YYYY-MM-DD>/`.

## Build / Regenerate
No build step committed. Embedded per-session arrays can be regenerated — build a `const SESSIONS = {...};` object keyed by day (`T0`, `N`, `fullA/B`, `secA/B`, `label`, `henryFile`, `hankFile`), then in `index.html` replace the span from `const SESSIONS = ` up to (and including) the line ending `};` directly before `let T0 = 0, N = 0, ...`:

```python
import xml.etree.ElementTree as ET, datetime, bisect, json
def parse_gpx(path):
    pts=[]
    for e in ET.parse(path).iter():
        if e.tag.endswith('trkpt'):
            lat=float(e.attrib['lat']); lon=float(e.attrib['lon'])
            t=next((c.text for c in e if c.tag.endswith('time')), None)
            if t: pts.append((datetime.datetime.fromisoformat(t.replace('Z','+00:00')).timestamp(), lat, lon))
    pts.sort(); return pts
def interp(pts,t):
    ts=[p[0] for p in pts]; i=bisect.bisect_left(ts,t)
    if i==0: return pts[0][1],pts[0][2]
    if i>=len(pts): return pts[-1][1],pts[-1][2]
    if ts[i]==t: return pts[i][1],pts[i][2]
    t0,la0,lo0=pts[i-1]; t1,la1,lo1=pts[i]; f=(t-t0)/(t1-t0)
    return la0+f*(la1-la0), lo0+f*(lo1-lo0)
sessions = [  # key, label, henry path, hank path
    ("2026-09-14","Mon 14 Sep 2026","activities/2026-09-14/henry-2026-09-14-234229.gpx","activities/2026-09-14/hank-2026-09-14-234827.gpx"),
    ("2026-09-21","Mon 21 Sep 2026","activities/2026-09-21/henry-2026-09-21-233222.gpx","activities/2026-09-21/hank-2026-09-21-234108.gpx"),
]
out={}
for key,label,hp,hk in sessions:
    a=parse_gpx(hp); b=parse_gpx(hk)
    t0=max(a[0][0],b[0][0]); t1=min(a[-1][0],b[-1][0]); N=int(t1-t0)+1
    secA=[interp(a,t0+k) for k in range(N)]; secB=[interp(b,t0+k) for k in range(N)]
    out[key]=dict(label=label, henryFile=hp, hankFile=hk, T0=t0, N=N,
                  fullA=[[la,lo] for _,la,lo in a], fullB=[[la,lo] for _,la,lo in b],
                  secA=secA, secB=secB)
block="const SESSIONS = "+json.dumps(out)+";\n"
text=open("index.html").read()
s=text.index("const SESSIONS = ")
e=text.index("];\nlet T0 = ", s)+3   # end of SESSIONS ]; line
open("index.html","w").write(text[:s]+block+text[e:])
```

- Session order in the dropdown/object = dropdown order. Add a new day by appending to `sessions`, adding the files under `activities/<YYYY-MM-DD>/`, and regenerating.
- `fullA/B` = raw track. `secA/B` = per-second linear interpolation (lat/lon) over the session overlap.
- **Gotcha:** Do not blind `replaceAll("1911","Henry")` — numeric coords contain `1911` (e.g. `-122.7661911...`). Past bug produced `Uncaught SyntaxError` from corrupting `fullA`. Label replacements must be HTML-only, never inside JS arrays.

## Running
- Open `index.html` directly (`file://`) or serve (`python3 -m http.server`). Requires internet for Leaflet JS/CSS + Esri tiles; track logic works offline.
- Hard refresh (`Ctrl+Shift+R`) after edits to bypass tile/JS cache.

## App Structure (`index.html`)
- `<style>` — `#map` (62vh), `#vmgBar` hero (grid: Henry | Δ | Hank), `#controls` rows (session + slider, trim, stats).
- `<div id="vmgBar">` — prominent hero above the map. Each boat card is a fixed-height 4-column grid of metrics (`.vmg-col`: value above, label directly underneath): **VMG**, **SOG**, **Dist to mark**, **Heading−mark** (bearing of travel vs bearing to mark, 0–180° via `bearing()`/`headingAt()`/`angleDiffToMark()`). Center card = **Δ (Henry − Hank)**: `vmgDelta` (Δ VMG, kn) and `closerDelta` (Δ dist), both signed — `+` blue favors Henry, `−` orange favors Hank (white when even); Set/Clear mark buttons, ★ coords badge.
- `const SESSIONS = {...}` — one entry per day (`T0`, `N`, `fullA/B`, `secA/B`, `files`). Active arrays are held in `let T0/N/fullA/B/secA/B`; switching the dropdown runs `loadSession(key)`.
- Map layers: `faintA/B` (full tracks, opacity 0.28 — shown by default, hidden via the "Hide full track" toggle), `trailA/B` (solid 10 s lag tail: `sec*.slice(max(trimStart,idx-10), idx+1)`), `leadA/B` (transparent 10 s lead ahead of cursor: `sec*.slice(idx, min(trimEnd-1,idx+10)+1)`), `dotA/B` (`circleMarker` + permanent SOG badge above each, `.sog-badge` tooltip via `setTooltipContent`), `windwardMarker` (draggable divIcon ★), `lineA/B` (dashed to mark), `mkAS/mkAE/mkBS/mkBE` (start/end markers, repositioned to trim window bounds when trimmed).
- Controls (row 2, `#trim-row`): **Cut start** / **Cut end** set the trim window at the current frame; **Reset trim** clears it; **Show full track** toggles the faint full-track override (on by default). Trim window `[trimStart, trimEnd)` clamps slider min/max, trails, dots, markers, and elapsed (`T+` is window-relative).
- **localStorage** (`srd:trims`, `srd:windward`, `srd:showFull`, `srd:speed`): trim window (stored per-day keyed by session id, clamped on restore), windward mark placement (shared across sessions), the show-full-track toggle, and the playback speed are auto-persisted via `savePrefs()` (called from trim buttons, show-full toggle, mark set/clear/dragend, speed change) and restored by `loadPrefs()` before the initial `loadSession`.
- Slider: `min=trimStart max=trimEnd-1 step=1`, ±1/10 s buttons, play/pause + speed select (1× / 2× / 5× simulated sec/real sec via `requestAnimationFrame`, choice persisted as `srd:speed`; playback stops at window end), Fit button.
- Keyboard: `ArrowLeft/Right` (±1, Shift ±10), `Space` play.

## VMG Logic
- Default windward mark: `[48.10815, -122.76681]` (`index.html: ~200`). Single mark shared across sessions.
- `haversine(a,b)` (R=6371000) for distances.
- `vmgForTrack(sec, i, mark)`: `d = haversine(pos, mark)`, then `VMG = (dPrev - dNext)/dt` (+ = closing). Central diff `dt=2` (or 1 at ends). SOG similarly `haversine(prev,next)/dt`. `fmtVMG(vMs)` → `kn = v/0.514444` with sign. Δ = `VMG_Henry - VMG_Hank`. VMG uses the full per-second arrays, so values at the trim window edges are real (adjacent points still exist). Render updates trails/dots/lines/VMG/markers on every `render(idx)`.

## Conventions
- Keep HTML single-file, preserve self-contained arrays. Don't reintroduce runtime GPX fetch.
- Tile provider is Esri Satellite only — don't re-add CARTO/OSM/OpenTopoMap controls without request.
- Labels: **Henry** (blue, `fullA/secA`) vs **Hank** (orange, `fullB/secB`). Update title/legend/VMG cards/tooltips together.