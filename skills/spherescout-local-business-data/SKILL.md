---
name: spherescout-local-business-data
description: Search and export local-business contacts through SphereScout. Use when the user needs businesses filtered by category, country, region, city, email, phone, or website.
license: MIT
metadata:
  author: SphereScout
  version: "1.0.2"
---

# SphereScout — Local Business Data Skill

Find local businesses by category and geography, preview market size for free, then export verified business contacts from SphereScout's database.

## Requirements
The agent needs internet access and a SphereScout API key for authenticated previews and exports.

## API Key
Ask the user for their SphereScout API key before browsing private previews or exporting. Store it as `SPHERESCOUT_API_KEY` and send it as:

```http
Authorization: Token $SPHERESCOUT_API_KEY
```

Users can create an API key at https://www.spherescout.io/dashboard.

## Connectivity
Send a standard browser `User-Agent` header on every request, for example a recent Chrome desktop UA string. Requests using a default HTTP-library user agent (`curl`, raw `requests`, etc.) may be blocked with `403 Forbidden` by the edge network before they ever reach the API. If every call returns 403 regardless of the endpoint or auth header, this is almost always the cause.

## Compatibility
This skill works with any AI agent that can read Markdown instructions and make authenticated HTTP requests, including Claude, Codex, Cursor, Windsurf, Hermes, Claw, ChatGPT tool workflows, LangChain or LlamaIndex agents, and custom Python or Node agents.

## Workflow
1. Resolve the category with `GET /api/categories/?q=<category>&limit=20`. Translate non-English category words to English first. Use `match_rank`, `matched_field`, and `matched_value` to choose or clarify. Do not load the full catalog unless the filtered search returns no useful match.
2. Resolve the country with `GET /api/locations/countries/` or a known ISO alpha-2 code. Always cross-check the target country against that list before calling `locations/search/`. An unsupported country returns the same empty `{"count":0,"locations":[]}` shape as a genuine no-match on a valid country, so a location search alone cannot tell you which happened — if the country isn't in the list, tell the user directly that SphereScout doesn't currently cover it instead of running a location search first.
3. Resolve a city, region, or county with `GET /api/locations/search/?q=<place>&country=<alpha2>`. Prefer the narrowest level that matches the user's named place; only use a broader parent when it is genuinely coextensive with the match (see Location Lookup).
4. Preview with `GET /api/companies/`. This returns counts plus samples and does not expose email addresses.
5. Ask for explicit confirmation before `GET /api/download-csv/`. Exports deduct credits, one credit per exported row.
6. Poll `GET /api/download-status/<search_id>/` until its `status` field is the lowercase string `completed` or `failed` (not `COMPLETED`/`FAILED`). Small exports often complete within a few seconds. Once complete, fetch `GET /api/download-completed-csv/<search_id>/`, which returns a time-limited signed `download_url` to fetch, not the file itself.

## Category Lookup
Use the filtered category endpoint first:

1. Translate the user's category term to English before searching. SphereScout category names are indexed in English, but the API also searches localized names when present.
2. Call `GET /api/categories/?q=<term>&limit=20`.
3. Prefer the lowest `match_rank`, but a low rank is not sufficient on its own — check that `matched_value` is plausibly related to what the user actually asked for before treating it as a candidate. A substring match like "yak" inside "Yakatabune" or "Yakiniku" can score the same rank as a real match while being completely unrelated. Use `matched_field` and `matched_value` to explain or disambiguate the match.
4. If an exact match also has many related matches, treat the exact match as an umbrella category: use that `id`, but tell the user they can narrow if they want.
5. If there is no match, retry with a broader synonym. Example: retry `roofer` as `roofing`.
6. If the term is ambiguous after a retry, ask one short clarification question.
7. Only call unfiltered `GET /api/categories/` as a fallback for unusual cases, and do not paste the full catalog into the conversation.

Language notes:
- French `entrepreneur` usually means contractor; search for `contractor`.
- French `artisan` is too broad; ask for the trade, such as plumber, electrician, roofer, carpenter, or painter.

