---
name: lead-gen-agency-systems
description: Framework for building B2B lead generation and AI automation assets for clients.
---

# Lead Generation & AI Agency Systems

This skill governs the creation of lead-generation assets, high-visual landing pages, and AI automation bots to be sold as agency services.

## Strategic Approach

### 1. The Value Ladder
- **Low Ticket:** Verified lead lists (Name, Email, LinkedIn).
- **Mid Ticket:** Appointment setting (Charging per qualified meeting).
- **High Ticket:** Custom AI Automation (Integrated bots, lead-capture systems, and high-visual websites).

### 2. Technical Implementation (The "Elite" Stack)
When building for clients, prioritize "The Wow Factor" alongside functionality.

#### UI/UX Standards (The "Modern" Look)
- **Glassmorphism:** Use `backdrop-filter: blur()` and semi-transparent white/black overlays.
- **Bento Grids:** Organize data into rounded, disparate-sized cards.
- **High Contrast:** Deep dark backgrounds with neon accent colors.
- **Smooth Transitions:** Use CSS transitions for all hover states and page entries.

#### Backend Standards
- **Modular Design:** Separate the voice/UI layer from the business logic and the database layer.
- **Database Integration:**Use SQL-based systems (SQLite for prototypes, PostgreSQL for production) to track customer status and tickets.
- **API-First:** Ensure bots can be easily connected to CRMs or external APIs.

## Workflow for New Client Project
1. **Niche Research:** Identify high-value industries (e.g., Roofers, Dentists, SaaS).
2. **Lead Capture:** Use `Apollo.io` or `Google Maps` for raw data.
3. **Prototype Build:** Create a "Proof of Concept" (PoC) bot or landing page to show the client first.
4. **Delivery:** Host via Termux/Cloud and transition the client to a monthly retainer.

## Pitfalls & Lessons
- **Avoid "Plain" UIs:** Clients pay for the *feeling* of innovation. Never deliver a basic HTML table; always use cards and modern CSS.
- **Data Quality over Quantity:** One verified "VIP" lead is worth more to a client than 100 unverified emails.
- **Integration is Key:** A bot that doesn't save data to a database is just a toy; a bot that updates a database is a business tool.
