# AGENTS.md — sailing_race_debrief

## Overview
Single-file offline web app (`index.html`) that overlays two GPX sailing tracks, scrubs their shared timeline per-second, and calculates Velocity Made Good (VMG) to a windward mark.

- **Blue = Henry** — `14-Sep-2026-1911.gpx` (Open GPX Tracker, 3199 pts, dense 1–4s)
- **Orange = Hank** — `activity_24376755775.gpx` (Garmin Connect, 1523 pts, sparse 1–15s, HR extensions ignored)
- Map via **Leaflet 1.9.4** (CDN `unpkg.com`) + **Esri World Imagery** satellite tiles (`server.arcgisonline.com`). No layer control — Esri only.
- All track data is **embedded** as JSON arrays in `index.html` — no runtime GPX fetch; `file://` works. GPX files are build inputs only.

## Files
```
index.html                  — app (self-contained except Leaflet + Esri CDN)
14-Sep-2026-1911.gpx        — Henry track
activity_24376755775.gpx    — Hank track
AGENTS.md                   — this file
```

## Build / Regenerate
No build step committed. Embedded arrays can be regenerated:

```bash
python3 - << 'PY'
import xml.etree.ElementTree as ET, datetime, bisect, json, pathlib
def parse_gpx(path):
    pts=[]
    for e in ET.parse(path).iter():
        if e.tag.endswith('trkpt'):
            lat=float(e.attrib['lat']); lon=float(e.attrib['lon'])
            t=next((c.text for c in e if c.tag.endswith('time')), None)
            if t: pts.append((datetime.datetime.fromisoformat(t.replace('Z','+00:00')).timestamp(), lat, lon))
    pts.sort(); return pts
base="."
a=parse_gpx("14-Sep-2026-1911.gpx"); b=parse_gpx("activity_24376755775.gpx")
t0=max(a[0][0],b[0][0]); t1=min(a[-1][0],b[-1][0]); N=int(t1-t0)+1
def interp(pts,t):
    ts=[p[0] for p in pts]; i=bisect.bisect_left(ts,t)
    if i==0: return pts[0][1],pts[0][2]
    if i>=len(pts): return pts[-1][1],pts[-1][2]
    if ts[i]==t: return pts[i][1],pts[i][2]
    t0,la0,lo0=pts[i-1]; t1,la1,lo1=pts[i]; f=(t-t0)/(t1-t0)
    return la0+f*(la1-la0), lo0+f*(lo1-lo0)
secA=[interp(a,t0+k) for k in range(N)]; secB=[interp(b,t0+k) for k in range(N)]
fullA=[[lat,lon] for _,lat,lon in a]; fullB=[[lat,lon] for _,lat,lon in b]
# inject JSON dumps into index.html for const fullA/fullB/secA/secB and T0/N
PY
```

- Overlap: `t0 = max(startA,startB) = 2026-09-14T23:48:27Z`, `t1 = min(endA,endB) ≈ 2026-09-15T01:54:07Z`, `N=7541` seconds.
- `fullA/B` = raw track. `secA/B` = per-second linear interpolation (lat/lon) over overlap.
- Existing `index.html` already has `T0`, `N`, `fullA/B`, `secA/B` inlined — replace via regex `const NAME = [...];` if rebuilding.
- **Gotcha:** Do not blind `replaceAll("1911","Henry")` — numeric coords contain `1911` (e.g. `-122.7661911...`). Past bug produced `Uncaught SyntaxError at index.html:115:18251` from corrupting `fullA`. Label replacements must be HTML-only, never inside JS arrays.

## Running
- Open `index.html` directly (`file://`) or serve (`python3 -m http.server`). Requires internet for Leaflet JS/CSS + Esri tiles; track logic works offline.
- Hard refresh (`Ctrl+Shift+R`) after edits to bypass tile/JS cache.

## App Structure (`index.html`)
- `<style>` — `#map` (62vh), `#vmgBar` hero (grid: Henry | Δ | Hank), `#controls` slider row.
- `<div id="vmgBar">` — prominent VMG hero above map: large 42px VMG values, Δ, distance-to-mark + SOG meta, Set/Clear mark buttons, ★ coords badge.
- Map layers: `faintA/B` (opacity 0.28 full tracks), `trailA/B` (bright sailed prefix `sec*.slice(0,idx+1)`), `dotA/B` (`circleMarker`), `windwardMarker` (draggable divIcon ★), `lineA/B` (dashed to mark).
- Slider: `<input type=range min=0 max=7540 step=1>`, ±1/10s buttons, play/pause + speed select (0.25–120× simulated sec/real sec via `requestAnimationFrame`), Fit button.
- Keyboard: `ArrowLeft/Right` (±1, Shift ±10), `Space` play.

## VMG Logic
- Default windward mark: `[48.10815, -122.76681]` (`index.html: ~200`).
- `haversine(a,b)` (R=6371000) for distances.
- `vmgForTrack(sec, i, mark)`: `d = haversine(pos, mark)`, then `VMG = (dPrev - dNext)/dt` (+ = closing). Central diff `dt=2` (or 1 at ends). SOG similarly `haversine(prev,next)/dt`. `fmtVMG(vMs)` → `kn = v/0.514444` with sign. Δ = `VMG_Henry - VMG_Hank`. Render updates trails/dots/lines/VMG on every `render(idx)`.

## Conventions
- Keep HTML single-file, preserve self-contained arrays. Don't reintroduce runtime GPX fetch.
- Tile provider is Esri Satellite only — don't re-add CARTO/OSM/OpenTopoMap controls without request.
- Labels: **Henry** (blue, `fullA/secA`) vs **Hank** (orange, `fullB/secB`). Update title/legend/VMG cards/tooltips together.
