# andrewzc API — Agent Context

This file is for AI agents making calls to the andrewzc REST API at `https://api.andrewzc.net`. It covers the data model, endpoint selection logic, key patterns, and how to interpret results.

---

## What the database contains

Two MongoDB collections:

- **`pages`** — 342 thematic lists (metros, confluence points, cathedrals, countries, etc.)
- **`entities`** — ~36,000 individual items, each belonging to one page via `list` field

Every entity has a `list` (which page it belongs to) and a `key` (unique within that list). Together, `list` + `key` identify any entity uniquely.

Most entities represent places Andrew has tracked or visited. The `been` boolean indicates whether they have been there. For transit systems, a `section` field gives finer-grained progress: `done` (completed every station/stop), `taken` (ridden), `visited` (been to the city), `want` (planning to visit).

---

## Key fields to know

### Entity fields

| Field         | Type     | Notes                                                                        |
|---------------|----------|------------------------------------------------------------------------------|
| `list`        | string   | Page key this entity belongs to (e.g. `metros`, `confluence`)                |
| `key`         | string   | Unique within list; URL-safe, derived from name                              |
| `name`        | string   | Display name                                                                 |
| `been`        | boolean  | Andrew has visited / completed this                                          |
| `section`     | string   | Progress category for transit: `done`, `taken`, `visited`, `want`           |
| `country`     | string   | ISO 2-letter code (e.g. `FR`, `DE`, `BE`)                                   |
| `countries`   | string[] | For entities spanning multiple countries                                     |
| `city`        | string   | City display name (e.g. `"Paris"`, `"New York, NY"`)                        |
| `icons`       | string   | Emoji string (usually country flag)                                          |
| `link`        | string   | Wikipedia URL                                                                |
| `coords`      | object   | `{ lat, lon }` display coordinates                                           |
| `size`        | number   | Numeric size — station count for transit, length in meters for bridges, etc. |
| `info`        | string   | Supplementary display string, e.g. `"(60%)"` for partial transit completion |
| `state`       | string   | State/province code — used for US and Canadian entities                      |
| `props`       | object   | Structured facts — schema varies by list (see below)                        |
| `notes`       | string[] | Personal notes in Andrew's voice                                             |
| `caption`     | string   | First-person travel log for visited places — use this in answers            |
| `images`      | string[] | Photo filenames — construct URLs as described below                          |
| `wikiSummary` | string   | Wikipedia intro paragraph — only returned by single-entity fetch            |

### The `props` object

Objective facts about an entity. Schema varies by list but is consistent within a list. Examples:

```json
// metros list
{ "stations": 68, "opened": 1900, "lines": 16, "automatic": true }

// countries list
{ "european-union": true, "eurozone": true, "schengen": true, "population": 67000000 }
```

Use `GET /entities/:list/props?filter=...` to query by props with MongoDB filter syntax.

### Transit section values

For the `metros`, `trams`, and `light-rail` lists, `section` is the primary progress indicator:

| section   | meaning                              |
|-----------|--------------------------------------|
| `done`    | Completed every station / stop       |
| `taken`   | Ridden the system but not completed  |
| `visited` | Been to the city but not ridden      |
| `want`    | Planning to visit                    |

Tram lists use `done`, `taken`, `seen` (seen trams in the city).

### Image URLs

Some entities have an `images` array of filenames. Construct URLs as:

- Thumbnail (600px square): `https://images.andrewzc.net/[LIST]/tn/[FILENAME]`
- Full resolution: `https://images.andrewzc.net/[LIST]/[FILENAME]`

Example: `alizava2.jpg` in the `confluence` list →
`https://images.andrewzc.net/confluence/tn/alizava2.jpg`

---

## Endpoint selection guide

Users ask questions in second person ("have you been to...", "what have you done in..."). Translate these into the appropriate API call.

### "Which metro systems have you completed?" / "Which trams have you finished?"
-> `GET /pages/metros` — fetch all, filter on `section: "done"` and relevant country codes.
-> Or: `POST /search {"query": "completed metro systems in Europe"}` — the natural language router will handle it.

### "Have you been to X?"
-> `GET /entities/:list/:key` — check `been` field.
-> If you don't know the key: `GET /entities?name=<n>&list=<list>` first.

### "What confluence points have you visited in France?"
-> `POST /search {"query": "confluence points I visited in France"}` — fastest option.

### "What's near X?"
-> If X is a known entity: `GET /entities/:list/:key/nearby?radius=50`
-> If X is a place name: `GET /entities/nearby?lat=...&lon=...&radius=...` — use your own knowledge for coordinates.

### "What's similar to X?" / "What else is like X?"
-> `GET /entities/:list/:key/similar` — uses Wikipedia embedding similarity.

### "Tell me about X" / "What do you know about X?"
-> `GET /entities/:list/:key` — returns `wikiSummary`, `caption`, `images`, and all fields.

