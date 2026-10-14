# Substack API guide — export newsletters to Markdown for RAG & LLMs

Substack has no official public API, but every publication (including custom domains) serves the same
**public JSON endpoints** its web app uses. This guide documents them and shows how to turn posts into
clean Markdown and RAG chunks. Verified October 2026.

> **Want it without code?** [Substack to Markdown on Apify](https://apify.com/mersetti/substack-to-markdown) exports
> posts as clean Markdown/text/HTML with metadata, RAG chunks, date/keyword filters and new-post monitoring —
> paywall-safe.

---

## Endpoints

Base URL is the publication origin: `https://{name}.substack.com` **or** its custom domain
(e.g. `https://www.lennysnewsletter.com`).

### Archive (list of posts, newest first)
```bash
curl "https://www.lennysnewsletter.com/api/v1/archive?sort=new&offset=0&limit=50"
```
- Returns an **array** of posts. Page with `offset` (step 50) until an empty array.
- Useful fields: `id, title, subtitle, slug, post_date, audience, canonical_url, type (newsletter | podcast | thread),
  wordcount, reaction_count, comment_count, publishedBylines[{name, handle}], postTags[{name}], cover_image,
  podcast_url, description, truncated_body_text`.
- `body_html` is **empty** in the archive — fetch each post for content.

### Single post
```bash
curl "https://www.lennysnewsletter.com/api/v1/posts/{slug}"
```
- Same fields plus `body_html`, `updated_at`, `previous_post_slug`, `next_post_slug`.

### Paywalls
- `audience` is `everyone` (free) or `only_paid` / `founding`.
- For paid posts, `body_html` contains only the public preview (roughly the part shown before the paywall).
  **Respect the paywall**: use metadata and `truncated_body_text` only.

### Inputs you'll meet
| Input | Resolve to |
|---|---|
| `astralcodexten` | `https://astralcodexten.substack.com` |
| `https://www.lennysnewsletter.com` | as is (custom domain) |
| `https://x.substack.com/p/some-post` | single post `some-post` |
| `https://substack.com/@user` | a *profile*, not a publication — no archive API |

---

## HTML → clean Markdown

Substack HTML contains UI widgets you don't want in an LLM context:

| Element | What it is | Action |
|---|---|---|
| `p.button-wrapper`, `a.button` | "Subscribe now" / "Share" buttons | drop |
| `div.subscription-widget*` | inline subscribe forms | drop |
| `a.image-link > img` | every image is wrapped in a link to the full-size file | keep only the `img` |
| `div.icon-container`, `svg` | expand / restack icons | drop |
| `figcaption` | image captions | keep as italic text |

With [turndown](https://github.com/mixmark-io/turndown) — note that removal must be an **`addRule`**, because
turndown's built-in paragraph rule wins over `remove()`:

```js
import TurndownService from 'turndown';

const td = new TurndownService({ headingStyle: 'atx', codeBlockStyle: 'fenced', bulletListMarker: '-' });
const JUNK = /subscription-widget|button-wrapper|captioned-button|icon-container|paywall|native-video-embed/;

td.addRule('junk', {
  filter: (n) => ['FORM', 'BUTTON', 'SCRIPT', 'STYLE', 'SVG'].includes(n.nodeName.toUpperCase())
    || JUNK.test(n.getAttribute?.('class') || ''),
  replacement: () => '',
});
td.addRule('imageLink', {
  filter: (n) => n.nodeName === 'A' && /image-link/.test(n.getAttribute('class') || ''),
  replacement: (content) => `\n\n${content.trim()}\n\n`,
});

const post = await (await fetch('https://www.lennysnewsletter.com/api/v1/posts/all-of-the-lenny-and-friends-summit')).json();
console.log(td.turndown(post.body_html));
```

## Chunking for RAG

Split on paragraph boundaries (`\n\n`), keep the latest Markdown heading as chunk context, cap chunks at
1,500–3,000 characters with ~200 characters of overlap. Store `url`, `title`, `postDate` and `heading` as metadata
so answers can cite the source.

## Monitoring new posts

The archive is sorted newest-first: remember the IDs you've seen and stop paging at the first known ID.
That makes a daily "new posts" job cost one request per publication.

## Be respectful

Keep request rates low (a few requests per second at most), cache results, never redistribute paywalled content,
and remember posts are copyrighted by their authors.

---

**Ready-made:** [Substack to Markdown — Apify Actor](https://apify.com/mersetti/substack-to-markdown) ·
**More tools:** [Career Sites Jobs Feed](https://apify.com/mersetti/ats-jobs-feed) ·
[Shopify Products & Price Monitor](https://apify.com/mersetti/shopify-products-monitor)
