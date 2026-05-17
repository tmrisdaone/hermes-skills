---
name: android-termux-environment
description: Guidelines for operating and configuring the development environment on Android 13 via Termux.
---

# Android Termux Development

This skill governs the setup and management of a Linux-like development environment on Android.

## Core Environment
- **Host:** Android 13 (Samsung SM-G781U1)
- **Shell:** Termux
- **Home Directory:** `/data/data/com.termux/files/home`

## Package Management
- Use `pkg install <package>` for system-level binaries (e.g., `ffmpeg`, `yt-dlp`).
- Always use the `-y` flag in automated scripts to prevent hanging on prompts.
- **Mirror Setup:** If `apt` warnings appear regarding mirrors, use `termux-change-repo` to select a stable mirror.

## Python Development
- **Venv Management:** Prefer `.venv` or `venv` for project isolation.
- **Installation:** Use `pip install -e .` for editable installs of local repositories.
- **Build Issues:** Be aware that heavy C-extensions (e.g., `matplotlib`, `numpy`) may take significant time to build on ARM64 Android devices and can occasionally time out in agent tool calls.

## Device Automation
- **Preferred Method:** Use **Shizuku (via rish)** over standard ADB for interacting with the Android system from within Termux.

## Document Processing (DOCX on Termux)

### Extracting Content from .docx Files
.docx files are ZIP archives. Process them without python-docx (which requires lxml, a heavy C-extension that often fails or times out building on ARM64 Android):

```bash
# Extract content
unzip file.docx word/document.xml

# Extract all embedded images
unzip file.docx "word/media/*"
```

Map image order to filenames by parsing `word/_rels/document.xml.rels` and `word/document.xml`:

```python
import re, zipfile, xml.etree.ElementTree as ET

# Build rId -> filename mapping from rels
with zipfile.ZipFile("file.docx") as z:
    rels = z.read("word/_rels/document.xml.rels").decode()
rid_map = {}
for m in re.finditer(r'Id="([^"]*)"[^>]*Target="(media/[^"]*)"', rels):
    rid_map[m.group(1)] = m.group(2)

# Get ordered image references from document.xml
content = open("word/document.xml").read()
ordered_rids = re.findall(r'r:embed="([^"]*)"', content)
image_order = [rid_map[rid] for rid in ordered_rids]
```

### Building .docx Files Manually
When python-docx is unavailable, construct .docx files using Python's built-in `zipfile` and `xml.sax.saxutils`:

1. Create `word/document.xml` with OpenXML namespace `w:http://schemas.openxmlformats.org/wordprocessingml/2006/main`
2. Create required boilerplate: `[Content_Types].xml`, `_rels/.rels`, `word/_rels/document.xml.rels`, `word/styles.xml`, `word/fontTable.xml`, `word/settings.xml`
3. Zip everything together with `ZIP_DEFLATED`

**Key XML elements:**
- `<w:p>` = paragraph, `<w:r>` = run (formatted text segment), `<w:t>` = text content
- Run properties (`<w:rPr>`): `<w:b/>` = bold, `<w:i/>` = italic, `<w:u w:val="single"/>` = underline
- `<w:rFonts w:ascii="Times New Roman" w:hAnsi="Times New Roman"/>` = font
- `<w:sz w:val="22"/>` = font size in half-points (22 = 11pt)
- Paragraph properties (`<w:pPr>`): `<w:jc w:val="both"/>` = justify, `<w:spacing w:after="120"/>` = spacing

### Batch Transcribing Images with Vision AI
For documents with many embedded images (e.g., scanned book pages):

```python
# Use delegate_task with vision toolset for parallel processing
delegate_task(tasks=[
    {"goal": "Transcribe images batch 1-10", "toolsets": ["vision"]},
    {"goal": "Transcribe images batch 11-20", "toolsets": ["vision"]},
], ...)
```

Each subagent receives local file paths and uses `vision_analyze` to extract text. Up to 3 concurrent tasks.

## Hermes Config Editing Style

When editing `~/.hermes/config.yaml` or any YAML config file:

