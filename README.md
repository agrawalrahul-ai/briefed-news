# Briefed — Daily News Digest

An automated pipeline that fetches top headlines across categories, summarizes each into a strict 60-word blurb using GPT-4o-mini, and publishes them to a static web page. The whole thing runs twice a day with no manual intervention.

**Live site:** https://newsbrief-ra.netlify.app/
**Data repo:** `agrawalrahul-ai/briefed-news` (GitHub)

## Architecture at a glance

```
n8n (schedule 6am/6pm) → GNews API → GPT-4o-mini summarizer → news.json
                                                                   │
                                                          commit via GitHub API
                                                                   │
                                                                   ▼
                                          GitHub repo: agrawalrahul-ai/briefed-news
                                                                   │
                              ┌────────────────────────────────────┴───────────────────────────┐
                              ▼                                                                  ▼
                 Netlify (builds/serves index.html)                          Browser fetches news.json directly
                                                                                from raw.githubusercontent.com
```

The key thing to understand: **Netlify only serves `index.html`.** It does not bundle or rebuild around `news.json`. The page fetches the news data client-side, directly from GitHub's raw content URL, on every load and every 15 minutes thereafter. So a new Netlify deploy is only needed when `index.html` itself changes — new articles show up automatically the moment n8n commits a fresh `news.json`, independent of Netlify.

---

## 1. n8n workflow (`NewsSummary.json`)

Workflow name: **NewsSummary**. Triggers: `Schedule Trigger` (cron `0 6 * * *` and `0 18 * * *`, i.e. 6am and 6pm) or a manual trigger for testing.

### Step-by-step

1. **Set Config** — hardcodes the category list: `world, business, technology, sports, health, science, entertainment`.
2. **Split Categories** — turns the array into 7 separate items, one per category.
3. **Loop Over Items** (`splitInBatches`, `reset:false`) — loops through the 7 categories one at a time:
   - **loop branch** → `HTTP Request (fetch news)` calls GNews `top-headlines` (`country=in`, `lang=en`, `max=3`) for the current category → `Edit Fields` extracts `body` + `category` → `Wait` (2s, rate-limit buffer) → back into the loop for the next category.
   - **done branch** (fires once all 7 categories are processed, since `reset:false` accumulates results) → passes all 7 categories' raw responses downstream together.
