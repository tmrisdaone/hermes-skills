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
Ensure the Zink driver and Vulkan ICD are installed. Use `vulkan-loader-generic` for the best compatibility with the freedreno ICD.
```bash
pkg update
pkg install mesa-zink virglrenderer-mesa-zink vulkan-loader-generic mesa-vulkan-icd-freedreno virglrenderer-android vulkan-tools mesa-demos
```

### 2. Start the GPU Bridge (virgl_test_server)
For Proot-Ubuntu (and other distributions) to access the GPU, a server must be running in the native Termux environment to bridge OpenGL commands to the Vulkan driver.

**Command to start the server:**
```bash
MESA_NO_ERROR=1 MESA_GL_VERSION_OVERRIDE=4.3COMPAT MESA_GLES_VERSION_OVERRIDE=3.2 GALLIUM_DRIVER=zink ZINK_DESCRIPTORS=lazy virgl_test_server --use-egl-surfaceless --use-gles &
```

**To make this start automatically on Termux open:**
Create a separate script and source it in `~/.bashrc`. This ensures the bridge is always active before you enter a proot container.
```bash
echo 'MESA_NO_ERROR=1 MESA_GL_VERSION_OVERRIDE=4.3COMPAT MESA_GLES_VERSION_OVERRIDE=3.2 GALLIUM_DRIVER=zink ZINK_DESCRIPTORS=lazy virgl_test_server --use-egl-surfaceless --use-gles &' > ~/.bashrc_gpu_accel
echo 'source ~/.bashrc_gpu_accel' >> ~/.bashrc
```

### 3. Set Environment Variables
For Zink to function, the following variables must be exported. **Crucial: Verify that the .aarch64.json file exists in the path below. If using vulkan-loader-generic, this is the standard path.**

**Command to activate in current session:**
```bash
export VK_ICD_FILENAMES=/data/data/com.termux/files/usr/share/vulkan/icd.d/freedreno_icd.aarch64.json
export GALLIUM_DRIVER=zink
export MESA_LOADER_DRIVER_OVERRIDE=zink
```

### 4. Proot Implementation (Ubuntu/Debian)
When running a program inside `proot-distro`, you must export the driver and override variables inside the container to utilize the native Termux Zink bridge.

**Example Command:**
```bash
proot-distro login ubuntu --termux-home -- bash -c "export GALLIUM_DRIVER=zink MESA_GL_VERSION_OVERRIDE=4.0 && program_name"
```

### 4. Proot Implementation (Ubuntu/Debian)
When running a program inside `proot-distro`, you must export the driver and override variables inside the container to utilize the native Termux Zink bridge.

**Example Command:**
```bash
proot-distro login ubuntu --termux-home -- bash -c "export GALLIUM_DRIVER=zink MESA_GL_VERSION_OVERRIDE=4.0 && program_name"
```

### 3. Make Permanent
Add the exports to `~/.bashrc` and source it:
```bash
echo 'export VK_ICD_FILENAMES=/data/data/com.termux/files/usr/share/vulkan/icd.d/freedreno_icd.aarch64.json' >> ~/.bashrc
echo 'export GALLIUM_DRIVER=zink' >> ~/.bashrc
echo 'export MESA_LOADER_DRIVER_OVERRIDE=zink' >> ~/.bashrc
source ~/.bashrc
```

## Verification
Run the following to confirm the GPU is being used:
```bash
glxinfo | grep "OpenGL renderer"
```
**Success:** The output should mention `zink` or `Adreno`.
**Failure:** If it says `llvmpipe`, the variables are not correctly exported or the ICD path is wrong.

## Pitfalls
- **ICD Filename Mismatch:** The driver file is often named `freedreno_icd.aarch64.json`, not just `freedreno_icd.json`. Using the wrong filename will result in `ERROR_INCOMPATIBLE_DRIVER`.
- **ELF Header Mismatch:** Zink works natively in Termux. DO NOT try to point a `proot-distro` (Ubuntu) guest to Termux's `.so` files; it will fail with an "invalid ELF header" error because of the glibc/Bionic mismatch. For GPU work, use the native Termux environment.
- **GitHub API Integration:** When publishing related tools or configs to GitHub, use the GitHub API (`/user/repos`) to create the repository before attempting to push. Use tokens in the remote URL (`https://<token>@github.com/...`) for seamless authentication in non-interactive environments.