- **Use file tools** (`read_file`, `patch`, `write_file`) for config edits — NOT Python scripts via terminal. The user expects direct file manipulation over scripting.
- Check the config with `read_file` or `search_files` first to see current state.
- Apply changes with `patch` for targeted changes, or `write_file` for full rewrites.
- After editing, verify with a quick `grep` or `read_file` on the changed section.

## User Interaction Style

This user communicates very directly and expects the same from you:

- **Be concise, be fast.** No preamble, no "sure!", no "great question!" — just do it.
- **Lead with the result.** Summary first, details only if they ask.
- **When they report a crash, fix it.** Don't explain what might be wrong, don't ask for logcat — find and fix.
- **Use sub-agents for parallel analysis.** When they say "find all bugs in X," delegate to parallel subagents focusing on different crash paths.
- **Surgical fixes only.** When fixing a working app that crashes on specific actions, change ONLY the crash-causing lines. Don't refactor, don't "improve" surrounding code, don't modernize patterns.
- **Revert aggressively.** If a fix breaks launch, revert immediately — the original pattern was fine.
- **They use slang** ("gang", "bro", "fr") — match their energy, don't lecture.

\n## GPU Acceleration (Vulkan/Zink)\n\n### Host-Side Setup (Termux)\nInstall Vulkan drivers for Adreno GPUs:\n```bash\npkg install mesa-vulkan-icd-freedreno mesa-zink vulkan-tools\n```\nVerify hardware visibility with `vulkaninfo | grep 'GPU'`. If `GPU0` is integrated, the host is ready.\n\n### Guest-Side Acceleration (Proot-Distro)\n`proot` cannot load kernel modules, meaning guests defaults to software rendering (`llvmpipe`). To utilize hardware acceleration:\n1. **Drivers:** Install `mesa-vulkan-drivers` and `vulkan-tools` inside the guest.\n2. **Bridge:** Route calls to the host by setting the Vulkan ICD path at launch:\n   ```bash\n   export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/freedreno_icd.json\n   proot-distro login ubuntu\n   ```\n3. **Verification:** Run `vulkaninfo` inside the guest. If it still reports `llvmpipe`, the bridge is not active.\n\n**Pitfall:** Proot isolation often blocks the necessary device node access for full acceleration without specific `--bind` flags or environment overrides. Hardware acceleration in proot is a "best-effort" translation via Zink/Virgl, not native passthrough.

See `references/hermes-update-termux.md` for the update workflow — `git pull`, `pip install --no-deps -e .` (psutil won't build on Android), and clearing the stale `.update_check` cache.

## Hermes Voice/TTS Configuration

See `references/hermes-voice-tts-config.md` for full voice configuration reference — TTS providers, key settings, voice selection, and testing.

## Hermes Spotify Authentication

See `references/hermes-spotify-auth.md` for Spotify OAuth setup — creating a Spotify Developer app, client ID, and running the auth flow on Termux.

## Android APK Build in Proot-Distro

See `references/android-apk-build-proot.md` for building Android APKs inside proot-distro on ARM64 — SDK setup, the critical AAPT2 architecture fix (SDK ships x86_64 aapt2, needs ARM64 replacement), icon resource creation, and common Kotlin/Compose crash patterns (LifecycleOwner cast, background state mutation, main-thread IO, OOM, etc.).

## Parallel Subagent Bug Analysis

See `references/parallel-subagent-bug-analysis.md` for the technique of using parallel subagents to analyze unfamiliar codebases — each subagent reads the same files but focuses on different crash paths, producing a consolidated fix plan.

## Workmode Template

See `templates/workmode.md` for the flow-state execution mode template — paste this into a project's workmode.md to put the agent into efficient execution mindset.

## Pitfalls & Lessons
- **Interactive Prompts:** Standard `apt`/`pkg` commands may abort if they encounter a prompt; always ensure non-interactive flags are used.
- **Resource Constraints:** Large builds on Android may trigger timeouts; increase timeout values or run in `background=true` if the process is long-lived.
- **python-docx on Termux:** The `lxml` dependency requires building a C-extension on ARM64 and frequently times out. Prefer manual XML+zipfile construction (technique above) or install with `--only-binary=:all:` flags if a wheel exists.
