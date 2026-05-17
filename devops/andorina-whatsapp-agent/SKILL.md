---
name: andorina-whatsapp-agent
description: Guide for installing and operating the Andoriña WhatsApp Agent for Hermes.
---

# 🕊️ Andoriña WhatsApp Agent Operations

This skill governs the deployment and usage of the Andoriña AI WhatsApp bridge. It provides a structured approach to messaging, contact discovery, and system maintenance.

## 🚀 Deployment & Installation

1. **Clone Repository:**
   `git clone https://github.com/AndorinaAI/Andorina-WhatsApp-Agent-for-Hermes.git`
2. **Installation:**
   Run the installation script: `bash Andorina-WhatsApp-Agent-for-Hermes/versions/v1.0.2/install.sh`
3. **Configuration:**
   Follow the interactive prompts to select language, agent (usually `hermes`), country prefix, and phone number.
4. **Critical Setup:**
   - **Google Contacts API:** Must be enabled for name-based searching.
   - **Qdrant Memory:** Required for persistent agent "soul" and memory.
   - **Autostart:** Enable to ensure multimedia/audio support works after reboots.

## 🛠️ Operational Workflow

### 1. Contact Discovery (The Exhaustive Search)
NEVER report a contact as missing without this sequence:
- **Step A:** `python3 scripts/contacts.py search "Name"`
- **Step B:** If A fails, run `python3 scripts/contacts.py groups` to check for group-based identities.
- **Step C:** If B fails, run `python3 scripts/contacts.py refresh` to sync cloud data, then repeat Step A.
- **Step D:** If all fail, run `python3 scripts/diag.py` to check bridge connectivity.

### 2. Messaging & Interaction
- **Text:** `python3 scripts/send.py message "ID" "Message Text"`
- **Files/Voice:** `python3 scripts/files.py "Path" "ID" [--voice]`
- **Reading:** `python3 scripts/inbox.py list` (recent) $\rightarrow$ `python3 scripts/inbox.py read "ID"` (details).

### 3. Maintenance & Health
- **Bridge Failures:** If the bridge is offline or unresponsive, run `python3 scripts/bridge_health.py` for auto-repair.
- **Rate Limits:** Check `python3 scripts/guard.py status` if messages are not sending.
- **General Health:** Run `python3 scripts/diag.py`.

## ⚠️ Pitfalls & Constraints
- **Empty Inbox:** A completely empty `inbox.py list` immediately after installation typically indicates the bridge is not yet authenticated/linked.
- **Case Sensitivity/Accents:** The agent uses NFD normalization, but explicit `refresh` is often needed for new contacts.
- ** Paths:** Always ensure scripts are called from the specific version directory (e.g., `versions/v1.0.2/scripts/`).
