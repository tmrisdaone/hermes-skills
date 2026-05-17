# Session Reference: COD Mobile Shipment Overlay (May 2026)

Full 163.5s COD Mobile Team Deathmatch on Shipment with accurate glassmorphism HUD overlay.

## Source Video Properties
- Resolution: 1280x576 @ 30fps
- Duration: 163.5s
- Codec: H.264 / AAC stereo
- Content: COD Mobile Shipment, Team Deathmatch mode
- Player: TMRIST
- Final score: Blue 40 - Red 15, 2 deaths

## Kill Timeline Data

The kill timeline was derived from 55 frames sampled every 3 seconds, analyzed with vision_analyze via delegate_task. Key events:

```
0s-23s:   Pre-match (loading, countdown, 0 kills)
24s:      Match starts
27s:      First Blood (1 kill)
30s:      Headshot + Double Kill (3 kills, score 4-0)
36s:      Berserker (4 kills, score 6-0)
45s:      Bloodthirsty — "TMRIST On a 5 killstreak!" (8 kills, score 10-6)
54s:      DEATH 1 — killed by Coyoteaucitron (score 12-10)
72s:      Triple Kill + Berserker + Avenger (15 kills, score 21-5)
81s:      Double Kill — "TMRIST On a 5 killstreak!" (17 kills, score 24-10)
90s:      DEATH 2 — killed by NotLogic (score 26-10)
108s:     Berserker (22 kills, score 32-11)
120s:     Headshot + Double Kill (25 kills, score 36-13)
132s:     Kingslayer + Savior + Avenger — "TMRIST On a 5 killstreak!" (33 kills, score 39-15)
135s:     VICTORY — Blue 40 - Red 15 (40 kills)
135-163s: Post-match (MVP screen, replay)
```

## PIL Frame Generation

The overlay generator script reads this timeline data and renders each frame as a 1280x576 RGBA PNG with chroma key green background (#00B140). Key implementation details:

- Font: DejaVuSans-Bold (Termux path: `/data/data/com.termux/files/usr/share/fonts/TTF/DejaVuSans-Bold.ttf`)
- Interpolation: linear between known kill values (kills = int(current + (next - current) * progress))
- 4905 total frames at 30fps for 163.5s
- Generation time: ~3 minutes on Android ARM64

## Streak Status Logic

```python
streak_phases = [
    (0, "COLD"), (24, "WARMING"), (27, "WARMING"), (30, "HOT"),
    (36, "ON_FIRE"), (45, "BLOODTHIRSTY"), (54, "BROKEN"),
    (63, "WARMING"), (69, "HOT"), (72, "ON_FIRE"), (81, "BLOODTHIRSTY"),
    (90, "BROKEN"), (102, "WARMING"), (108, "ON_FIRE"),
    (120, "BLOODTHIRSTY"), (129, "MERCILESS"), (135, "NUCLEAR"),
]
```

## Compositing Command

```bash
ffmpeg -i snaptik_7638108059109231892_v3.mp4 \
  -i full_edit/overlay_render2.mp4 \
  -filter_complex \
    "[1:v]colorkey=0x00B140:0.15:0.05,format=yuva420p[ov];\
     [0:v][ov]overlay=0:0:shortest=1[base];\
     [base]subtitles=full_edit/subtitles.srt:\
       force_style='FontName=DejaVuSans-Bold,FontSize=16,\
         PrimaryColour=\&H00FFFFFF,OutlineColour=\&H00000000,\
         BorderStyle=1,Outline=2,Shadow=0,Alignment=2,MarginV=55'[outv]" \
  -map "[outv]" -map 0:a \
  -c:v libx264 -preset medium -crf 18 \
  -c:a aac -b:a 128k \
  -movflags +faststart \
  -y full_edit/final_v2.mp4
```

Output: 74MB, 163.68s, H.264 yuv420p
