# Fallback News Sources (When web_search / web_extract / browser fail)

When `web_search` (HTTP 4xx/5xx), `web_extract` (no extract backend), or `browser_navigate` (Camofox not running) are unavailable, use `terminal` + `curl` to source news directly from public APIs and RSS feeds.

## Quick Fetch Commands

### HN Algolia (tech/AI/startup news — always works, no API key)
```bash
curl -s "https://hn.algolia.com/api/v1/search?tags=front_page&hitsPerPage=15"
```
Parses as JSON. Key fields: `hits[].title`, `hits[].url`, `hits[].points`, `hits[].author`.
For story-only search (no comments): add `&tags=story`
For keyword-filtered queries: `&query=AI+artificial+intelligence`

Python parsing one-liner:
```python
data = json.load(sys.stdin)
for hit in data.get('hits', []):
    title = hit.get('title', '')
    url = hit.get('url', hit.get('story_url', ''))
    points = hit.get('points', 0)
    author = hit.get('author', '')
```

### Ars Technica RSS (tech, policy, science, gaming)
```bash
curl -sL "https://feeds.arstechnica.com/arstechnica/index"
```
Parse with regex for `<item>` blocks extracting `<title>` and `<link>`:
```python
items = re.findall(r'<item>.*?<title>(.*?)</title>.*?<link>(.*?)</link>.*?</item>', content, re.DOTALL)
```

### The Verge RSS (tech, AI, gadgets)
```bash
curl -sL "https://www.theverge.com/ai-artificial-intelligence/rss/index.xml"
```
Same `<item>` pattern as Ars Technica.

### Getting Story Summaries (meta description extraction)
Once you have a news article URL, extract its summary without needing to render the full page:
```bash
curl -sL "https://arstechnica.com/tech-policy/2026/05/some-article/" | python3 -c "
import sys, re
c = sys.stdin.read()
m = re.search(r'<meta name=\"description\" content=\"(.*?)\"', c)
if m: print(m.group(1))
"
```
The `<meta name="description">` tag is present on most news sites and gives the article's lede/summary without parsing the full body.

## When to Use Which

| Tool | Best for | Limitation |
|------|----------|------------|
| HN Algolia API | Tech/AI/startup trending stories | Only HN-curated content |
| Ars Technica RSS | Tech policy, science, security | Single source |
| The Verge RSS | Broad tech & AI | Single source |
| Meta description extraction | Story summaries from any URL | Only first ~160 chars |

## Order of Preference
1. Try `web_search` and `web_extract` first (if available)
2. Fall back to HN Algolia for trending tech/AI (most efficient — single JSON response)
3. Supplement with RSS feeds from Ars Technica, The Verge, Reuters, etc.
4. Extract summaries via meta description tags on individual article URLs

## Pitfalls
- Twitter/X links in HN stories often cannot be scraped via curl (JS-rendered). Skip or note the tweet author/handle only.
- Some sites redirect curl to login pages. If `<meta name="description">` is missing, the article probably requires JS rendering — move on.
- HN Algolia rate-limits generously but don't hammer it in a loop. One call per subtask is sufficient.