## Request Map
- Base URL: `https://api.spherescout.io`
- Category: `category_id` in the skill maps to API query param `category`.
- Country: `country` maps to API query param `countries`, e.g. `countries=FR`.
- Locations: `state_id` maps to `level1_location`; `city_id` maps to `level2_location`; `district_id` maps to `level3_location`.
- Narrowing a location search: pass one or more `states=<level1_id>` or `counties=<level2_id>` params (repeatable) on `GET /api/locations/search/` to restrict results to a known parent. These take IDs, not names — do not append a state or country name into the `q` text string; `q=Austin, Texas` and similar combined strings return zero results.
- Browsing without a text query: `GET /api/locations/level1/?country=<alpha2>` lists every state/region in a country; `GET /api/locations/level2/?country=<alpha2>&states=<level1_id>` lists every county/department within a given state; `GET /api/locations/level3/?country=<alpha2>&counties=<level2_id>` lists every city within a given county. All three also accept `ids=<id>` (repeatable) for direct batch lookup by ID. Unlike `GET /api/locations/search/`, these are not fuzzy-matched and are not capped at 10 rows per type — use them whenever the user's request is itself an enumeration ("every county in California"), or whenever you need certainty that no same-named row was cut off by the search endpoint's cap.
- Filters: `has_email=true` maps to `email=true`; `has_phone=true` maps to `phone_number=true`; `has_website=true` maps to `website=true`; `main_activity_only=true` maps to `main_activity_only=true`.
- Export format: pass `export_format=csv` or `export_format=excel`.

## Location Lookup
Prefer the narrowest geography that matches the user's named place; only broaden to a parent when it is genuinely coextensive with that place, not merely to maximize coverage:

Typical levels:
- `level1`: state, region, province, or equivalent.
- `level2`: county, department, metropolitan area, or equivalent parent area.
- `level3`: city, commune, town, district, arrondissement, postal zone, or neighborhood-like child area.

1. Call `GET /api/locations/search/?q=<place>&country=<alpha2>`. Use the bare place name only; do not append a state, county, or country to the text query.
2. Each result only carries a `type` field (`level1`/`level2`/`level3`), never a query-param hint — map it yourself: `level1` to `level1_location`, `level2` to `level2_location`, and `level3` to `level3_location`.
3. `type` is a fixed, country-independent ladder: `level1` is always the state/region tier, `level2` is always the county/department tier, `level3` is always the city/commune tier. Two results with the same `name` but different `type` are almost always two genuinely different real places at different administrative tiers, not duplicates — for example a county and a city can share a name by historical coincidence (an "Austin County" distinct from the city of Austin). Pick the `type` that matches what the user is actually asking for; default to `level3` (a city) unless they explicitly asked for a broader state, region, county, or department.
4. Only choose a broader `level1`/`level2` parent over a narrower `level3` match when they are genuinely coextensive with the place the user means, meaning the parent and child cover the same population, such as a city-state or city-emirate like Berlin or Dubai where parent and child return the same count. Do not broaden by default just to maximize coverage.
5. Use a smaller `level3_location` only when the user explicitly asks for that city, district, arrondissement, postal zone, or neighborhood.
6. Whenever a state, region, or county is known — whether the user named it upfront in the same message or supplied it after a clarification question — narrow immediately: find its `level1.id` (or `level2.id`) among the bare-name results and re-call `GET /api/locations/search/?q=<place>&country=<alpha2>&states=<level1_id>` (or `&counties=<level2_id>`) before doing anything else with the results. Do not skip this narrowing step just because the state was given upfront rather than asked for — the unfiltered result list can include an unrelated same-named place (such as a county) that happens to sort first and superficially looks like a match. This can still return more than one row for the same state — for example a real city can straddle two counties — so re-apply rule 3 to the narrowed results rather than assuming a single match.
7. If the bare-name results span multiple different states/regions and the user gave no state/region/context at all, do not preview with the first result. Ask one short clarification question first, for example: "Which Austin do you mean — which state?" Once they answer, apply rule 6's narrowing technique using that answer. Two extra cases to watch for, both confirmed against live data:
   - If one of the results is itself a `level1` exact match (the place name IS a state/region, like "Washington"), that is a legitimate candidate in its own right, not just "more context needed." Ask a first-branch question distinguishing the state/region itself from a same-named place inside another state, for example: "Do you mean the state of Washington itself, or a place named Washington within another state?"
   - The visible candidate list from the bare-name search is never guaranteed complete, because `level1`/`level2`/`level3` results are each capped at 10 rows (rule 10) — a country with more than 10 states/regions can cut off a real `level1` match the same way a large county can cut off a `level3` match. A real state, county, or city can exist and simply not appear in that list. If the user names a place that isn't among the visible candidates, do not treat that as "no match there" — resolve the parent id by name (`GET /api/locations/level1/?country=<alpha2>` or `level2/`) and re-query with `states=<level1_id>` (or `counties=<level2_id>`) per rule 6 anyway.
