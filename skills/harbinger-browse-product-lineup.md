---
name: Browse the Harbinger Motors product lineup and events
description: >-
  Walk the Harbinger Motors product page hierarchy, pull the published spec-sheet and brochure
  PDFs, and read the trade-show calendar, from the public read-only content API. No credentials
  required.
api: openapi/harbinger-pages-api-openapi.yml
operations: [listPages, getPage, listMedia, getMedia, listEvents, getEvent, getOembed]
---

# Browse the Harbinger Motors product lineup and events

Harbinger's medium-duty platform — Class 4-6 electric and plug-in hybrid stripped chassis, the HC
Series low-cab-forward cab chassis, the Sevna cab, the step van, and the Industria auxiliary power
system — plus the trade shows the company exhibits at. 35 pages, 624 media items, 7 events.

Base URL: `https://harbingermotors.com/wp-json`

## Authentication

None. Anonymous HTTPS, read-only.

## Steps

1. **Find the product root and its children** — `listPages`
   `GET /wp/v2/pages?per_page=100&_fields=id,slug,title,link,parent&orderby=title&order=asc`
   The lineup hangs off page id **660** (`/our-products/`). Everything with `parent: 660` is a
   product: `plugin-hybrid-chassis` (503), `electric-chassis` (695), `step-van` (1093),
   `cab-chassis` (1241), `hc-series-cab` (1694), `industria` (1714). Titles carry the site's own
   numeric ordering prefix (`3.3 Step Van`) — strip it for display.

2. **Read a product page** — `getPage`
   `GET /wp/v2/pages/{id}?_fields=id,slug,title,content,excerpt,link,featured_media`
   `content.rendered` is HTML. There is **no structured product entity** on this surface: range,
   GVWR, payload, wheelbase and battery capacity are inside the rendered markup, not fields.

3. **Get the real specifications — they are PDFs** — `listMedia`
   `GET /wp/v2/media?per_page=100&mime_type=application/pdf&_fields=id,title,source_url,date`
   This is where the numbers live: `Harbinger_Product-Lineup_FA.pdf`,
   `HC-Series-Cab-One-Pager.pdf`, `Harbinger_StepVanDocument_UPDATE.pdf`,
   `Harbinger-Power-System-One-Pager.pdf`, `Telematics_1_Pager_FA.pdf`. `source_url` is a direct,
   anonymous download. Expect to parse PDF, not JSON.

4. **Pull imagery at a known size** — `getMedia`
   `GET /wp/v2/media/{id}?_fields=id,title,alt_text,source_url,media_details`
   `media_details.sizes` enumerates every generated variant with its own `source_url`, width and
   height. Pick a variant rather than the full-size original.

5. **Read the trade-show calendar** — `listEvents`
   `GET /wp/v2/event?per_page=100&_fields=id,slug,title,link,date,content`
   7 entries at capture (ACT Expo 2026, Work Truck Week 2026, Government Fleet Expo, FedEx Forward
   Service Provider Summit and similar). **`date` is the publication timestamp, not the event
   date** — and `acf` comes back as an empty array, so the actual event date, venue and city are
   only inside `content.rendered`. If you need them structured, you have to parse the HTML.

6. **Get a clean one-line product summary** — `getOembed`
   `GET /oembed/1.0/embed?url=<product page url>`
   The `description` field is the tightest marketing summary the surface exposes, without HTML.

## Things that will bite you

- **No product, vehicle or spec entity exists.** Nothing on this surface exposes drivetrain,
  range, GVWR or telematics data as fields. The `/technology/` page markets integrated telematics,
  ELD integration and predictive maintenance; there is no public interface to any of it.
- **`acf` is always `[]`.** The custom-field group behind events is registered but not exposed to
  REST, so structured event data is unavailable.
- **Media is the largest collection by far** (624 items across 63 pages at the default page size).
  Always constrain with `mime_type` or `per_page=100` plus `_fields`.
- **`per_page` maximum is 100**; 999 returns 400 `rest_invalid_param`.
- **No rate-limit headers, no caching headers.** Self-throttle; you cannot revalidate.

## Error handling

Match on `code`. `rest_post_invalid_id` (404) for a bad page, event or media id;
`rest_invalid_param` (400) for a bad parameter; `oembed_invalid_url` (404) if the URL passed to the
oEmbed endpoint is not a harbingermotors.com resource. Full catalog in
`errors/harbinger-problem-types.yml`.