### "What kind of lists do you have?" / "What do you track?"
-> `GET /pages/summaries` — lightweight list of all 342 pages with key, name, description.

### Open-ended / thematic questions
-> `POST /search {"query": "..."}` — uses OpenAI function calling to pick the best strategy. Best for questions that don't map cleanly to a single list + filter.

---

## How keys work

Entity keys are derived from the name: lowercased, spaces → hyphens, diacritics stripped, punctuation removed, "the-" prefix removed.

- "Paris Metro" → `paris-metro`
- "Sao Paulo" → `sao-paulo`
- "O'Hare" → `ohare`
- "The Hague" → `hague`

Some lists use `reference-key` to disambiguate duplicate names by appending a reference value:
- "Springfield, Illinois" → `springfield-illinois`
- "Gare du Nord" (Paris) → `paris-gare-du-nord`

If you're unsure of a key, use `GET /entities?name=<n>&list=<list>` to look it up by name first.

---

## Important data nuances

**`been` vs `section`**: For transit systems, `section` is more informative than `been`. `section: "done"` means every station/stop has been visited; `section: "taken"` means ridden but not finished. Use `section` for transit questions; `been` for everything else.

**Andrew uses they/them pronouns** — use "they" not "he" when referring to Andrew in any generated text.

**`country` vs `countries`**: Most entities have a single `country`. Entities spanning multiple countries use `countries` (array). The `/search` endpoint and `/countries/:code` both handle this automatically.

**`city` vs `reference`**: A few lists (notably German grand unions) use `reference` instead of `city`. If a city filter returns no results, try `reference`.

**`caption` and `images`**: Present on visited entities in `confluence`, `tripoints`, `swimming`, and `heritage` lists. The caption is a first-person travel log — always use it when available. Show up to 3 thumbnail images inline, each linked to the full-resolution original.

**`wikiSummary`**: Only included in single-entity fetches (`GET /entities/:list/:key`). Stripped from list/search results to keep payloads small.

**`complete` flag on pages**: `complete: true` means the list is a closed set (EU countries, Belgian provinces, etc.) — not that Andrew has visited everything on it.

**Transit progress** (approximate, as of early 2026 — fetch live data for current counts):
- 31 metro systems completed (`section: "done"`)
- 60 tram systems completed
- 9 light rail systems completed
- 100 total rail systems completed
- 80.7% of all tram stops in Czechia visited
- 100% completion: Netherlands, USA, and Belgium grand unions
- Close to completing all EU, Eurozone, and NATO countries (4 remaining across all three)

**Visit philosophy** — edge cases affecting how to interpret `been`:
- Japan prefectures: passing through on the Shinkansen does not count; must have meaningful contact
- Entities marked with `*` in `name`: been nearby but not technically visited
- Tripoints where state lines meet international borders: not counted
- Monaco and Gambia are treated as enclave-like

---

## Common API call patterns

```
# Metro systems completed
GET /pages/metros  -> filter results where section == "done"

# Confluence points visited in France
POST /search  {"query": "confluence points I visited in France"}

# Countries in the Schengen area
GET /entities/countries/props?filter={"props.schengen":true}

# Metros with 50+ stations
GET /entities/metros/props?filter={"props.stations":{"$gte":50}}

# Everything visited in Belgium (across all lists)
GET /countries/BE  -> filter results where been == true

# Transit systems started but not finished
GET /pages/metros  -> filter results where section == "taken"
```

---

## Response shape

All list/search results include: `name`, `list`, `key`, `icons`, `link`, `been`, and a `page` sub-object with the page's `name`, `icon`, and `key`. Geo results additionally include `distanceKm`.

Single-entity fetches additionally include: `wikiSummary`, `caption`, `images`, `notes`, `props`, `coords`, `country`, `city`, `section`, and all other stored fields.

Errors: `{ "error": "<code>", "message": "<detail>" }` — codes: `not_found`, `page_not_found`, `bad_request`.

---

## Endpoint quick reference

```
GET  /pages/summaries                        — all page keys + names + descriptions
GET  /pages/:key                             — page metadata + all its entities
GET  /entities?name=<q>[&list=][&limit=]     — name substring search
GET  /entities?search=<q>[&list=][&limit=]   — semantic/vector search
GET  /entities/:list/:key                    — single entity (includes wikiSummary, caption, images)
GET  /entities/:list/:key/similar            — semantically similar entities
GET  /entities/:list/:key/nearby             — geo: nearby a known entity
GET  /entities/:list/props?filter=<json>     — filter by props fields
GET  /entities/nearby?lat=&lon=&radius=      — geo: nearby a coordinate
GET  /countries/:code                        — all entities for a country
GET  /cities/:key                            — all entities for a city
POST /search  { "query": "..." }             — natural language search (best for complex queries)
```
