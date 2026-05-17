# Chroma Key Compositing Reference

## Problem
Hyperframes overlay rendered with solid green background (#00B140) needs to be composited over gameplay footage via ffmpeg colorkey.

## Correct Command (single-pass)

```bash
ffmpeg -i background_video.mp4 -i hyperframes_overlay.mp4 \
  -filter_complex "[1:v]colorkey=0x00B140:0.15:0.05,format=yuva420p[ov];[0:v][ov]overlay=0:0:shortest=1[outv]" \
  -map "[outv]" -map 0:a \
  -c:v libx264 -preset medium -crf 18 \
  -c:a aac -b:a 128k \
  -movflags +faststart \
  -y final_edit.mp4
```

## What NOT to do

### Don't pre-apply colorkey then re-encode separately
```bash
# WRONG — alpha is silently lost on re-encode:
ffmpeg -i overlay.mp4 -t 46 -pix_fmt yuva420p -y trimmed_overlay.mp4  # -> yuv420p!
ffmpeg -i base.mp4 -i trimmed_overlay.mp4 -filter_complex "[0:v][1:v]overlay=0:0" out.mp4  # opaque!
```

Even with `-pix_fmt yuva420p`, the trimmed overlay ends up as yuv420p, making it fully opaque over the background.

### Don't separate colorkey and overlay into two passes
```bash
# WRONG — two separate encodings:
ffmpeg -i overlay.mp4 -vf "colorkey=0x00B140:0.15:0.05,format=yuva420p" -y keyed.mp4
ffmpeg -i base.mp4 -i keyed.mp4 -filter_complex "[0:v][1:v]overlay=0:0" out.mp4
```

This double-encodes and degrades quality.

## Verify After Compositing

```bash
ffmpeg -ss 10 -i final_edit.mp4 -vframes 1 -q:v 2 verify_frame.jpg
```

Check: gameplay visible behind glass elements? Glass UI legible? No green spill?

## Key Parameters

| Parameter | Value | Reason |
|-----------|-------|--------|
| colorkey color | `0x00B140` | Hyperframes chroma key green |
| similarity | `0.15` | Tight tolerance for uniform green |
| blend | `0.05` | Minimal edge softening |
| shortest | `1` | Ends when shorter input ends |

## Chrome Key Green Hex Values
- Hyperframes default: `#00B140` (RGB: 0, 177, 64)
- Classic chroma green: `#00FF00` (RGB: 0, 255, 0)
- Match to whatever your Hyperframes composition uses as background-color