8. Never re-query `q` with a combined "place, state" string such as `Austin, Texas` or `Austin Texas` — this returns zero results. Narrowing always uses the `states=`/`counties=` ID params from rule 6, never by editing the text query.
9. If the search returns two or more rows with the same `name` **and the same `type`** but different IDs, that is the real duplicate case: do not assume they are interchangeable, their underlying data can differ substantially. Previews never consume credits, so run one for each candidate and prefer the one with a non-trivial count, or surface both to the user if counts are close.
10. If the request is itself an enumeration — "every county in California," "each city in Travis County" — do not try to answer it with `GET /api/locations/search/`; it needs a text query and caps results at 10 per type, so it cannot return a complete list. Resolve the parent (state or county) once, then call `GET /api/locations/level2/?country=<alpha2>&states=<level1_id>` or `GET /api/locations/level3/?country=<alpha2>&counties=<level2_id>` to get the full, uncapped child list, and preview or export per child as needed.
11. A parent geography's count always includes all of its children's counts — a county's total already contains every city inside it. If a user explicitly asks to see a city and its wider county together (for example "check the wider county too if the city is too small"), never sum the two numbers. Report them as "N across the county overall, of which M are within the city itself."

## Preview Example
```http
GET /api/companies/?paginate=true&page=1&category=<category_id>&countries=FR&level3_location=<lyon_id>&email=true
Authorization: Token $SPHERESCOUT_API_KEY
```

## Export Example
```http
GET /api/download-csv/?export_format=csv&category=<category_id>&countries=FR&level3_location=<lyon_id>&email=true
Authorization: Token $SPHERESCOUT_API_KEY
```

## Resolution Example
User asks: `Find plumbers in Lyon with emails.`

1. Search `GET /api/categories/?q=plumber&limit=20`; use the returned `id` for the best exact match.
2. Search locations with `GET /api/locations/search/?q=Lyon&country=FR`; choose the Lyon result and map its returned parameter, for example `level3_location=<lyon_id>`.
3. Preview:

```http
GET /api/companies/?paginate=true&page=1&category=<plumber_category_id>&countries=FR&level3_location=<lyon_id>&email=true
Authorization: Token $SPHERESCOUT_API_KEY
```

Show the count before offering an export.

## US Resolution Example
User asks: `Find dentists in Los Angeles with phone numbers.`

1. Search `GET /api/categories/?q=dentist&limit=20`; use the exact `Dentist` match.
2. Search locations with `GET /api/locations/search/?q=Los%20Angeles&country=US`; if the user means Los Angeles county/metro, use the parent result such as `level2_location=<los_angeles_county_id>`; if they mean the city only, use `level3_location=<los_angeles_city_id>`.
3. Preview:

```http
GET /api/companies/?paginate=true&page=1&category=<dentist_category_id>&countries=US&level2_location=<los_angeles_county_id>&phone_number=true
Authorization: Token $SPHERESCOUT_API_KEY
```

## Field Guidance
- Use `has_email=true` when the user needs outreach-ready records.
- Use `has_phone=true` or `has_website=true` only when the user asks for those fields.
- Use `main_activity_only=true` for strict category matching.
- Prefer the narrowest location level that matches the user's named place; only broaden to a parent when it is genuinely coextensive with that place (see Location Lookup). Never broaden by default to "maximize coverage."
- `page_size` is not read by `/api/companies/` in either mode and has no effect — omit it. Without `paginate=true` the preview sample is always up to 10 rows; with `paginate=true` each page is always 50 rows. Only trust `count`/`totalCount` as the true number of matches.

## Export Rule
Never export before showing the preview count and asking the user to confirm the credit spend.

If `GET /api/download-csv/` returns `403` with `validation_code: insufficient_credits`, no credits were deducted. Tell the user `required_credits` vs `available_credits` in plain language, and suggest narrowing the search — add a location filter, or a `has_email`/`has_phone`/`has_website` filter — rather than retrying the same export.

## Example Prompts
- "Find plumbers in Lyon with emails."
- "Find dentists in Los Angeles with phone numbers."
- "How many dentists are available in France?"
- "Export Italian restaurants in Austin with websites to CSV."
- "Build a county-level prospect list for roofers in California."
