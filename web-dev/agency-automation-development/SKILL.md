---
name: agency-automation-development
description: Framework for building and deploying AI-driven business automation tools (Lead Gen, CS Bots) for clients using Termux.
---

# Agency Automation Development

This skill governs the creation of "Productized AI Services"—tools that can be sold to businesses as high-value automations rather than simple scripts.

## Core Principles
- **Productization:** Don't just provide code; provide a "System." (e.g., move from "Lead List" $\rightarrow$ "Lead Gen Engine").
- **Integration First:** Always architect tools to connect to data sources (SQL, APIs, CRMs) so they solve a business problem.
- **Visual Appeal:** Use modern design (Glassmorphism, etc.) for any client-facing components to increase perceived value.

## Workflow: Building a Service
1. **Niche Identification:** Identify a business pain point (e.g., "Slow customer response times").
2. **Modular Architecture:**
   - **Interface Layer:** Voice, Web, or Chat (e.g., `voice.py`).
   - **Logic Layer:** AI process and decision tree (e.g., `main.py`).
   - **Data Layer:** Database connection and state management (e.g., `database.py`).
3. **Prototyping:** Use SQLite for local development before migrating to client-side PostgreSQL/MySQL.
4. **Deployment:** Host via standalone Python implementations in Termux to ensure stability and easy migration.

## Implementation Patterns
- **The "SaaS" structure:** Keep agency projects in structured directories: `~/ai_agency/<project_name>/`.
- **Database Mocking:** Always include a `.seed_data()` method in database handlers to allow for immediate demonstration of the "Value Proposition" to clients.

## Pitfalls & Lessons
- **The "List" Trap:** Avoid selling raw data. Sell the **result** (meetings booked, tickets resolved) to shift from low-cost freelancer to high-ticket consultant.
- **GUI Limitations:** Remember that the agent cannot visually interact with VNC/X11; focus on the backend "plumbing" and the user's frontend experience.

## Templates
- See `templates/cs_bot_structure.txt` for the base modular layout.
