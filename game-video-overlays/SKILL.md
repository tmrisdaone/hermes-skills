---
name: game-video-overlays
description: Build accurate stat-tracking HUD overlays for gameplay footage (COD Mobile, PUBG, etc.) using frame-by-frame vision analysis, PIL-generated glassmorphism UI, and ffmpeg compositing. No transcript needed — game audio has no dialogue.
---

# Game Video Overlay Workflow

When a user provides gameplay footage and wants a custom HUD overlay with accurate kill/death/score tracking, use this pipeline. It's designed for scenarios where the standard "transcribe → cut → composite" pipeline doesn't apply because the video has no spoken dialogue.

## User Style Reference

This user's preferred editing style is demonstrated in: https://youtu.be/K2Ud7tZ6ekE (Hermes Agent as Video Editor by MM Editing). Key style elements:
- Glassmorphism HUD overlays (frosted glass cards, orange #FF5A00 accents, rounded corners)
- Frame-by-frame vision analysis for accurate stat tracking (kills, deaths, score, streak status)
- PIL-generated PNG sequence overlays for Android ARM64 reliability
- Event-based subtitles for kill feeds and medals (no speech transcription)
- Full-length video treatment (no trimming unless the content explicitly warrants it — talking/ranting content gets silence-based trim to remove dead air and menu screens)
- Professional branded callouts and motion graphics compositing
- **Don't propose strategy for confirmation — the user said "4 basically now start to do it". Build first, iterate after.**

Apply this production quality to all gaming video edits.

## Trigger
User provides a gameplay recording (especially mobile games like COD Mobile, PUBG, etc.) and asks for a stats overlay, glassmorphism HUD, or branded treatment on the full-length video without trimming.

## Pipeline

### 1. Frame Sampling for Stat Tracking

Since game videos have no transcript, extract frames every 3 seconds and analyze with vision:

```bash
for t in $(seq 0 3 $DURATION); do
  ffmpeg -ss $t -i source.mp4 -vframes 1 -q:v 2 frames/frame_$(printf "%03d" $t)s.jpg -y
done
```

Delegate frame analysis to a subagent (delegate_task) that uses vision_analyze on batches to build a kill timeline JSON with:
- kill count (from medals, scoreboard, kill feed)
- death events (killcam / respawn screens)
- team score (from top-center HUD)
- streak status (Bloodthirsty, Merciless, etc.)
- event labels for subtitles

Full brief template at `templates/frame-analysis-brief.md` — read and populate before spawning the subagent.

### 2. Build the Overlay (PIL + PNG Sequence)

Use PIL frame generation for anything over ~60 seconds. Hyperframes renders are too slow for long videos on Android ARM64.

**Directory structure:**
```
full_edit/
  generate_overlay.py    ← reads kill_timeline.json, renders frames with PIL
  overlay_frames/        ← frame_0000.png ... frame_NNNN.png
  overlay_render.mp4     ← compiled from PNG sequence
  subtitles.srt          ← game event captions
  final.mp4              ← composited output
```

**Pitfall: Text color contrast on glassmorphism**
White text (#FFFFFF) on a white frosted glass background is invisible. Always use contrasting colors:
- Kill count numbers → orange (#FF5A00)
- Score / gold → warm yellow (#FFC800)
- Labels → dim grey (180,180,180)
- Deaths → red-tinted (255,180,180)
- Glass background opacity → alpha 30-50 max (not opaque white)

Glassmorphism card template (PIL):
```python
draw.rounded_rectangle([x, y, x+w, y+h], radius=12, fill=(255,255,255,35))  # glass bg
draw.rounded_rectangle([x, y, x+w, y+3], radius=2, fill=accent_color)       # top accent
draw.rounded_rectangle([x, y, x+w, y+h], radius=12, outline=(255,255,255,70), width=1)  # border
```

### 3. Compile Overlay Video

```bash
ffmpeg -framerate 30 -i overlay_frames/frame_%04d.png \
  -c:v libx264 -preset medium -crf 18 \
  -y overlay_render.mp4
```

Note: render as plain yuv420p (no alpha). The chroma key green background will be removed at composite time.

### 4. Composite with Colorkey + Subtitles (Single Pass)

Apply colorkey at composite time — NOT in the overlay render. This avoids libx264 silently discarding the alpha channel.

```bash
ffmpeg -i source.mp4 -i overlay_render.mp4 \
  -filter_complex "[1:v]colorkey=0x00B140:0.15:0.05,format=yuva420p[ov];[0:v][ov]overlay=0:0:shortest=1[base];[base]subtitles=subtitles.srt:force_style='FontName=DejaVuSans-Bold,FontSize=16,PrimaryColour=\&H00FFFFFF,OutlineColour=\&H00000000,BorderStyle=1,Outline=2,Shadow=0,Alignment=2,MarginV=55'[outv]" \
  -map "[outv]" -map 0:a \
  -c:v libx264 -preset medium -crf 18 \
  -c:a aac -b:a 128k -movflags +faststart \
  -y final.mp4
```

Key flags:
- `colorkey` on `[1:v]` (overlay stream), never on the background
- `shortest=1` handles mismatched durations without pre-trimming
- `subtitles` is LAST in the filter chain (per Hard Rule 1 in video-use skill)
- Escape `&H` as `\&H` in shell commands

### 5. Verify

Extract frames at key moments and use vision_analyze to confirm:
1. Gameplay is visible behind the glass elements (not solid green)
2. Kill counters are readable with correct values
3. Status/streak indicators show the right state
4. Subtitles appear at the right timestamps and don't overlap overlays

## Game Mode Variants

### COD Mobile Multiplayer (Team Deathmatch / Domination)

- **Top-left:** KILLS (orange #FF5A00) + DEATHS
- **Top-right:** STATUS (streak level: COLD → NUCLEAR)
- **Bottom-left:** Map name + game mode + team score
- **Bottom-right:** Match timer (countdown)
- **Accent:** Orange
- **Subtitle events:** Medal names (First Blood, Double Kill, Bloodthirsty, Kingslayer), kill feed

### COD Mobile Zombies

- **Top-left:** ROUND (green #00CC50) + POINTS
- **Top-right:** ZOMBIES LEFT (red) + current WEAPON
- **Bottom-left:** Map name + game mode
- **Bottom-right:** Elapsed timer
- **Accent:** Green (survival/health theme)
- **Subtitle events:** Round announcements, zombie count milestones, death/continue prompts
- **No streak tracking** — zombies is survival-based, not PvP

### Generic FPS / Battle Royale

- Adapt the MP pattern with game-appropriate stats
- Avoid any COD-specific terminology (use "Players Left" instead of "Zombies Left")
- Subtitle style: kill feed logs instead of medal names

## Streak Status Levels (FPS Games)

| Kills | Streak | Color | Glow |
|-------|--------|-------|------|
| 0 | COLD | Blue/cyan | None |
| 1 | WARMING | Yellow | Slow pulse |
| 3 | HOT | Orange | Medium pulse |
| 5 | ON_FIRE | Red-orange | Strong pulse |
| 8 | BLOODTHIRSTY | Red | Intense + glow |
| After death | BROKEN | Grey | Dim, no effect |
| 25+ | MERCILESS | Purple | Intense glow |
| 30+ | NUCLEAR | Gold | Max glow |

## Subtitles for Game Events

Use SRT format with timestamps matching game events. For a 163s video:
```
1
00:00:24,000 --> 00:00:26,000
Match begins!

2
00:00:27,000 --> 00:00:29,000
FIRST BLOOD!

3
00:00:54,000 --> 00:00:57,000
Killed by PlayerName
```

Apply LAST in the ffmpeg filter chain. Subtitles must be after the overlay composite.

## Anti-patterns

- **White text on white glass** — kill count becomes invisible. Always use colored text on glass backgrounds.
- **Baking colorkey into overlay** — pre-applying colorkey and then re-encoding strips alpha. Apply colorkey at composite time.
- **Using Hyperframes for >60s overlays** — headless Chromium screenshot mode renders slowly. Use PIL frame generation instead.
- **Trying to transcribe game audio** — no dialogue to transcribe. Use frame vision instead.
