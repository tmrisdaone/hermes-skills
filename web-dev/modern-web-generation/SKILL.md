---
name: modern-web-generation
description: Create and host visually modern, single-page sites — glassmorphism luxury OR futuristic cyberpunk, with full e-commerce interactivity.
triggers:
  - modern website
  - landing page
  - glassmorphism
  - futuristic design
  - cyberpunk
  - e-commerce site
  - shopping cart
  - product grid
  - visually appealing web
  - host local website
---

# Modern Web Generation & Hosting

## Overview
This skill governs creating high-visual single-page websites, including full e-commerce stores, and hosting them locally. It covers two distinct design dialects — **luxury glassmorphism** and **futuristic cyberpunk** — and includes patterns for shopping cart, wishlist, product modal, filters, sort, search, and LocalStorage persistence.

---

## Design Dialects

### A. Luxury Glassmorphism
- **Glassmorphism:** `backdrop-filter: blur(20px)`, semi-transparent `rgba(255,255,255,0.03-0.08)` backgrounds, thin `1px solid rgba(255,255,255,0.06)` borders.
- **Typography:** Pair serif for headings (Playfair Display) with clean sans-serif for body (Inter, Plus Jakarta Sans).
- **Color Palette:** Neutral base (white/off-white/deep black) + one vibrant accent (gold, indigo).
- **Depth:** 3D transforms (`translateZ`, `perspective`), Three.js MeshPhysicalMaterial for crystalline/metallic hero elements.
- **Easing:** `cubic-bezier(0.2, 0.8, 0.2, 1)` for organic weighted movement.
- **Micro-interactions:** Card tilt effects via `rotateX`/`rotateY` from mouse position relative to card center.

### B. Futuristic / Cyberpunk (Dark Neon)
- **Background:** Deep dark (`#0a0a0f`) with animated floating gradient orbs (200-500px, blur 120px, slow drift keyframes), a grid overlay (`1px rgba(0,240,255,0.03)` lines at 60px spacing), and floating particle spans (random size, position, animation-delay, cyan/magenta colors).
- **Typography:** Orbitron (display/headings) + Inter (body) + JetBrains Mono (prices/data). All-caps uppercase nav links with underline hover effect.
- **Color Palette:** Dark base + neon cyan (`#00f0ff`), neon magenta (`#ff00e6`), purple (`#7c3aed`), blue (`#3b82f6`), green (`#00ff88`).
- **Glow Effects:** `box-shadow` with `0 0 10px rgba(cyan,0.5), 0 0 40px rgba(cyan,0.15)` for neon halos on interactive elements.
- **Hero Ring:** Orbital animation — outer ring spins 30s linear, a conic-gradient `::before` spins reverse 10s, inner circle has backdrop-filter and centered icon.
- **Mouse-Reactive Glow:** Each product card has a `radial-gradient(circle at var(--mx,50%) var(--my,50%), ...)` overlay — listen for `mouseover` on `.product-card`, compute `--mx`/`--my` as percentage of card rect, set CSS custom properties.
- **Particle System:** Generate `<span>` elements into a fixed container, each with random size (1-4px), position, animation-duration (10-25s), and color (cyan or magenta). Animate `translateY` from bottom to top with opacity fade-in/out.

---

## Interaction Style (Txrao)

This user communicates in short, action-oriented commands. Signals observed:
- **Terse directives**: "START it up", "Can u continue from where u left off?"
- **No explanation needed**: Just execute — don't ask "what next?" or explain what you're about to do, just do it.
- **Proactive continuation**: After delivering a feature, continue building without waiting for explicit "now add X" prompts. The expectation is autonomous progression.
- **Functional corrections > stylistic preferences**: Corrections are about what the site *does* (buttons need to link somewhere) not how it *looks*.
- **Fallback planning**: When a user says "try X, and if that doesn't work use Y", execute X immediately, and if it fails, pivot to Y without asking permission. They've already given the go-ahead.
- **Concise execution reports**: After completing an operation, state what was done and what the next thing is — no preamble, no "I've gone ahead and..." framing. Just the facts.

Embed this preference in the task loop: plan → execute → report concisely → repeat. Skip the "shall I?" questions on obvious next steps.

