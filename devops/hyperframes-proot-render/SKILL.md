---
name: hyperframes-proot-render
description: Render Hyperframes compositions on Android ARM64 using a Proot-Ubuntu environment to bypass native binary incompatibilities.
---

# Hyperframes Proot-Ubuntu Rendering

Rendering Hyperframes animations on Android (ARM64) requires a Linux environment because `onnxruntime` and the required Headless Chromium binaries are typically not available or compatible with native Termux. This skill provides the workflow for setting up and executing renders inside `proot-distro`.

## Prerequisites
- `proot-distro` installed in Termux.
- An installed Ubuntu distro (`proot-distro install ubuntu`).
- Termux-native `chromium` installed (`pkg install chromium`) to be used as the browser bridge.

## Workflow

### 1. Setup the Environment
Ensure the Ubuntu container is updated and has the basic Node.js runtime.

```bash
proot-distro login ubuntu -- bash -c "apt update && apt install -y nodejs npm"
```

### 2. Configure the Browser Path
Hyperframes requires a functional Chromium binary. Because Ubuntu's `apt install chromium-browser` is often a snap-wrapper (which fails in Proot), you must point Hyperframes to the **Termux-native** chromium binary, which Proot can access via the shared home directory.

**The Magic Path:**
`/data/data/com.termux/files/usr/bin/chromium-browser`

### 3. Execution Command
Use the following command structure to render. You must `export` the browser path and `cd` into your project directory within the login shell.

```bash
proot-distro login ubuntu --termux-home -- bash -c "export HYPERFRAMES_BROWSER_PATH=/data/data/com.termux/files/usr/bin/chromium-browser && cd /absolute/path/to/project && npx hyperframes render"
```

## Critical Implementation Details

### Avoid "Screenshot-Only" Rendering
**Pitfall:** By default, Hyperframes captures screens. If you use a `<video>` tag as a background, the renderer may only capture the first frame or fail to decode the video, resulting in a static image.
**Solution:** For true video editing (cuts, movements, and rhythmic edits), do NOT rely on Hyperframes to play the video. Instead:
1. Use **FFmpeg** to handle the video stream, trimming, and rhythmic clipping.
2. Render the Hyperframes glass elements as a separate transparent overlay (alpha channel) or as a mask.
3. Composite the two using FFmpeg's `overlay` filter:
   `ffmpeg -i background_video.mp4 -i glass_overlay.mp4 -filter_complex \"[0:v][1:v]overlay=0:0\" output.mp4`

### HTML/JS Requirements for Deterministic Rendering
(Keep existing window.__hf documentation)
When using local video files as backgrounds in Hyperframes within Proot:
- **Path Resolution:** Headless Chromium in Proot cannot access `/sdcard/` paths directly via its internal FileServer. 
- **The Workaround:** Copy the source video file from `/sdcard/Download/` into the project directory and reference it as a relative path in the HTML.
- **Playback Control:** Since Hyperframes needs deterministic control, avoid using `video.currentTime` manually inside scripts. Instead, use the `seek` function in `window.__hf` to synchronize the video state with the render clock.

### HTML/JS Requirements for Deterministic Rendering
To avoid `window.__hf not ready` errors, your `index.html` must expose the Hyperframes API for frame-accurate seeking:

```javascript
window.__hf = {
    duration: 5, // total duration in seconds
    seek: (time) => {
        // 1. Update GSAP timeline
        gsap.globalTimeline.seek(time);
        // 2. Manually update any custom effects (like typing)
        updateTypingEffect(time);
    }
};
```

### Compositing the Overlay with Chroma Key (Correct Approach)

