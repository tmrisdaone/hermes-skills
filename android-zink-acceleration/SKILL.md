---
name: android-zink-acceleration
description: Activate and manage high-performance Zink (OpenGL over Vulkan) GPU acceleration on Android Termux.
---

# Zink GPU Acceleration on Android

This skill provides the workflow for enabling Zink, which allows OpenGL applications to run on top of Vulkan, providing near-native GPU performance on Adreno hardware.

## Trigger Conditions
- User wants to run GPU-accelerated software in Termux.
- User reports slow "llvmpipe" (software) rendering.
- Setting up high-performance graphics for AI or gaming tools on ARM64.

## Activation Steps

### 1. Install Required Packages
Ensure the Zink driver and Vulkan ICD are installed:
```bash
pkg update
pkg install mesa-zink mesa-vulkan-icd-freedreno vulkan-tools mesa-demos
```

### 2. Set Environment Variables
For Zink to function, the following variables must be exported:
- `VK_ICD_FILENAMES`: Path to the Adreno Vulkan JSON file.
- `GALLIUM_DRIVER`: Set to `zink`.
- `MESA_LOADER_DRIVER_OVERRIDE`: Set to `zink`.

**Command to activate in current session:**
```bash
export VK_ICD_FILENAMES=/data/data/com.termux/files/usr/share/vulkan/icd.d/freedreno_icd.aarch64.json
export GALLIUM_DRIVER=zink
export MESA_LOADER_DRIVER_OVERRIDE=zink
```

### 3. Make Permanent
Add the exports to `~/.bashrc`:
```bash
echo 'export VK_ICD_FILENAMES=/data/data/com.termux/files/usr/share/vulkan/icd.d/freedreno_icd.aarch64.json' >> ~/.bashrc
echo 'export GALLIUM_DRIVER=zink' >> ~/.bashrc
echo 'export MESA_LOADER_DRIVER_OVERRIDE=zink' >> ~/.bashrc
```

## Verification
Run the following to confirm the GPU is being used:
```bash
glxinfo | grep "OpenGL renderer"
```
**Success:** The output should mention `zink` or `Adreno`.
**Failure:** If it says `llvmpipe`, the variables are not correctly exported or the ICD path is wrong.

## Pitfalls
- **ELF Header Mismatch:** Zink works natively in Termux. DO NOT try to point a `proot-distro` (Ubuntu) guest to Termux's `.so` files; it will fail with an ELF header error. Use the native Termux environment for maximum speed.
- **ICD Filename:** The exact filename can change (e.g., `freedreno_icd.aarch64.json`). Always verify the file exists in `/data/data/com.termux/files/usr/share/vulkan/icd.d/`.