---

## Multi-Page Site Architecture

When a single-page store needs sub-pages (About, Contact, Support, Careers, Press, Legal), create standalone `.html` files that share the same theme, header, footer, CSS, and JS. Do NOT use a Flask/Jinja2 backend — keep it pure static HTML served by Python's http.server.

### Page Shell Pattern

Each sub-page needs the complete HTML structure inlined (no template inheritance):

1. **Copy the `<style>` block** from `index.html` into each page (or reference a shared `style.css`)
2. **Copy the background elements** (`.bg-grid`, `.bg-glow-*`, `#bg-particles`)
3. **Copy the header** with navigation links pointing to actual pages (not `#` anchors)
4. **Insert page-specific content** under `<main>`
5. **Copy the footer** with all links pointing to actual pages
6. **Copy the JS** (particles, scroll reveal, header scroll, back-to-top, toast)

### Navigation Wiring

| Link destination | File target |
|---|---|
| Home / Logo | `index.html` |
| Shop | `index.html#products` (anchor to product section) |
| About | `about.html` |
| Careers | `careers.html` |
| Press | `press.html` |
| Contact | `contact.html` |
| Support | `support.html` |
| Privacy | `privacy.html` |
| Terms | `terms.html` |
| Cookies | `cookies.html` |
| GDPR | `gdpr.html` |

### Template Migration (Jinja2 → Static HTML)

When migrating from Flask/Jinja2 templates to standalone HTML:

1. **Extract body content**: Strip `{% extends "base.html" %}`, `{% block %}...{% endblock %}` wrappers
2. **Replace `{{ url_for('page_name') }}`** with the actual file path (e.g., `about.html`, `index.html#products`)
3. **Rebrand**: Search-and-replace old brand name with new (use `patch` with `replace_all=true`)
4. **Replace email domains**: `@oldbrand.io` → `@newbrand.io`
5. **Add page-specific JS handlers**: Contact form submit → toast, newsletter submit → toast
6. **Verify all pages return `200`** via `curl -s -o /dev/null -w "%{http_code}"`

### Bulk Rebranding Pattern

To rename a brand site-wide (e.g., NEXUS → Txrao):

```bash
# Find all occurrences first
grep -rn "OLD_NAME" *.html

# Patch each occurrence with replace_all=true
patch(path="file.html", old_string="OLD_NAME Product", new_string="NEW_NAME Product")
patch(path="file.html", old_string="OLD_NAME Corp", new_string="NEW_NAME Corp")
# ... etc for each distinct text pattern
```

For email domains, chain replacements:
```
"mailto:hello@oldbrand.io" → "mailto:hello@newbrand.io"
"@oldbrand.io" → "@newbrand.io"
```

### Page-Specific Interactivity

- **Contact form**: Add `submit` handler that calls `showToast()` and `reset()` instead of actual form submission
- **Newsletter signup**: Same pattern — prevent default, toast, reset
- **Legal pages**: No interactive elements needed beyond navigation and back-to-top

### Verification Checklist

- [ ] Every `<a href="#">` in the project is resolved to an actual page
- [ ] All sub-pages return HTTP 200
- [ ] Header/footer navigation works in both directions (page A → page B → page A)
- [ ] Brand name is consistent across all pages
- [ ] Theme (CSS, background, particles, scroll reveal) matches the main page

---

## E-Commerce Architecture (Single-File)

Build the entire store as **one HTML file** with embedded `<style>` and `<script>`.

### Data Layer
```js
const products = [
  { id: 1, title: 'Aura Band', desc: '...', price: 299, category: 'wearables', badge: 'new', icon: '◈', originalPrice: null },
  ...
];
const categoryNames = { wearables: 'Wearables', audio: 'Audio', accessories: 'Accessories', gear: 'Gear' };
```

### State Management
```js
let cart = [];
let wishlist = new Set();
let activeCategory = 'all';
let searchQuery = '';
let sortMode = 'default';
const STORAGE_KEY = 'store_cart';
```

### Core Components

