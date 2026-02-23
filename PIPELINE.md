# WP Pulse — Daily Pipeline Instructions

> This file is read by the cron agent every morning.
> Follow these steps exactly, in order. Do not skip steps.

---

## Step 0 — Setup

- Today's date (Israel time): get from session_status
- Newsletter filename: `newsletters/YYYY-MM-DD.html`
- Repo local path: `/root/.openclaw/workspace/wordpress-news/`
- GitHub remote: `https://github.com/navarroido/wordpress-news`

---

## Step 1 — Load Covered History

Read `/root/.openclaw/workspace/wordpress-news/data/history.json`
Extract the list of already-covered URLs.
**Do not include any of these URLs in today's newsletter.**

---

## Step 2 — Spawn Dex to Gather News

Spawn Dex (model: gpt-4o) with this task:

```
You are Dex, a research agent specializing in WordPress ecosystem news.

TODAY: [insert today's date]

Your job: Find 5 fresh, noteworthy WordPress/Elementor news items from the last 48 hours.

Sources to search:
- wordpress.org/news
- wptavern.com
- elementor.com/blog
- make.wordpress.org/core
- kinsta.com/blog
- wpbeginner.com news
- searchengineland.com (WordPress-related)
- torquemag.io

**Rules:**
- Only include items published in the last 48 hours
- Skip any URL already in history (you will receive the list)
- Each item must have: title, URL, 2-sentence summary in Hebrew, category tag
- Categories: Core Update / Plugin / Security / Industry News / Tutorial / Podcast / Research
- Prefer: releases, major announcements, significant research — avoid minor blog posts

**Already covered URLs (skip these):**
[INSERT history URLs here]

**Output format (JSON array):**
[
  {
    "title": "...",
    "url": "...",
    "summary_he": "...",
    "category": "...",
    "emoji": "..."
  }
]

Return ONLY the JSON array, nothing else.
```

Wait for Dex to return 5 items.
If fewer than 3 items found — still proceed with what's available.

---

## Step 3 — Fetch OG Images

For each item URL, run:
```bash
curl -sL --max-time 8 "[URL]" | grep -o 'og:image[^>]*content="[^"]*"' | head -1
```
Extract the content="..." value as the image URL.
If no OG image found, use: `https://s.w.org/images/home/wordpress-default-ogimage.png`

---

## Step 4 — Spawn Ink to Write Copy

Spawn Ink (model: gpt-4o) with this task:

```
You are Ink, a Hebrew copywriter specializing in tech newsletters.

Write polished Hebrew newsletter copy for these WordPress news items.
Keep it sharp, professional, and readable. Avoid jargon overload.
Each item should have:
- headline_he: punchy Hebrew headline (max 12 words)
- body_he: 2-3 sentences in Hebrew, engaging and informative
- cta_he: call-to-action text (e.g. "קרא עוד", "האזן כאן", "לפרטים נוספים")

Also write a "trend_of_the_week" block:
- title_he: headline for the week's main trend (max 10 words)  
- body_he: 2-3 sentences about the dominant theme across this week's news

Items: [INSERT DEX OUTPUT HERE]

Return ONLY a JSON object with keys matching each item's URL, plus "trend".
```

Wait for Ink to return copy.

---

## Step 5 — Build Newsletter HTML

Use the template below. Replace all [PLACEHOLDERS] with actual data.
Save to: `/root/.openclaw/workspace/wordpress-news/newsletters/YYYY-MM-DD.html`

**Template rules:**
- RTL, Hebrew, Heebo font
- Items 1 and 2: full-width cards with large hero image
- Items 3 and 4: side-by-side half-width cards
- Item 5: horizontal card (image right, text left)  
- Trend box: dark purple gradient at the bottom
- Issue number: count from history.json totalIssues + 1
- Footer: issue number, date, "נוצר על ידי Navarro 🦅"

---

## Step 6 — Update index.html

Read `/root/.openclaw/workspace/wordpress-news/index.html`
Add a new `.issue-card` entry at the TOP of the issues list.
New entry format:
```html
<a href="newsletters/YYYY-MM-DD.html" class="issue-card">
  <div class="issue-left">
    <div class="issue-num">גיליון #[N]</div>
    <div class="issue-title">
      <span class="latest-badge">חדש</span>
      [FIRST HEADLINE] &amp; עוד [N-1] ידיעות
    </div>
    <div class="issue-meta">
      <span class="pill">[DATE IN HEBREW FORMAT]</span>
      <span class="pill">[N] ידיעות</span>
    </div>
  </div>
  <div class="arrow">←</div>
</a>
```
Remove "חדש" badge from the previous issue's entry.

---

## Step 7 — Update history.json

Add all new items to the `covered` array.
Increment `totalIssues` by 1.
Update `lastIssue` to today's date.

---

## Step 8 — Git Commit & Push

```bash
cd /root/.openclaw/workspace/wordpress-news
git add -A
git commit -m "WP Pulse #[N] — [YYYY-MM-DD]"
git push
```

---

## Step 9 — Notify Ido

Send a Telegram message to Ido (channel: telegram) with:
```
🗞️ WP Pulse #[N] — [DATE] מוכן!

[EMOJI] [HEADLINE 1]
[EMOJI] [HEADLINE 2]  
[EMOJI] [HEADLINE 3]
[EMOJI] [HEADLINE 4]
[EMOJI] [HEADLINE 5]

🔗 https://navarroido.github.io/wordpress-news/newsletters/YYYY-MM-DD.html
```

---

## Error Handling

- If Dex finds 0 items: send Ido a message "לא נמצאו ידיעות חדשות היום — מדלג על הגיליון"
- If git push fails: retry once, if still fails — notify Ido
- If OG image fetch fails: use the default WP image (no crash)
- If any agent times out: proceed with what's available, note in commit message
