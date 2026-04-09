#  AI Newsletter Digest — n8n Workflow

> Automatically summarise RSS feeds daily and deliver a clean, categorised briefing to your inbox. No doom-scrolling. Just signal.

![n8n](https://img.shields.io/badge/built%20with-n8n-orange?style=flat-square) 
![AI](https://img.shields.io/badge/AI-OpenRouter%20%2F%20OpenAI%20%2F%20Claude-blue?style=flat-square)
![License](https://img.shields.io/badge/license-MIT-green?style=flat-square)

---

##  What It Produces

A daily HTML email delivered to your inbox at 7:00 AM containing:

- A dark-header banner with today's date and article count
- Articles grouped by category (AI & Tech, Business, Science, World News)
- Each article: linked title + 2-sentence AI summary + author + date
- Footer with source links and unsubscribe

---

##  Workflow Architecture

```
Schedule Trigger (7:00 AM)
        │
        ├──▶ TechCrunch RSS
        ├──▶ Hacker News RSS      ← run in parallel
        └──▶ MIT Tech Review RSS
                │
                ▼
           Merge node (append all feeds)
                │
                ▼
           IF node (published in last 24 hrs?)
             │ Yes              │ No
             ▼                  ▼
          Limit node          Stop (skip old)
             │
             ▼
      ┌─ Loop Over Items ──────────────┐
      │                                │
      │   Basic LLM Chain              │
      │   (summarise in 2 sentences)   │
      │                                │
      └────────────────────────────────┘
                │  (loop done)
                ▼
        Code node 1 — Group by category
                │
                ▼
        Code node 2 — Build HTML email
                │
                ▼
          Gmail node → 📧 your inbox
                │
                ▼
           IF node (highlights?)
             │ Yes              │ No
             ▼                  ▼
      Telegram node           Stop
```

---

##  Nodes Used

| Node | Purpose |
|---|---|
| Schedule Trigger | Fires every day at 7:00 AM |
| RSS Feed (×3) | Fetches TechCrunch, Hacker News, MIT Tech Review |
| Merge | Combines all feed items into one list |
| IF | Filters articles older than 24 hours |
| Limit | Caps the number of articles per run |
| Loop Over Items | Iterates articles one at a time |
| Basic LLM Chain | Sends each article to AI for a 2-sentence summary |
| OpenRouter Chat Model | The underlying AI model (swap for OpenAI or Claude) |
| Code node 1 | Groups articles by category tag |
| Code node 2 | Builds the HTML email string |
| Gmail | Sends the digest to your inbox |
| Telegram (optional) | Posts top highlights to a channel |

---

##  Project Files

```
├── README.md
├── group_by_category.js      # Code node 1 — categorisation logic
└── build_html_email.js       # Code node 2 — HTML email builder
```

### `group_by_category.js`

Reads all items from the loop output. Separates LLM summary items (identified by `text` field) from RSS article items (identified by `isoDate` or `pubDate`). Zips them together by index, resolves each article's category from its RSS `categories[]` tags using a lookup map, and returns a single item with a `categories` array.

**Output shape:**
```json
{
  "categories": [
    {
      "category": "AI & Tech",
      "articles": [
        {
          "title": "...",
          "link": "...",
          "summary": "...",
          "author": "...",
          "publishedAt": "...",
          "tags": ["AI", "startups"]
        }
      ],
      "articleCount": 3
    }
  ]
}
```

### `build_html_email.js`

Reads the `categories` array from the previous node. Builds a complete, inline-styled HTML email string safe for Gmail. Returns `{ html, subject, articleCount, generatedAt }` for the Gmail node to consume.

---

##  Setup Instructions

### 1. Import the workflow

Download the workflow JSON from n8n and import via **File → Import from file** in your n8n instance.

### 2. Add RSS feeds

Add one **RSS Feed** node per source, all connected directly to the Schedule Trigger in parallel. Suggested feeds:

| Source | RSS URL |
|---|---|
| TechCrunch | `https://techcrunch.com/feed/` |
| Hacker News | `https://hnrss.org/frontpage` |
| MIT Tech Review | `https://www.technologyreview.com/feed/` |
| The Verge | `https://www.theverge.com/rss/index.xml` |
| VentureBeat | `https://feeds.feedburner.com/venturebeat/SZYF` |

### 3. Configure the AI model

The **Basic LLM Chain** node uses this system + user prompt:

**System prompt:**
```
You are a concise newsletter assistant. Summarise news articles in exactly 2 sentences. Be factual and direct — no fluff.
```

**User prompt:**
```
Summarise this article in 2 sentences.

Title: {{ $json.title }}
Content: {{ $json.contentSnippet }}
```

Connect an **OpenRouter Chat Model** (or OpenAI / Claude) as the model sub-node.

### 4. Connect your Gmail account

In the **Gmail** node:
- Authenticate via OAuth (click the credential field → Connect → Sign in with Google)
- Set **To** → your email address
- Set **Subject** → `{{ $json.subject }}`
- Set **Email Type** → HTML
- Set **Message** → `{{ $json.html }}`

### 5. Paste the Code nodes

Copy the contents of `group_by_category.js` into **Code node 1** and `build_html_email.js` into **Code node 2**. Both nodes must be set to mode **Run Once for All Items**.

### 6. Activate

Toggle the workflow to **Active**. It will run automatically every morning at 7:00 AM.

---

##  Category Tag Map

Articles are categorised by matching their RSS `categories[]` tags against this lookup table. Add your own entries as needed in `group_by_category.js`:

| Tag (lowercase) | Digest Category |
|---|---|
| `ai`, `artificial intelligence`, `machine learning` | AI & Tech |
| `technology`, `tech`, `software`, `gadgets` | AI & Tech |
| `startups`, `venture`, `funding`, `fintech` | Business |
| `business`, `finance`, `economics`, `enterprise` | Business |
| `science`, `health`, `biotech`, `climate` | Science |
| `politics`, `policy`, `government`, `world` | World News |

Articles whose tags don't match any entry fall into **General**.

---

##  Customisation

**Change the schedule** — open the Schedule Trigger node and adjust the time.

**Add more feeds** — add extra RSS Feed nodes and connect them to the Merge node.

**Add a category** — add a new entry to `TAG_CATEGORY_MAP` in `group_by_category.js` and a matching colour in `CATEGORY_COLORS` in `build_html_email.js`.

**Change the AI model** — swap the OpenRouter Chat Model sub-node for any n8n-supported model (OpenAI GPT-4o, Anthropic Claude, Mistral, etc.).

**Add Telegram highlights** — after the Gmail node, add an IF node checking `articleCount > 0`, then a Telegram node that posts `{{ $json.subject }}` and the top 3 summaries.

---

##  Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Email arrives with 0 articles | HTML builder not connected to group-by-category node | Check node connections; ensure Code node 1 feeds into Code node 2 |
| All articles in "General" | RSS category tags not in the map | Add the tags to `TAG_CATEGORY_MAP` in Code node 1 |
| Summaries missing | LLM Chain output field name mismatch | Check that the `text` field exists on LLM output items |
| Gmail auth error | OAuth token expired | Re-authenticate the Gmail credential in n8n |
| Python runner error | n8n Python runner not configured | Use the JavaScript version (default); see n8n docs to enable Python |
| No output from Code node | `sortedCategories` is empty | Enable "Always Output Data" in node Settings, or check item separation logic |

---

##  License

MIT — free to use, modify, and distribute.

---

##  Credits

Built with [n8n](https://n8n.io) · Powered by [OpenRouter](https://openrouter.ai) · Feeds from TechCrunch, Hacker News, MIT Tech Review