4. **Code (unpack articles)** — parses each response's JSON body and flattens all articles into individual items (title, description, content, url, image, publishedAt, source, category). Up to 21 articles total (3 × 7 categories).
5. **Filter (remove bad articles)** — *currently disabled.* When enabled, drops articles with `title === "[Removed]"` (GNews's placeholder for pulled stories) or an empty `description`.
6. **LLM Sumamrizer** (OpenAI, GPT-4o-mini, temperature 0.3) — system prompt instructs a neutral, exactly-60-word summary; runs once per article.
7. **Shape Article Object** — merges each LLM summary back with its source article's metadata (by matching array position) and computes `wordCount` (naive `split(' ').length`).
8. **Validate Word Count** — keeps only articles where `wordCount <= 60` (not an exact-60 check — see note below).
9. **Aggregate** — merges all surviving articles into one item, `articles: [...]`.
10. **Group by Category** — reshapes the flat list into `{ updatedAt, categories: { world: [...], business: [...], ... } }`.
11. **Prepare JSON File** — `JSON.stringify`s that object (pretty-printed).
12. **Base64 Encode** — base64-encodes the string (required by GitHub's Contents API).
13. **Get File SHA** — fetches the current `news.json`'s blob SHA from GitHub (required to update rather than create a file).
14. **Commit to GitHub** — `PUT`s the new content + SHA to `agrawalrahul-ai/briefed-news/contents/news.json`, overwriting the file in place (same path = update, not a new file).

### Known issues / things to revisit
- **Exact-vs-max word count mismatch:** the LLM prompt says "exactly 60 words," but the filter only enforces `<= 60`, and there's no retry/correction step. Real output has ranged 55–60 words — the prompt slightly overpromises relative to what's enforced.
- **Bad-article filter is disabled**, so `[Removed]`/empty-description articles currently pass through to the LLM unfiltered.
- **Positional pairing** in `Shape Article Object` assumes the LLM output array and article array stay in the same order — holds under normal n8n execution but is a fragile assumption.
- **Hardcoded credentials:** the GNews API key and GitHub token were originally hardcoded directly in node parameters rather than stored as n8n credentials (the `Commit to GitHub` node does this correctly via a stored credential; `Get File SHA` did not). Any exposed token should be rotated, and both nodes should reference the same stored credential.

---

## 2. `index.html`

Static single-page site (vanilla JS, no build step, no framework). Structure:

- **Masthead** — brand name, tagline, current date (`#today-date`, from the visitor's local clock), and last-updated indicator (`#last-updated`).
- **Category nav** — buttons built dynamically from whatever categories exist in `news.json`; clicking one filters the grid client-side.
- **News grid** — card per article: image (or a placeholder glyph if missing), category badge, title, 60-word summary, source, relative time ("2h ago"), and a "Read →" link to the original article.
- **Footer** — static disclaimer + `#footer-updated`, a full last-refresh timestamp.

### Data flow (`loadNews()`)
1. Fetches `https://raw.githubusercontent.com/agrawalrahul-ai/briefed-news/main/news.json`, cache-busted with `?t=<timestamp>`.
2. Flattens `data.categories` into one array, sorted newest-first by `publishedAt`.
3. Builds the category nav (with an "All" tab appended at the end) and defaults the active tab to the first real category (not "All").
4. Updates `#last-updated` and `#footer-updated` from `data.updatedAt`.
5. Re-runs automatically every 15 minutes (`setInterval(loadNews, 15 * 60 * 1000)`), and on error shows a friendly "could not load" state instead of a blank page.

`#last-updated` now displays both date and time (e.g. `Updated Jul 20, 2026, 08:37 AM`) — originally it showed only the time.

---

## 3. `news.json`

Generated entirely by the n8n workflow and committed to the GitHub repo root. Shape:

```json
{
  "updatedAt": "2026-07-20T08:37:14.864Z",
  "categories": {
    "world": [
      {
        "title": "…",
        "url": "…",
        "source": "…",
        "image": "…",
        "publishedAt": "2026-07-19T20:31:40Z",
        "category": "world",
        "summary": "… (~60 words) …",
        "wordCount": 59
      }
    ],
    "business": [ /* ... */ ],
    "technology": [ /* ... */ ],
    "sports": [ /* ... */ ],
    "health": [ /* ... */ ],
    "science": [ /* ... */ ],
    "entertainment": [ /* ... */ ]
  }
}
```

`index.html` expects exactly these field names per article — any change to this shape (e.g. renaming `summary` or dropping `category`) requires a matching change in `index.html`'s `makeCard()` and `processData()` functions.

---

## 4. Deployment process

Two independent surfaces, updated by two independent mechanisms:

### Content updates (`news.json`) — fully automatic
1. n8n runs on schedule (6am/6pm) → commits a new `news.json` to `agrawalrahul-ai/briefed-news` via the GitHub Contents API.
2. No further action needed. The live site's browser-side fetch picks up the new file directly from `raw.githubusercontent.com` on next page load or within its 15-minute polling interval. Netlify does not need to rebuild for this.

### Site code updates (`index.html` or other static files) — manual
1. Upload the changed file to the **same path** in `agrawalrahul-ai/briefed-news` on GitHub (web UI "Add file → Upload files", or `git add`/`commit`/`push`). Same path = GitHub treats it as an update to the existing file, not a new one — filenames are matched by path, and are case-sensitive.
2. Because `newsbrief-ra.netlify.app` is linked to this GitHub repo ("Import from Git"), the push triggers Netlify's build webhook automatically.
3. Check **Netlify dashboard → Site → Deploys** for the new build (labeled with your commit message); once "Published," hard-refresh the live site to confirm.
4. If a push doesn't trigger a deploy, check Netlify's **Build & deploy → Branches** setting matches the branch you pushed to, and confirm there's no `netlify.toml` publish-directory mismatch.

### Credentials note
The n8n workflow holds a GNews API key and a GitHub token. These should live only in n8n's credential store (not hardcoded in node parameters), and any token that has been exposed in a shared/committed file should be rotated immediately.
