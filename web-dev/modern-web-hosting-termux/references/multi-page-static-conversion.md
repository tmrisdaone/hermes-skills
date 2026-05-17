# Multi-Page Static Site from Flask Templates

A reference for converting Flask/Jinja2 templates into standalone static HTML pages while preserving a consistent futuristic theme across all pages.

## Context

Created a 10-page e-commerce site (Txrao/NEXUS) from existing Flask templates. The original templates used Jinja2 `{% extends "base.html" %}` and `{{ url_for() }}` syntax. Converted to standalone HTML pages served by `python3 -m http.server`.

## Conversion Process

### Source Templates (Flask/Jinja2)
```
templates/
├── base.html          → Full layout with header, footer, bg, particles, toast
├── index.html         → Home page (products grid, hero, features, newsletter)
├── about.html         → Mission + values + timeline
├── careers.html       → Job listings
├── contact.html       → Contact form with office info
├── press.html         → Press releases
├── support.html       → Help center
├── privacy.html       → Privacy policy
├── terms.html         → Terms of service
├── cookies.html       → Cookies policy
└── gdpr.html          → GDPR compliance
```

### What to Replace

| Jinja2 Syntax | Static Replacement |
|---|---|
| `{% extends "base.html" %}` | Remove — embed base layout directly in each page |
| `{% block title %}...{% endblock %}` | Extract text for `<title>` tag |
| `{% block content %}...{% endblock %}` | Extract inner body content |
| `{{ url_for('page_name') }}` | `page_name.html` |
| `{{ url_for('static', filename='style.css') }}` | Inline CSS or `style.css` |
| `NEXUS` | `Txrao` (or whatever brand name) |
| `nexus.io` | Domain replacement |
| `mailto:privacy@nexus.io` | Email replacement |

### Page Generation Script Pattern

```python
def build_page(title, body_content):
    return f'''<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>{title}</title>
  <link href="https://fonts.googleapis.com/..." rel="stylesheet" />
  <style>
    /* Full CSS from base template — same on every page */
  </style>
</head>
<body>
  <!-- Background elements (grid, glow orbs, particles) -->
  <!-- Header with nav links pointing to .html pages -->
  <main>{body_content}</main>
  <!-- Footer with same .html links -->
  <!-- Toast container -->
  <!-- JS: particles, scroll reveal, header scroll, back-to-top, toast -->
</body>
</html>'''
```

### Shared Elements Per Page
Every sub-page must include these for visual consistency:
- `bg-grid`, `bg-glow-*`, `bg-particles` divs (animated background)
- Header with `<a href="index.html" class="logo">` and nav links to other `.html` pages
- Footer with the same 3-column layout (Shop, Company, Legal)
- `<div class="toast-container">` for notifications
- `initParticles()`, `observeReveals()`, header scroll listener, back-to-top button + JS
- Back-to-top button: `<button class="back-to-top" id="backToTop">`

### Page-Specific Additions
- **contact.html:** Add form submit handler that calls `showToast()` and `reset()`
- **index.html:** Cart sidebar, product modal, search, category filter, sort dropdown, wishlist — all the interactive e-commerce features

## Hosting

After conversion, serve with:
```bash
python3 -m http.server 8080
```

## Public Exposure

Cloudflare Tunnel (reliable in Termux):
```bash
cloudflared tunnel --url http://localhost:8080 > /tmp/cf.log 2>&1
sleep 8
grep -o 'https://.*\.trycloudflare\.com' /tmp/cf.log
```

## Stats from This Session
- 10 pages generated from 10 Jinja2 templates
- ~40KB per page (CSS-heavy due to inline styles)
- All pages return HTTP 200 after generation
