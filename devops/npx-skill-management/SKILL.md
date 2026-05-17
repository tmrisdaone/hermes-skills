---
name: npx-skill-management
description: Managing Hermes skills using the npx skills CLI.
category: devops
---

# NPX Skill Management

Guidelines for discovering, installing, and operating skills via the `npx skills` CLI.

## Installation
Use `npx skills add <provider/skill>` to install. 
- Use the `-y` flag to bypass interactive prompts.
- Use the `-g` flag for global installation where applicable.

## Proot-Distro Execution
Some skills rely on native binaries (e.g., `onnxruntime`) that are not compiled for Android/ARM64. When a skill fails with "Unsupported Platform" or binary errors:

1. **Use Proot-Distro**: Wrap the command in a proot login to simulate a standard Linux environment.
   - Command: `proot-distro login ubuntu --termux-home -- <command>`
2. **Absolute Pathing**: Always use absolute paths when passing files (compositions, assets) into the proot environment to avoid home-directory mapping confusion.
3. **Dependency Sync**: If a skill requires specific system libraries, install them via `apt` within the proot session first.