| Component | Implementation |
|---|---|
| **Product Grid** | `renderProducts()` — filter by category + search query, sort by price/name, render cards via template literal, re-attach event listeners after innerHTML replacement. Show "no results" placeholder when empty. |
| **Shopping Cart** | Sidebar (fixed right, `translateX(100%)` → `0`), overlay backdrop. Items rendered dynamically with +/- qty, remove, subtotal. Checkout clears cart. |
| **Cart Persistence** | `saveCart()` — `localStorage.setItem(STORAGE_KEY, JSON.stringify(cart))`. `loadCart()` — called on init. Call `saveCart()` after every mutation (add, remove, qty change). |
| **Wishlist** | Heart toggle on each product card. Toggle via `Set.add/delete`. Re-render grid to reflect state. Not persisted (session-only). |
| **Product Modal** | Overlay with split layout (icon left, details right). Shows category, title, price, full description, specs list (warranty/free shipping/returns/support). "Add to Cart" button calls `addToCart()`. |
| **Category Filter** | Button group, `.active` class on selected. `data-cat` attribute for category name. Click handler sets `activeCategory` and re-renders. |
| **Sort Dropdown** | `<select>` with options: default, price-asc, price-desc, name. Handler calls `renderProducts()` with sort logic. |
| **Search** | Text input with `input` + `keydown(Enter)` + button click handler. Filters by title, description, or category name (lowercased). |
| **Toast Notifications** | Fixed container bottom-right. Create `<div>` with animation (translateX in), auto-remove after 2.5s with fade-out transition. |
| **Animated Counters** | Elements with `data-target` attribute. On scroll/load, increment from 0 to target over ~60 frames using `setInterval(25ms)`. Append `+` suffix for large numbers. |

### Scroll Reveal
```js
let revealObserver = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      entry.target.classList.add('visible');
      revealObserver.unobserve(entry.target);
    }
  });
}, { threshold: 0.1, rootMargin: '0px 0px -40px 0px' });
```
Apply class `.reveal` (opacity:0, translateY:40px → opacity:1, translateY:0 with 0.8s ease transition).

---

## Hosting (Termux / Local)

```bash
cd /path/to/project
python3 -m http.server 8080
```

**Background process pattern** (for Termux):
```bash
# Start
cd /path && python3 -m http.server 8080 &

# Kill previous first (port conflict)
pkill -f "python3 -m http.server 8080" 2>/dev/null

# Verify
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/
```

**Important:** The default file served is `index.html` — no directory listing appears. The "embedded Python host" pattern (base64 encoding HTML into a Python script) is unnecessary unless you need a fully portable single `.py` executable.

---

## Pitfalls & Lessons

- **No linter for HTML** — the `write_file` tool won't syntax-check HTML. Test by serving and viewing in browser, checking console for JS errors.
- **Event listeners after innerHTML** — always re-attach event listeners after setting `innerHTML` on the grid/cart container. Use data attributes (`dataset.id`, `dataset.delta`) to identify targets.
- **Cart badge animation** — use `transform: scale(0)` → `scale(1)` with `cubic-bezier(0.68, -0.55, 0.27, 1.55)` for a springy reveal. Toggle `.visible` class.
- **Mouse glow on cards** — the glow overlay uses CSS custom properties. The `mouseover` event must compute position relative to the card's `.product-image` rect, not the full card.
- **LocalStorage errors** — wrap `getItem`/`setItem` in try-catch (private browsing / storage quota).
- **Mobile** — hide nav-links via `display:none` at ≤600px. Cart sidebar goes full-width. Product grid goes 2 columns. Newsletters form stacks vertically.

## Verification
- [ ] Site loads at `http://localhost:8080` (no directory listing).
- [ ] Product grid renders, filters by category, responds to search and sort.
- [ ] Add to cart → badge updates → cart sidebar shows items → qty +/- works.
- [ ] Cart persists across page refresh (LocalStorage).
- [ ] Product card click opens modal with full details.
- [ ] Wishlist heart toggles visually.
- [ ] Animated counters count up on load.
- [ ] Scrolled sections fade in (IntersectionObserver).
- [ ] Back-to-top button appears after 500px scroll.
- [ ] Responsive layout works on mobile widths (<600px).