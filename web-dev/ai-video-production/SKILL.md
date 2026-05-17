---
name: ai-video-production
description: Framework for building AI-driven video generation pipelines incorporating text-to-speech, automated visual sourcing, and cinematic assembly.
triggers: ["video generator", "text to video", "automatic video creation", "ai movie", "cinematic assembly"]
---

# AI Video Production Pipeline

## Overview
This skill governs the creation of automated "Text-to-Video" pipelines. It focuses on orchestrating multiple AI services (LLMs for scripts, TTS for audio, and Stock/AI APIs for visuals) and assembling them into a final render using programmatic video editing.

## Workflow

### 1. Pipeline Orchestration
A production-grade video generator should follow a linear synchronous pipeline:
**Script $\to$ Voiceover $\to$ Visual Mapping $\to$ Assembly $\to$ Render.**

- **Scripting**: Break the narrative into "segments." Each segment must contain the spoken text and a corresponding "visual keyword" for asset sourcing.
- **Audio Generation**: Use high-quality neural TTS (e.g., `edge-tts` for free, high-quality voices) to generate the master audio track first. This provides the "clock" for the video duration.
- **Visual Sourcing**: 
    - Map each script segment to a visual keyword.
    - Source clips via APIs (e.g., Pexels, Pixabay) or generate images (DALL-E/Midjourney) and animate them.
    - **Pro Tip**: Always download clips to a local `out/` directory to avoid network timeouts during the render phase.

### 2. Assembly & Rendering (MoviePy Pattern)
Use `MoviePy` for programmatic assembly:
- **Duration Matching**: Calculate `segment_duration = total_audio_duration / number_of_segments`.
- **Clip Trimming**: Force every sourcing clip to match the `segment_duration` using `.subclip(0, segment_duration)`.
- **Cinematic Overlays**: 
    - Implement "Glassmorphism" captions: A semi-transparent white box (`ColorClip` with low opacity) behind a high-contrast `TextClip`.
    - Position captions in the lower third or center for a modern aesthetic.
- **Concatenation**: Use `concatenate_videoclips(clips, method="compose")` to ensure consistent sizing.

### 3. Integration Backend (FastAPI/Flask)
Since rendering is CPU/GPU intensive, never run it in a request-response cycle.
- **Async Job Queue**: Use a `BackgroundTasks` (FastAPI) or a task queue to process the render.
- **Job Tracking**: Provide a `/status/{job_id}` endpoint so the frontend can poll for completion.
- **Static Delivery**: Mount the output directory as a static folder (e.g., `/out`) to allow direct video downloads.

## Pitfalls & Lessons
- **MoviePy Versioning:** MoviePy v2.0+ changed the import structure and method names. 
    - **Imports**: **Do not use** `from moviepy.editor import ...`. Use top-level imports: `from moviepy import TextClip, ColorClip, ...`.
    - **Method Changes**: The `set_` methods were renamed to `with_`. Use `.with_duration()`, `.with_position()`, and `.with_opacity()` instead of `set_duration()`, `set_position()`, and `set_opacity()`.
    - **TextClip API**: The `font` argument is now the first positional argument. To avoid `multiple values for argument 'font'`, always use keyword arguments (e.g., `TextClip(text=\"Hello\", font=\"Arial\")`). Also, use `font_size` instead of `fontsize`.
- **Pexels API Methods:** The `pexels-api` Python wrapper uses `.search()` for both images and videos, not `.search_videos()`. Verify the API object methods if an `AttributeError` occurs.
- **Resource Exhaustion:** Rendering high-res video is heavy. In Termux/Android, limit resolution to 720p and FPS to 24 to avoid crashes.
- **FFmpeg Dependencies:** MoviePy requires FFmpeg. If `ImageMagick` is missing, `TextClip` will fail. Ensure it is installed or use a custom drawing function via OpenCV/PIL if `ImageMagick` is unavailable.
- **Font Availability in Termux:** `TextClip` failures like `cannot open resource` occur when the requested font is not installed. Use `fc-list` (via `pkg install fontconfig-utils`) to verify available system fonts. In Android/Termux, `Roboto-Bold` is a reliable default.
- **Network Flakes during Pip:** If `pip install` fails with `ReadTimeoutError` or `ConnectionResetError`, use `python3 -m pip install --quiet` or check connectivity to PyPI.
- **Hyperframes (Headless Browser Rendering):** When using `hyperframes` for high-visual / programmatic animations:
    - **Deterministic Capture:** Avoid `repeat: -1` in GSAP; use finite counts based on total duration to prevent capture engine hangs.
    - **Runtime API:** Expose `window.__hf = { duration, seek }` and register timelines in `window.__timelines` to enable the renderer to seek and capture frames reliably.
    - **Android/Proot Chromium:** The Ubuntu `chromium-browser` package is a snap wrapper and will fail in Proot. Use the Termux-native `chromium` binary and point `HYPERFRAMES_BROWSER_PATH` to the Termux path (e.g., `/data/data/com.termux/files/usr/bin/chromium-browser`).
    - **Hardware Accel:** Headless GPU acceleration often fails in Proot/Android; be prepared for software fallback (screenshot mode), which is slower but functional.

- **API Rate Limits:** When sourcing 20+ clips for a 3-minute video, implement small sleep delays between Pexels/Pixabay requests to avoid 429 errors.
- **Audio Sync:** Always derive video length from the audio file, not the other way around. Audio is a linear constant; video is a collection of variables.

## Verification
- [ ] Voiceover matches the generated script.
- [ ] Visual clips correctly correspond to the keywords in each segment.
- [ ] Final MP4 is playable and contains the correct audio track.
- [ ] Frontend allows script input and provides a download link upon completion.
