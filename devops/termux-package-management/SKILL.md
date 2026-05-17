---
name: termux-package-management
description: Guidelines for installing and managing large-scale software suites and automation scripts within Termux on Android.
---

# Termux Package & Suite Management

Installing complex environments (e.g., hacking labs, full Linux distros, desktop environments) in Termux requires specific handling due to resource constraints, timeouts, and Android's background process management.

## Workflow for Large Installations

1. **Clone & Inspect:** Always clone the repository and check for an `install.sh` or `setup.py` before executing.
2. **Handle Timeouts:** Large scripts (like `termux-hacklab`) frequently exceed the standard execution timeout of AI agents.
   - **Never** run massive installation scripts in the foreground if they are expected to take >5 minutes.
   - **Action:** Use background execution (e.g., `bash script.sh &` or agent-specific background tools) to prevent session timeouts.
3. **Monitoring:** Use `ps aux` or process polling to verify the script is still active.
4. **Log Verification:** If the script supports it, redirect output to a log file for later auditing.

## Pitfalls & Troubleshooting

- **Interactive Prompts:** Many installation scripts expect user input (Press Enter to continue). If run blindly in the background, they may hang.
  - *Fix:* Use `yes | bash install.sh` or check if the script has a `--yes` or `-y` flag.
- **Package Mirror Issues:** Termux mirrors can be slow or unstable.
  - *Fix:* Suggest `termux-change-mirror` if installation hangs indefinitely at the "Updating package lists" stage.
- **Binary Incompatibilities (npm/Platform):** Some npm packages (e.g., tunnel software) distribute pre-compiled binaries that fail in Termux because the OS is identified as `android` rather than `linux`.
  - *Symptom:* `EBADPLATFORM` error during installation or "Syntax error / ELF" errors when executing.
  - *Pitfall:* Using `--force` or `--ignore-platform` may allow installation, but the resulting binary will likely fail to execute because it's not linked for the Termux environment.
  - *Fix:* Favor pure JavaScript implementations or official Termux packages over pre-compiled npm binaries.
- **GPU/Driver Failures:** Android GPU drivers (Turnip/Zink) often fail on specific chipsets.
  - *Fix:* If a GPU step fails but the rest of the script continues, notify the user but proceed with the installation as the system may still function in software rendering mode.

## Verification Steps
- Check for the existence of the installed environment (e.g., check if `.vnc` or `~/.termux-x11` directories were created).
- Verify the primary binary of the suite is in the path.
