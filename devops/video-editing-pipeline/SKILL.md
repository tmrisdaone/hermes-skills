---
name: video-editing-pipeline
description: Modern video editing workflow using the video-use helpers (render.py, grade.py, timeline_view.py) and Hyperframes for glassmorphism motion graphics overlays on Android/Termux.
---

# Video Editing Pipeline

## Core Pipeline (in order)

1. **Inventory** — `ffprobe` every source to get codec, resolution, duration, FPS
2. **Transcribe** — `helpers/transcribe.py <video>` (needs ELEVENLABS_API_KEY)
3. **Pack transcripts** — `helpers/pack_transcripts.py --edit-dir <dir>` → `takes_packed.md`
4. **Silence detection** — `ffmpeg -i <video> -af "silencedetect=noise=-25dB:d=0.4" -f null -`
5. **Build EDL** — JSON with sources, ranges, grade, overlays, subtitles
6. **Extract segments** — `helpers/render.py <edl.json> -o final.mp4 --build-subtitles` (or use the helper functions programmatically)
7. **Composite** — Base → overlays (PTS-shifted) → subtitles LAST → loudnorm → final

## Critical Rules (from render.py)

### Overlay PTS shifting
`setpts=PTS-STARTPTS+{start_in_output}/TB` — shifts overlay frame 0 to its window start.
Compositing: `overlay=enable='between(t,{start},{end})'` — only shows overlay during its window.

### Subtitles LAST (Rule 1)
Always apply subtitles filter AFTER all visual overlays, otherwise overlays hide captions.
Force style for vertical video (TikTok/Shorts/Reels):
```
FontName=Helvetica,FontSize=18,Bold=1,
PrimaryColour=&H00FFFFFF,OutlineColour=&H00000000,BackColour=&H00000000,
BorderStyle=1,Outline=2,Shadow=0,Alignment=2,MarginV=90
```
MarginV=90 clears the bottom ~30% UI zone on all vertical platforms.

### Per-segment extract → lossless concat (Rule 2)
NOT single-pass filtergraph. Extract each segment with grade + 30ms audio fades:
```
ffmpeg -ss {start} -i {source} -t {duration} -vf "{grade}" -af "afade=t=in:st=0:d=0.03,afade=t=out:st={dur-0.03}:d=0.03" -c:v libx264 -preset fast -crf 18 -c:a aac -b:a 128k {out}
```
Then concat with `-f concat -safe 0 -i {list} -c copy {base}`

### 30ms audio fades (Rule 3)
Every segment boundary gets `afade=t=in:st=0:d=0.03,afade=t=out:st={dur-0.03}:d=0.03` — prevents audible pops.

### Never cut inside a word (Rule 6)
Snap every cut edge to a word boundary from the transcript.

### Pad every cut edge (Rule 7)
Working window: 30-200ms. Tighter for fast-paced, looser for cinematic.

### Loudnorm to -14 LUFS (social standard)
Two-pass loudnorm: I=-14 LUFS, TP=-1 dBTP, LRA=11 LU. Matches YouTube/Instagram/TikTok/X/LinkedIn.

## Color Grading (grade.py)

4 presets available:
- `subtle`: `eq=contrast=1.03:saturation=0.98` — barely perceptible cleanup
- `neutral_punch`: contrast 1.06 + gentle S-curve — minimal corrective
- `warm_cinematic`: contrast 1.12, crushed blacks, -12% sat, warm shadows + cool highs (creative)
- `none`: no grade — straight copy

Auto-grade system (`auto_grade_for_clip()`):
- Samples N frames via ffmpeg signalstats
- Measures Y mean, range, saturation
- Applies bounded corrections (±8% max) for underexposure, flatness, desaturation
- Returns filter string + stats dict

Usage: `python helpers/grade.py <input> -o <output>` (auto mode)
Or: `python helpers/grade.py <input> -o <output> --preset warm_cinematic`

## Hyperframes Overlays (Android/Termux)

