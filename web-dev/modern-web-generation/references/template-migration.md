# Jinja2 Template → Static HTML Migration

## Pattern: Converting Flask/Jinja2 templates to standalone pages

### Input
- Flask templates using `{% extends "base.html" %}`, `{% block title %}`, `{% block content %}`, `{% endblock %}`
- Template refs like `{{ url_for('page_name') }}`, `{{ url_for('static', filename='...') }}`
- Brand names embedded in prose

### Extraction Steps

1. **Strip Jinja2 tags** using regex:
   ```python
   import re
   body = re.sub(r"{% extends .*?%}", "", raw)
   body = re.sub(r"{% block title %}.*?{% endblock %}", "", body)
   body = re.sub(r"{% block content %}", "", body)
   body = re.sub(r"{% endblock %}", "", body)
   ```

2. **Map `url_for()` to file paths**:
   ```python
   replace_map = {
       "{{ url_for('index') }}": "index.html",
       "{{ url_for('shop') }}": "index.html#products",
       "{{ url_for('about') }}": "about.html",
       "{{ url_for('careers') }}": "careers.html",
       "{{ url_for('contact') }}": "contact.html",
       "{{ url_for('press') }}": "press.html",
       "{{ url_for('support') }}": "support.html",
       "{{ url_for('privacy') }}": "privacy.html",
       "{{ url_for('terms') }}": "terms.html",
       "{{ url_for('cookies') }}": "cookies.html",
       "{{ url_for('gdpr') }}": "gdpr.html",
       "{{ url_for('cart_page') }}": "index.html",
       "{{ url_for('static', filename='style.css') }}": "style.css",
       "{{ url_for('static', filename='script.js') }}": "script.js",
   }
   ```

3. **Rebrand prose**: Search-and-replace old brand name in body text
4. **Fix email domains**: `@oldbrand.io` → `@newbrand.io`
5. **Wrap in full HTML shell** with inline CSS from main page
6. **Add page-specific JS** (contact form handler, newsletter handler)

### Page Shell Template

```
<!DOCTYPE html>
<html>
<head>
  <title>{page title}</title>
  <link href="https://fonts.googleapis.com/css2?family=Orbitron:...&family=Inter:...&family=JetBrains+Mono:..." rel="stylesheet" />
  <style>
    /* Full CSS from index.html */
  </style>
</head>
<body>
  <!-- Background elements (bg-grid, bg-glow, bg-particles) -->
  <!-- Header with nav links pointing to actual pages -->
  <main>
    {page-specific content}
  </main>
  <!-- Footer with actual page links -->
  <!-- Back-to-top button -->
  <!-- Toast container -->
  <script>
    // Particles, scroll reveal, header scroll, back-to-top, toast, page-specific handlers
  </script>
</body>
</html>
```

### Key Details from Session (Txrao Rebrand)

- Brand renamed from `NEXUS` to `Txrao`
- Email domain: `nexus.io` → `txrao.io`
- 9 pages generated: about, careers, contact, press, support, privacy, terms, cookies, gdpr
- Contact form handler: preventDefault → showToast → reset
- All pages verified with `curl -s -o /dev/null -w "%{http_code}"`
- Back-to-top button: add `<button>` before toast container, JS already handles it
