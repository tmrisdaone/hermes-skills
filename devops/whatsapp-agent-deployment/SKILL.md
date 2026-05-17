---
name: whatsapp-agent-deployment
description: Guidelines for installing and configuring the Andoriña WhatsApp Agent for Hermes.
---

# 🕊️ WhatsApp Agent Deployment (Andoriña)

This skill covers the installation and first-run configuration of the Andoriña WhatsApp Agent bridge.

## 🚀 Installation Workflow
1. **Clone Repository:**
   `git clone https://github.com/AndorinaAI/Andorina-WhatsApp-Agent-for-Hermes.git`
2. **Run Installer:**
   Navigate to the latest version directory (e.g., `versions/v1.0.2/`) and execute:
   `bash install.sh`
3. **Interactive Setup:**
   Follow the prompts to select language, agent (usually `hermes`), country prefix, and WhatsApp number.
4. **Essential Components:**
   - **Google Contacts API:** Required for searching contacts by name. Requires `GOOGLE_CONTACTS_CLIENT_ID` and `GOOGLE_CONTACTS_CLIENT_SECRET` in `~/.hermes/.env`.
   - **Qdrant Memory:** Required for persistent agent "soul" and memory across reboots.
   - **Autostart:** Should be enabled to ensure multimedia/audio files are sent reliably.

## 🛠️ First-Run Connection
The bridge typically requires an interactive session to authorize the WhatsApp account (QR Code scanning).

### Connection Sequence:
1. **Health Check:** Run `python3 scripts/bridge_health.py` to restart the bridge on port 3000.
2. **Diagnosis:** Run `python3 scripts/diag.py` to verify if the Bridge and Memory Engine are online.
3. **Authentication:** If the bridge is offline or not linked, the user must run the bridge startup script (or `bridge_health.py`) and check the bridge logs/terminal for the registration QR code.

## ⚠️ Pitfalls & Troubleshooting
- **Silent Inbox:** If `inbox.py list` is empty but the bridge is "Online", the session may not be fully authenticated or synchronized.
- **Contact Not Found:** Always try `contacts.py refresh` before reporting a contact as missing.
- **Port 3000:** The bridge operates on port 3000. If this port is blocked, the connection will fail.
- **Case Sensitivity:** Use normalization (NFD) via `contacts.py` to avoid issues with accents/special characters.

## 🧰 Core Toolkit
- `scripts/contacts.py`: Search and refresh cloud contacts.
- `scripts/send.py`: Immediate message delivery.
- `scripts/inbox.py`: Read recent conversations.
- `scripts/bridge_health.py`: Repair and restart the gateway.
- `scripts/diag.py`: Overall system health check.