### When to use Hyperframes vs PIL
- **Hyperframes** — Browser-native HTML/CSS/GSAP video compositions. Best for complex UI motion, kinetic typography, product UI mockups, WebM alpha overlays. Requires proot-distro Ubuntu with nodejs + Termux-native chromium.
- **PIL + PNG sequence** — Simple overlay cards: counters, bars, glassmorphism elements. More reliable on Android ARM64 since browser rendering can be slow/unstable in proot.

### Hyperframes Render Command
```bash
proot-distro login ubuntu --termux-home -- bash -c "
  export HYPERFRAMES_BROWSER_PATH=/data/data/com.termux/files/usr/bin/chromium-browser
  cd /path/to/project
  npx --yes hyperframes render . -o render.mp4
"
```

### Key Hyperframes Rules
1. Every timed element needs `data-start`, `data-duration`, `data-track-index`
2. Visible timed elements must have `class="clip"`
3. GSAP timelines paused + registered on `window.__timelines`:
   ```js
   window.__timelines = window.__timelines || {};
   window.__timelines["id"] = gsap.timeline({ paused: true });
   ```
4. No `Date.now()`, `Math.random()`, or network fetches
5. Lint after changes: `npm run check`
6. `npx hyperframes docs <topic>` for reference docs in terminal

### PIL Fallback (for stat overlays on Android)
For game stat overlays (kills, rounds, zombies, timer), PIL generates frames faster:
1. Write `generate_overlay.py` with frame-by-frame drawing
2. Generate all PNG frames
3. Compile: `ffmpeg -framerate 30 -i frames/frame_%04d.png -c:v libx264 -preset medium -crf 18 -vf "colorkey=0x00B140:0.15:0.05,format=yuva420p" -pix_fmt yuva420p -y overlay.mp4`
4. Composite: colorkey applied at render time:
   ```
   -filter_complex "[1:v]colorkey=0x00B140:0.15:0.05,format=yuva420p[overlay];[0:v][overlay]overlay=0:0:shortest=1[outv]"
   ```

### Alpha Channel Issue on Android
When re-encoding with libx264, `-pix_fmt yuva420p` may silently become `yuv420p`. Fix: apply colorkey during compositing rather than during overlay encoding. The overlay can be opaque green; the colorkey filter strips it during the final composite step.

## Modern Animation Techniques

### Glassmorphism Aesthetic
- Semi-transparent backgrounds: rgba(255,255,255,0.12-0.18)
- 1px borders at 30% opacity
- 12px border-radius on all cards
- Subtle drop shadows (2px offset, 4px blur, 20% opacity black)
- Accent lines (2-3px thin strip at top of each card)
- Orange (#FF5A00) for action/highlight, green for zombies, blue for cold status

### Game Stat Overlays (Accurate Data)
1. Extract frames every 3s across the full video
2. Use vision_analyze to read actual game HUD stats
3. Build a timeline of kill/points/round events
4. Interpolate between known data points
5. Render overlay with interpolated values at 30fps

### Status/Streak Animation
- COLD (blue) → WARMING (yellow pulse) → HOT (orange pulse) → ON_FIRE (red pulse) → BLOODTHIRSTY (red glow) → MERCILESS (purple glow) → NUCLEAR (gold)
- Pulse formula: `abs(sin(t * frequency)) * 0.5 + 0.5` mapped to glow alpha
- Status changes at event boundaries

## Timing Helper Functions

```python
def get_state(t, timeline):
    """Interpolate stats at time t from a list of (timestamp, value) events."""
    kills, deaths = 0, 0
    for ts, k, d in timeline:
        if t >= ts:
            kills, deaths = k, d
    return kills, deaths

def ease_out_cubic(t):
    return 1 - (1 - t) ** 3

def ease_in_out_cubic(t):
    return 4 * t ** 3 if t < 0.5 else 1 - (-2 * t + 2) ** 3 / 2
```

## Reference
- MM Editing video style: https://youtu.be/K2Ud7tZ6ekE
- Hyperframes docs: https://hyperframes.heygen.com/introduction
- Video-use pipeline helpers: `~/.hermes/skills/video-use/helpers/`