When the Hyperframes overlay was rendered with a solid chroma key green (#00B140) background, compositing requires applying the colorkey filter in the same filtergraph as the overlay — **not as a separate pre-processing step**.

**Pitfall: Alpha channel silently lost on re-encode**
If you pre-render the overlay with colorkey (`ffmpeg ... -vf "colorkey=0x00B140:0.15:0.05,format=yuva420p"`) and then trim or re-encode it separately (`ffmpeg -i overlay.mp4 -t 46 -pix_fmt yuva420p ...`), the alpha channel gets silently converted to yuv420p. The result is an opaque overlay that completely covers the background.

**Fix: Apply colorkey + overlay in one filtergraph**

```bash
# Correct: colorkey + overlay + duration matching in one pass
ffmpeg -i background_video.mp4 -i hyperframes_overlay.mp4 \
  -filter_complex "[1:v]colorkey=0x00B140:0.15:0.05,format=yuva420p[ov];[0:v][ov]overlay=0:0:shortest=1[outv]" \
  -map "[outv]" -map 0:a \
  -c:v libx264 -preset medium -crf 18 \
  -c:a aac -b:a 128k \
  -movflags +faststart \
  -y final_edit.mp4
```

Key flags:
- `shortest=1` — stops when the shorter input ends (handles mismatched durations without pre-trimming)
- `colorkey=0x00B140:0.15:0.05` — 0.15 similarity, 0.05 blend (tight tolerance for a uniform green)
- `colorkey` goes on the overlay stream `[1:v]`, never on the background

**Verify after compositing**
Extract a frame and check visually that game footage is visible behind the glass elements:

```bash
ffmpeg -ss 10 -i final_edit.mp4 -vframes 1 -q:v 2 verify_frame.jpg
# Open verify_frame.jpg and check: gameplay visible? glass UI legible?
```

### Hybrid Rhythmic Editing Workflow
When a user asks for a video that "actually plays" or "cuts to the beat," do not use Hyperframes as the primary video engine. Use it as an overlay layer.

**The "Rhythmic Cut" Pipeline:**
1. **Audio Mapping**: Generate a beat-map (e.g., `audio_reactive_map.json`) using energy-detection scripts.
2. **Programmatic Slicing**: 
   - Loop through the map and use `ffmpeg -ss [start] -t [duration] -i [source] -c copy [clip_name].mp4`.
   - Use a `concat.txt` file to assemble these clips into a `rhythmic_base.mp4`.
3. **Layered Animation**:
   - Create a Hyperframes project with a Chroma Key Green (`#00B140`) background.
   - Sync GSAP animations (glass cards, status bars) to the same beat-map timestamps.
   - Render this as a separate `overlay.mp4`.
4. **Final Compositing** (see "Compositing the Overlay with Chroma Key" above for the correct single-pass command):
   - `ffmpeg -i rhythmic_base.mp4 -i overlay.mp4 -filter_complex "[1:v]colorkey=0x00B140:0.15:0.05,format=yuva420p[ov];[0:v][ov]overlay=0:0:shortest=1[outv]" -map "[outv]" -map 0:a -c:v libx264 -preset medium -crf 18 -c:a aac -b:a 128k -movflags +faststart -y final_edit.mp4`

**Pitfall: "Screenshot Mode"**
Hyperframes natively captures browser screenshots. If you put a `<video>` tag inside Hyperframes and render, it may only capture a single frame or a static image. Always move the "video" part of the edit to FFmpeg and keep Hyperframes for the "UI/Motion Graphics" layer.
- **GPU:** Note that WebGL acceleration is often unavailable in Proot; the render will fall back to "screenshot mode," which is slower but accurate.

## Reference Files

- `references/compositing.md` — exact ffmpeg commands for overlaying the green-screen Hyperframes output onto gameplay footage, including the critical pitfall about alpha-channel loss during re-encoding.

## Troubleshooting
- **`command requires the chromium snap`**: You are using `/usr/bin/chromium-browser` (Ubuntu) instead of the Termux binary. Update `HYPERFRAMES_BROWSER_PATH`.
- **`window.__hf not ready`**: Your JS is not exposing the `seek` function or is throwing an error before it reaches the `window.__hf` assignment.
- **Slow Renders**: Reduce worker count or use a simpler backdrop to avoid Chrome compositor starvation.
- **Alpha channel silently lost on re-encode**: When re-encoding a yuva420p overlay with libx264, the alpha channel may silently become yuv420p. Always apply colorkey during the FINAL compositing step instead of during overlay encoding. Keep the overlay as opaque green; strip it at composite time: `-filter_complex "[1:v]colorkey=0x00B140:0.15:0.05,format=yuva420p[overlay];[0:v][overlay]overlay=0:0:shortest=1[outv]"`
