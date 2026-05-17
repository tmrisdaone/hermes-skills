# Jinja2 → Standalone HTML Conversion

When a project has Flask/Jinja2 templates but needs to be served as static HTML (Python http.server, nginx, etc.):

## Step-by-Step

1. **Read the base template** (`base.html`) to extract the shared layout structure (header, footer, CSS links, JS)
2. **For each content template**, extract:
   - `{% block title %}...{% endblock %}` → page title
   - `{% block content %}...{% endblock %}` → page body HTML
3. **Strip** all Jinja2 tags:
   ```python
   import re
   body = re.sub(r"{% extends.*?%}", "", content)
   body = re.sub(r"{% block content %}", "", body)
   body = re.sub(r"{% endblock %}", "", body)
   ```
4. **Replace Jinja2 URL refs** with static paths:
   ```python
   body = body.replace("{{ url_for('index') }}", "index.html")
   body = body.replace("{{ url_for('shop') }}", "index.html#products")
   body = body.replace("{{ url_for('about') }}", "about.html")
   # ... for all known routes
   ```
5. **Replace hardcoded brand names** (NEXUS → Txrao, domain names, etc.)
6. **Wrap each body** in a complete standalone HTML page with:
   - Same CSS (extract from index.html `<style>...</style>`)
   - Same animated background HTML
   - Same header with working navigation links
   - Same footer with working navigation links
   - Same JS for particles, scroll reveal, toast
7. Add **back-to-top button** to every sub-page
8. Add **page-specific JS** (e.g., contact form handler)

## Key Commands

Extract CSS from main page:
```bash
sed -n 'START,ENDp' index.html  # where START/END are line numbers of <style>...</style>
```

## Navigation Pattern

```
Header:      Shop → index.html#products
             Features → index.html#features
             Reviews → index.html#reviews
             Support → support.html

Footer Shop: Wearables/Audio/Accessories → index.html#products
Footer Co:   About → about.html
             Careers → careers.html
             Press → press.html
             Contact → contact.html
Footer Legal: Privacy → privacy.html
              Terms → terms.html
              Cookies → cookies.html
              GDPR → gdpr.html
```

## JS Functions Every Sub-Page Needs

```javascript
// Particles animation
(function initParticles() { ... })();

// Scroll reveal (IntersectionObserver)
let revealObserver;
function observeReveals() { ... }
document.addEventListener('DOMContentLoaded', observeReveals);

// Header scroll effect
header.classList.toggle('scrolled', window.scrollY > 50);

// Back to top button
backBtn.addEventListener('click', () => window.scrollTo({ top: 0, behavior: 'smooth' }));

// Toast function
function showToast(message, type) { ... }

// Page-specific: contact form, newsletter, etc.
```
