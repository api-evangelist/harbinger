---
name: Read the Harbinger Motors newsroom
description: >-
  Search and retrieve Harbinger Motors' press releases, funding and partnership announcements and
  fleet-operations articles from the public read-only content API. No credentials required.
api: openapi/harbinger-posts-api-openapi.yml
operations: [search, listPosts, getPost, listCategories, getSeoHead]
---

# Read the Harbinger Motors newsroom

34 published items covering serial production of Harbinger's medium-duty electric and plug-in hybrid
platform, funding rounds, OEM and fleet partnerships, and a small set of fleet-operations articles.
Available anonymously as JSON.

Base URL: `https://harbingermotors.com/wp-json`

## Authentication

None. Anonymous HTTPS. The collection answers `Allow: GET` to an OPTIONS request, which is the
server itself stating that GET is the only method available without credentials.

## Steps

1. **Search across every content type** — `search`
   `GET /wp/v2/search?search=<terms>&per_page=20`
   Returns lightweight `{id, title, url, type, subtype}` records across posts, pages and events.
   Branch on `subtype` — `post` for a newsroom item, `page` for a product or policy page, `event`
   for a trade show. `type` is always `post` and is not useful for routing. Each result's
   `_links.self.href` gives the correctly typed URL to fetch next.

2. **Or walk the archive by date** — `listPosts`
   `GET /wp/v2/posts?per_page=100&orderby=date&order=desc&_fields=id,date,slug,title,link,categories`
   `X-WP-Total` was 34 at capture. Narrow with `after=<ISO8601>` / `before=<ISO8601>`, or use
   `modified_after` to pick up edits rather than new posts.

3. **Filter to the announcement stream only** — `listPosts` with `listCategories`
   `GET /wp/v2/categories?_fields=id,name,slug,count` returns exactly three terms:
   Press Release (id 1, 25 posts), News (id 2, 7), Blogs (id 18, 2).
   `GET /wp/v2/posts?categories=1` is the corporate announcement stream on its own.
   Do not bother with tags — the `post_tag` taxonomy is registered but holds zero terms, so a
   tag filter always returns an empty set.

4. **Retrieve the article body** — `getPost`
   `GET /wp/v2/posts/{id}?_fields=id,date,modified,title,content,excerpt,link,categories`
   `content.rendered` is HTML, not markdown or plain text, and `title.rendered` carries HTML
   entities (`&#038;` for `&`, `&#8217;` for a curly apostrophe). Decode before display.

5. **Get structured metadata instead of scraping** — `getSeoHead`
   `GET /yoast/v1/get_head?url=<article url>`
   Returns the rendered head block including the schema.org JSON-LD `@graph`, which carries the
   canonical URL, headline, publication and modification dates and the publisher organization —
   cleaner than parsing `content.rendered`.

## Things that will bite you

- **You cannot resolve an author.** Records carry a numeric `author`, but `/wp/v2/users` returns
  401 `rest_user_cannot_view` anonymously. Use `_embed=1` on the post to get the embedded author
  record instead.
- **`context=edit` is a 401.** Stay on the default `context=view`.
- **`per_page` is capped at 100.** 999 returns 400 `rest_invalid_param` with the bound in
  `data.params.per_page`.
- **No rate-limit headers and no Retry-After.** Nothing tells you when you are close to a limit.
  Keep concurrency low and space requests out; the whole corpus is a few dozen requests.
- **No caching signal.** No `ETag`, `Last-Modified` or `Cache-Control` on API responses, so
  conditional requests are not available — you cannot revalidate cheaply, you can only re-fetch.
- **Responses set a `_fbp` Meta advertising cookie.** Discard cookies in a machine client.
- **`X-Robots-Tag: noindex` on every response.** The provider signals this data is not to be
  indexed as content.

## Error handling

Match on `code`, never on `message`. The full catalog of the 14 codes observed live is in
`errors/harbinger-problem-types.yml`. The ones this flow will hit: `rest_invalid_param` (400),
`rest_forbidden_context` (401), `rest_post_invalid_id` (404), `rest_no_route` (404).

## Nothing here is a Harbinger developer program

Harbinger Motors publishes no API documentation, no developer portal, no SDK, no status page and no
API support channel. This skill was written by API Evangelist against the site's own published
route index. Treat the surface as it is: useful, unversioned as a contract, and liable to change
without notice when the site is upgraded.
