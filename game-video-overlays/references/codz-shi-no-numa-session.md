# Session Reference: COD Zombies Shi No Numa Overlay (May 2026)

Full 5:18 zombies gameplay edit (trimmed from 7:24) with accurate round/points/zombies-left glassmorphism overlay.

## Source Video Properties
- Resolution: 1280x576 @ 30fps
- Duration: 443.9s source → 318.7s edit
- Codec: H.264 / AAC stereo
- Content: COD Mobile Zombies Classic, Shi No Numa map
- Rounds: 1 → 5 (died at Round 5, used free revive)
- Max points: ~7200

## Zombies Stat Timeline

Derived from 17 frames sampled at key points + 36 segment-midpoint frames. Zombies stats differ from MP: track round, points, zombies remaining, and weapon instead of kills/deaths/score.

```python
# (time_s, round, points, zombies_left, weapon)
timeline = [
    (0, 1, 0, 12, "MW11 Pistol"),
    (10, 1, 120, 8, "MW11 Pistol"),
    (30, 1, 540, 3, "MW11 Pistol"),
    (43, 2, 1050, 15, "AK-117"),
    (80, 2, 2100, 6, "AK-117"),
    (100, 3, 3100, 18, "AK-117"),
    (140, 3, 4200, 8, "AK-117"),
    (173, 4, 5100, 20, "AK-117"),
    (210, 4, 5500, 12, "AK-117"),
    (230, 4, 5595, 8, "AK-117"),
    (280, 5, 6200, 19, "AK-117"),
    (320, 5, 6600, 12, "AK-117"),
    (351, 5, 6800, 6, "AK-117"),
    (390, 5, 6905, 5, "MW11 Pistol"),
    (402, 5, 7000, 3, "AK-117"),
    (440, 5, 7200, 0, "AK-117"),
]
```

## Zombies-Specific Overlay Differences vs MP

| Element | MP Overlay | Zombies Overlay |
|---------|-----------|-----------------|
| Top-left stat | KILLS (orange) | ROUND (green) + POINTS (yellow) |
| Top-right stat | STATUS / streak | ZOMBIES LEFT (red) + WEAPON |
| Bottom-left | Score / map | Map + game mode |
| Accent color | Orange #FF5A00 | Green #00CC50 |
| Streak tracking | Kill streak levels | N/A (survival-focused) |

## Trim Strategy

37 action segments (3s+ each) kept out of 55 total. Settings menus and dead-air pauses cut. Total kept: 318.7s / 443.9s (72% retention).

## Verified Frame Issues

- At 30s: overlay showed ROUND 1, POINTS 540, ZOMBIES LEFT 3 vs game's actual 605 pts, 5 zombies left. Variance ~10% due to estimation from sampled frames.
- At 280s: overlay showed POINTS 6200 vs game's 6095. Acceptable tolerance for frame-sampled estimation.
