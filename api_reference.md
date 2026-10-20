# BibleAquifer API — reference

Reference for every endpoint of the BibleAquifer API: Bible study resources
(dictionaries, study notes, translation guides, lexicons, images and video),
Bible texts with Greek/Hebrew word alignment, and the ACAI registry of
people, places and other entities in the Bible.

## Conventions

- **Format:** JSON over HTTPS. Field and query-parameter names are
  `snake_case`. An interactive explorer is served at `/docs` and the OpenAPI
  document at `/openapi.json`.
- **Contract:** the field lists in this document describe each response.
  Fields documented as `null`-able are always present; fields documented as
  "absent" appear only under the stated condition.
- **Pagination**, where an endpoint has it: `{page, per_page, total,
  has_more}`, default `per_page=50`, maximum `100`. Exceptions:
  - `GET /resources` allows up to **500**
    per page; the catalog is small, fixed-shape metadata.
  - `GET /resources/updates` uses `limit`/`offset`, because it is a
    poll-and-checkpoint change feed rather than a browsable list.
- **Passage references (`{ref}`)** are accepted in three notations wherever
  a path or query parameter takes one, plus ranges of each:
  - **BCV8** — an 8-digit `BBCCCVVV` (e.g. `01001001` = Genesis 1:1).
    Ranges: `01001001-01001003`.
  - **USFM** — `"GEN 1:1"`; ranges `"GEN 1:1-3"`.
  - **OSIS** — `"Gen.1.1"`; ranges `"Gen.1.1-Gen.1.3"`.
  - **Whole chapters / books** — `"JHN 3"`, `"JHN 3-4"` (chapters 3–4),
    `"GEN"`, `"GEN-EXO"`. Accepted **only** by `/passages/{ref}/articles`,
    `/passages/{ref}/resource-counts`, and the single `?ref=` of
    `/bibles/{id}/texts`; every other endpoint (including
    `/passages/{ref}/tokens` and the `?refs=` batch) rejects them with a
    **400**. A whole-chapter or whole-book reference expands to an open end
    (chapter/verse `999`).
  A descending range (end before start) is a **400**.
  `/passages/{ref}/tokens` and `/bibles/{id}/texts` additionally require the
  reference to stay within **one book** (400 otherwise).
- **`content_format`** (`html` default, `markdown`, `json`) is accepted by
  every endpoint that returns article or verse `content`:
  - `html` — the stored HTML string.
  - `markdown` — GitHub-flavoured Markdown.
  - `json` — a single ProseMirror/TipTap `doc` object; `doc.attrs.dir`
    (`ltr`/`rtl`) reflects the language's script direction. It uses its own
    node set (including generic `div` and `span` nodes that carry the
    source's `class`/`id`), so it is not compatible with TipTap's StarterKit
    schema.
  An unrecognized value is a **400**.
- **Renamed resources:** a few resources have been renamed
  (e.g. `TyndaleBibleDictionary` → `AquiferOpenBibleDictionary`). The old
  `resource_code` keeps working wherever a `resource_code` is accepted: the
  response uses the current code and carries `Deprecation: true` and
  `Link: rel="successor-version"` headers (article detail also adds a
  `deprecation` body block).
- **404 error bodies** for language and resource lookups are structured:
  `{"detail": ..., "available_languages": [...]}` for an unknown language;
  `{"detail": ..., "language": ..., "did_you_mean": [...], ...}` for an
  unknown `resource_code` (with `available_in_language` when you asked for a
  specific `resource_type` such as `"Bible"`).

### Authentication

**Every request needs an `api-key` header:**

```
curl -H "api-key: <your key>" https://.../languages
```

- A valid key identifies the caller and unlocks the whole API; there is no
  per-resource access control. A key's rate-limit tier affects request rate
  only.
- `/docs`, `/redoc` and `/openapi.json` do not need a key.
- **Errors:**
  - `401` — missing, unrecognized, or revoked key.
  - `429` — the key's rate limit is exhausted (when limits are enforced).
    Carries `Retry-After` (seconds).
  - `503` — the key store is unavailable on the server; retry later.
- **Rate-limit headers:** authenticated responses carry `RateLimit-Limit`,
  `RateLimit-Remaining` and `RateLimit-Reset`. Limits are per key, by tier
  (`standard` / `elevated`); heavier routes cost more.

### How the endpoints relate

A few narrower endpoints are views over the same data another endpoint
exposes in bulk. Each is cross-referenced under **See also** below:

- **`GET /passages/{ref}/tokens?include=alignment`** is the alignment
  (interlinear) view; bare `tokens` (no `include`) is the lightweight
  token stream.
- **`GET /resources/{rc}` vs `GET /resources/{rc}/{language}`** are two
  scope levels of one resource: the first is a language-agnostic rollup, the
  second is one language's full detail (`by_book`, `download_urls`, etc.).
- **Article-level `.../associations` and `.../available-languages`** are
  single-article views of the joins that
  `GET /resources/{rc}/{language}/articles?include=associations,localizations`
  returns in bulk.
- **`/acai/{id}` and its sub-resources** (`/tokens`, `/articles`,
  `/references`, `/speeches`) are distinct data sets; `/acai/{id}` itself
  returns counts and small inline blocks and leaves the unbounded lists to
  the sub-resources.
- **`GET /tokens/{token_id}`** is unrelated to `/passages/{ref}/tokens`
  despite the name: it looks up one Greek/Hebrew source-text token, not
  target-language translation tokens.

---

## Endpoint index

### Languages
| Method & path | Purpose |
|---|---|
| `GET /languages` | Every available language, with metadata |

### Resources (catalog)
| Method & path | Purpose |
|---|---|
| `GET /resources` | Paginated resource catalog, optional `?language=` |
| `GET /resources/types` | Resources grouped by type |
| `GET /resources/search` | Title search over articles |
| `GET /resources/updates` | Corpus-wide change feed |
| `GET /resources/{resource_code}` | Language-agnostic resource detail + rollup |
| `GET /resources/{resource_code}/{language}` | One-language resource detail |
| `GET /resources/{resource_code}/{language}/files` | Per-file download inventory |

### Articles
| Method & path | Purpose |
|---|---|
| `GET /resources/{resource_code}/{language}/articles/{content_id}` | Single article detail |
| `GET /resources/{resource_code}/{language}/articles` | Bulk article listing with `?include=` |
| `GET /resources/{resource_code}/{language}/articles/{content_id}/associations` | One article's forward links |
| `GET /resources/{resource_code}/{language}/articles/{content_id}/available-languages` | One article's sibling translations |
| `GET /resources/{resource_code}/{language}/articles/{content_id}/tokens` | Source-text tokens aligned to a lexicon article |

### Passages
| Method & path | Purpose |
|---|---|
| `GET /passages/{ref}/articles` | Every article touching a passage |
| `GET /passages/{ref}/resource-counts` | Per-language, per-type counts touching a passage |
| `GET /passages/{ref}/tokens` | Translation-token stream for a passage, optional `?include=alignment` |

### Bibles
| Method & path | Purpose |
|---|---|
| `GET /bibles` | Bible catalog with filters |
| `GET /bibles/books` | Static canonical book list |
| `GET /bibles/{bible_id}/texts` | Verse text (single range, or `?refs=` batch) |

### ACAI (biblical-entity registry)
| Method & path | Purpose |
|---|---|
| `GET /acai/types` | Per-category entity counts |
| `GET /acai` | Paged entity list |
| `GET /acai/{acai_id}` | Entity detail |
| `GET /acai/{acai_id}/references` | Paged Scripture references |
| `GET /acai/{acai_id}/speeches` | Paged speech/quote token spans |
| `GET /acai/{acai_id}/tokens` | Paged word-level mentions |
| `GET /acai/{acai_id}/articles` | Paged articles tagged with this entity |

### Tokens
| Method & path | Purpose |
|---|---|
| `GET /tokens/{token_id}` | Single Greek/Hebrew source-token lookup |

---

## Languages

### `GET /languages`

Every language the API has content for, with static language metadata.

**Query params:** none.

**Response:**

| Field | Type | Notes |
|---|---|---|
| `languages` | array | |
| `languages[].language` | string | ISO 639-3 code, e.g. `"eng"` |
| `languages[].resource_count` | int | number of resources available in this language |
| `languages[].name` | string \| null | cleaned English display name; `null` if not in the static registry |
| `languages[].localized_name` | string \| null | native name; `null` when none is available |
| `languages[].iso639_1` | string \| null | falls back to the 639-3 code where none exists |
| `languages[].is_rtl` | bool \| null | |
| `languages[].word_locale` | string \| null | Word/COM locale string, e.g. `"en_US"` |
| `languages[].id` | int \| null | stable numeric language id |

```json
{
  "languages": [
    {
      "language": "eng",
      "resource_count": 21,
      "name": "English",
      "localized_name": "English",
      "iso639_1": "en",
      "is_rtl": false,
      "word_locale": "en_US",
      "id": 1
    }
  ]
}
```

**Errors:** none.

---

## Resources (catalog)

### `GET /resources`

Paginated resource catalog.

**Query params:**

| Name | Type | Default | Notes |
|---|---|---|---|
| `language` | string \| null | `null` | omitted → every language |
| `page` | int | `1` | |
| `per_page` | int | `50` | max **500** (catalog cap, not the usual 100) |

**Response:** `{language, resources: [...], pagination}`. Each resource
entry:

| Field | Type | Notes |
|---|---|---|
| `resource_code` | string | |
| `abbreviation` | string \| null | |
| `name` | string | |
| `display_name` | string | falls back to `name` |
| `resource_type` | string | fine-grained type, e.g. `"Bible Dictionary"` |
| `aquifer_type` | string | coarser category: `Dictionary`, `Guide`, `StudyNotes`, `Images` or `Videos` |
| `content_type` | string | `"Html"` or `"Json"` |
| `content_order` | string \| null | `canonical` / `alphabetical` / `monograph` |
| `version` | string \| null | |
| `language_code` | string | |
| `has_audio` | bool | |
| `has_alignments` | bool | true if word-level alignment data exists for the resource: Bible-to-source alignment, or lexical alignment for lexicons |
| `license_info` | object | see shape below |
| `localization` | object \| null | see shape below; `null` if not available |
| `id` | int | **Bibles only** — stable numeric id |
| `is_language_default` | bool | **Bibles only** |

`license_info` shape (reused across every endpoint that returns it):

```json
{
  "title": "...",
  "copyright": {"dates": "...", "holder": {"name": "...", "url": "..."}, "statement": "<html>... | null"},
  "licenses": [{"eng": {"name": "CC BY-SA 4.0", "url": "https://..."}}],
  "show_adaptation_notice_for_english": true,
  "show_adaptation_notice_for_non_english": false,
  "adaptation_notice": "<p>...</p>"
}
```

`copyright.statement` is the resource's optional HTML copyright/attribution
statement (`null` when the resource has none; the UBS lexicons, for
example, carry one).

`localization` shape (non-Bible resources only; `null` for Bibles and for
a resource whose localization has not been assessed):

```json
{
  "percent": 73.2,
  "band": "Substantial",
  "reference_language": "eng",
  "computed_at": "2026-09-01T00:00:00Z"
}
```
Bands: `Initial` (0–20) / `Early` (21–40) / `Developing` (41–60) /
`Substantial` (61–80) / `Near Completion` (81–100) / `Complete`
(an editorial designation for resource+language pairs asserted to be
complete; it replaces the band that would otherwise apply, and `percent`
is still reported).

**Errors:** `404` (unknown language) if `?language=` names
a language the API has no content for.

---

### `GET /resources/types`

Resources grouped by type.

**Query params:**

| Name | Type | Default | Notes |
|---|---|---|---|
| `group_by` | string | `"resource_type"` | or `"aquifer_type"`; anything else → 400 |

By default resources are grouped by the fine-grained `resource_type`.
`group_by=aquifer_type` uses the coarser 5-value category instead, which
merges types (Semantic Lexicon, Study Notes and Bible Dictionary all become
`Dictionary`). Every entry carries both fields either way.

**Response:**

```json
{
  "group_by": "resource_type",
  "types": [
    {
      "type": "Bible Dictionary",
      "resource_count": 3,
      "resources": [
        {
          "resource_code": "AquiferOpenBibleDictionary",
          "name": "...",
          "display_name": "...",
          "resource_type": "Bible Dictionary",
          "aquifer_type": "Dictionary",
          "content_type": "Html",
          "license_info": {"...": "..."}
        }
      ]
    }
  ]
}
```

A resource available in several languages is listed once.

**Errors:** `400` for an unrecognized `group_by`.

---

### `GET /resources/search`

**Title search** over article titles. It does not search article bodies.

**Query params:**

| Name | Type | Default | Notes |
|---|---|---|---|
| `query` | string \| null | `null` | ≥3 chars if given (else 400); accent/case-folded; last term prefix-matched |
| `book_code` | string \| null | `null` | USFM code, e.g. `"GEN"` |
| `start_chapter` / `end_chapter` / `start_verse` / `end_verse` | int \| null | `null` | omitted end = rest of book |
| `language` | string \| null | `null` | omitted → every language, merged by score |
| `resource_type` | string \| null | `null` | fine-grained type string, e.g. `Study Notes` |
| `aquifer_type` | string \| null | `null` | coarser category, e.g. `Dictionary` |
| `resource_code` | string \| null | `null` | old codes auto-resolve |
| `content_order` | string \| null | `"alphabetical,monograph"` | comma list of `canonical`/`alphabetical`/`monograph`, or `all` |
| `page` / `per_page` | int | `1` / `50` | standard pagination |

At least one of `query`/`book_code`/`resource_type`/`aquifer_type`/
`resource_code` is required, or **400**.

`content_order` defaults to excluding `canonical`-ordered resources: a
canonical resource titles its articles by passage ref (`"Genesis 1:1-3"`),
not headword, so a title search against it is mostly noise unless you ask
for it with `content_order=all`.

**Response:**

```json
{
  "query": "Aaron",
  "results": [
    {
      "content_id": "3263",
      "title": "Aaron",
      "reference_id": "2431",
      "review_level": "Professional",
      "media_type": "Text",
      "language_code": "eng",
      "resource_code": "AquiferOpenBibleDictionary",
      "resource_name": "Aquifer Open Bible Dictionary",
      "resource_type": "Bible Dictionary",
      "aquifer_type": "Dictionary",
      "content_order": "alphabetical",
      "score": -12.83
    }
  ],
  "pagination": {"page": 1, "per_page": 50, "total": 7, "has_more": false}
}
```
`score` (relevance; lower is better) is present only when `query` was given.

**Errors:** `400` — missing all filters, `query` too short, unknown
`book_code`, unknown `content_order` token, descending chapter/verse
range. `404` (unknown language) for an unknown `?language=`.

---

### `GET /resources/updates`

Corpus-wide change feed: every article changed since a timestamp, oldest
first, so a poller can checkpoint on the last `timestamp` it saw. Uses
`limit`/`offset`, not `page`/`per_page`.

**Query params:**

| Name | Type | Default | Notes |
|---|---|---|---|
| `start_timestamp` | string | **required** | 400 if missing |
| `end_timestamp` | string \| null | `null` | exclusive |
| `language_code` | string \| null | `null` | |
| `resource_code` | string \| null | `null` | old codes auto-resolve |
| `limit` | int | `1000` | clamped to `[1, 1000]` |
| `offset` | int | `0` | clamped to `>= 0` |

**Response:**

```json
{
  "returned_item_count": 42,
  "total_item_count": 917,
  "offset": 0,
  "items": [
    {
      "update_type": "updated",
      "language_code": "fra",
      "resource_code": "FIATranslationGuide",
      "content_id": "1004",
      "timestamp": "2026-09-01T00:00:00Z"
    }
  ]
}
```
`update_type` is `"updated"` or `"deleted"` (the article was removed).
There is no separate "new" type; `timestamp` is when the API ingested the
change, not when the source was edited. Items carry identifiers only, no
content — fetch each via the article-detail endpoint.

**Errors:** `400` if `start_timestamp` is missing.

---

### `GET /resources/{resource_code}`

Language-agnostic resource detail: core metadata plus a per-language
rollup.

**Query params:**

| Name | Type | Default | Notes |
|---|---|---|---|
| `format` | string \| null | `null` | narrows each `available_languages[].download_urls` to one kind |

**Response:**

| Field | Type | Notes |
|---|---|---|
| `resource_code`, `name`, `display_name`, `abbreviation` | string | |
| `resource_type`, `aquifer_type`, `content_type`, `content_order`, `version` | string \| null | |
| `has_audio`, `has_alignments` | bool | |
| `license_info` | object | from the English edition where present |
| `id` | int \| null | Bibles only |
| `available_languages` | array | the rollup |
| `available_languages[].language` | string | |
| `available_languages[].id` | int \| null | numeric language id |
| `available_languages[].display_name` | string | |
| `available_languages[].article_count` | int | |
| `available_languages[].formats` | array\<string\> | downloadable content formats for that language (`json`/`md`/`pdf`/`docx`/…, excludes `alignments`/`audio`) |
| `available_languages[].download_urls` | object \| null | `{kind: [{url, size_bytes}]}`; `null` if no release resolved |
| `available_languages[].localization` | object \| absent | `{percent, band}` — present only when computed |

There is no top-level `localization` block; use the per-language entries
in `available_languages`.

```json
{
  "resource_code": "AquiferOpenBibleDictionary",
  "name": "Aquifer Open Bible Dictionary",
  "display_name": "Aquifer Open Bible Dictionary",
  "abbreviation": "AOBD",
  "resource_type": "Bible Dictionary",
  "aquifer_type": "Dictionary",
  "content_type": "Html",
  "version": "1.0",
  "content_order": "alphabetical",
  "has_audio": false,
  "has_alignments": false,
  "license_info": {"...": "..."},
  "id": null,
  "available_languages": [
    {
      "language": "eng",
      "id": 1,
      "display_name": "Aquifer Open Bible Dictionary",
      "article_count": 6120,
      "formats": ["docx", "json", "md", "pdf"],
      "download_urls": {"all": [{"url": "https://...", "size_bytes": 4213000}]},
      "localization": {"percent": 100.0, "band": "Complete"}
    }
  ]
}
```

**Errors:** `404` if `resource_code` names no known resource
(with `did_you_mean`).

**See also:** [How the endpoints relate](#how-the-endpoints-relate) — this is scope level 1 of 2 for a resource; `/resources/{resource_code}/{language}` is the other.

---

### `GET /resources/{resource_code}/{language}`

One language's full resource detail — the natural parent of
`.../articles`.

**Query params:**

| Name | Type | Default | Notes |
|---|---|---|---|
| `format` | string \| null | `null` | narrows `download_urls` to one kind (e.g. `?format=pdf`); `{}` if the release has none for that kind, not a 404 |

**Response:** everything `/resources/{resource_code}` has, plus:

| Field | Type | Notes |
|---|---|---|
| `language` | string | |
| `article_count` | int | |
| `localization` | object \| null | full block (percent/band/reference_language/computed_at) — present here, unlike the language-agnostic route |
| `download_url` | array \| null | the whole-language release archive URL(s) (array since a size-split language, e.g. Arabic, has more than one asset); `null` if no usable release yet |
| `release_tag` | string \| null | the release tag `download_url` came from |
| `download_urls` | object \| null | `{kind: [{url, size_bytes}]}`, narrowed by `?format=` if given |
| `formats` | array\<string\> | downloadable content formats for this resource+language |
| `by_book` | array | **canonical-order resources only** — per-book existence + translation stats |
| `available_languages` | array\<string\> | bare language-code list (not the fat rollup) |

`by_book[]` entry:

```json
{
  "book": 1,
  "book_code": "GEN",
  "reference_count": 45,
  "translated_count": 45,
  "percent": 100.0,
  "files": [
    {
      "path": "json/01.content.json",
      "format": "json",
      "mime_type": "application/json",
      "size_bytes": 88213,
      "books": ["GEN"],
      "download_url": "https://raw.githubusercontent.com/BibleAquifer/.../GEN..."
    }
  ],
  "alignments": [],
  "audio": []
}
```
`reference_count`/`translated_count`/`percent` are `null` when no
translation-completion comparison exists (the reference language itself,
or a Bible — book *existence* is still reported for those, just without
the completion stats). `files` (content), `alignments` and `audio` are
pre-filtered to that book, in the same shape `.../files` returns.

**Errors:** `404` (unknown language) for an unknown `?language=`
path segment; `404` (unknown resource) if the resource_code isn't
in that language.

**See also:** Scope-level sibling: `/resources/{resource_code}`.

---

### `GET /resources/{resource_code}/{language}/files`

Per-file download inventory for one resource in one language, optionally
filtered. Files are returned in three classes:

- `files` — the resource's content files (`json`, `md`, `pdf`, `docx`, …).
- `alignments` — word-alignment files: Bible-to-source alignments for
  Bibles, lexical alignments for lexicons. They are organized by book even
  for resources whose content is alphabetical, so they are kept out of
  `files`.
- `audio` — narration timing files (Bibles with audio).

All three keys are always present, possibly empty.

**Query params:**

| Name | Type | Default | Notes |
|---|---|---|---|
| `book_code` | string \| null | `null` | USFM code; unknown code → 400 |
| `format` | string \| null | `null` | one format only (`alignments`/`audio` fill their own key) |
| `group` | string \| null | `null` | one `group_key` only; matched case-insensitively |

**Response:**

```json
{
  "resource_code": "AquiferOpenBibleDictionary",
  "language": "eng",
  "files": [
    {
      "path": "json/001.content.json",
      "format": "json",
      "mime_type": "application/json",
      "size_bytes": 4213,
      "books": [],
      "group_key": "A",
      "download_url": "https://raw.githubusercontent.com/..."
    }
  ],
  "alignments": [],
  "audio": []
}
```
`books` is empty for resources not organized by book. All three lists are **empty**, not an error, when no file inventory is
available for the resource+language; the resource may still exist and have
articles.

`group_key` is **always present** (never `null`) on every file and names
the category the file belongs to:

- **Canonical resources** (organized by book): the book's USFM code.
- **Alphabetical resources:** the language's own grouping label — a letter
  for English or French, a native-script letter for Arabic or Hindi, a
  Hanyu Pinyin syllable for Chinese. Keys are per-language: English has 26
  groups, Hindi about 80 (including a non-alphabetic `"0"` bucket), Chinese
  over 200.
- **Other resources:** the numeric or dotted prefix of the file name (e.g.
  `"1000"`, `"01.01"`), which is the thematic or taxonomic category the
  resource is organized around.

Keys are upper-cased only for languages whose own alphabet is Latin letters
(English, French, …); Pinyin keys stay lower-case. `?group=` matches
case-insensitively. Because keys differ by resource and language, list a
resource's files first to discover its `group_key` values.

**Errors:** `400` for an unknown `book_code`. `404`
(unknown language / unknown resource) for an unknown
language or resource_code.

**See also:** the same `files` shape appears pre-filtered inside each
`by_book[]` entry of `/resources/{resource_code}/{language}`. Note the
naming collision: `?format=` here filters the file list; on the resource
detail routes it instead narrows `download_urls`. To find which file a known article lives in, use article detail's
`source_file` field instead of reasoning about `group_key`.

---

## Articles

### `GET /resources/{resource_code}/{language}/articles/{content_id}`

Single article detail.

**Query params:**

| Name | Type | Default | Notes |
|---|---|---|---|
| `content_format` | string | `"html"` | `html`/`markdown`/`json`; unknown → 400 |

**Response:**

| Field | Type | Notes |
|---|---|---|
| `resource_code` | string | current form of the code |
| `resource_id`, `resource_name`, `resource_abbreviation`, `language_code` | string | identifying fields |
| `content` | string \| object | rendered per `content_format` |
| `content_format` | string | echoes the request |
| `license_info` | object | |
| `acai` | array | `[{acai_id, entity_type, confidence}]` — this article's forward ACAI links |
| `content_id`, `title`, `reference_id`, `media_type`, `review_level`, `index_reference`, `updated_at`, `deleted_at`, `content_hash` | — | article fields; `updated_at` is when the API ingested the latest change, `deleted_at` is non-null for a removed article |
| `source_file` | object \| null | `{path, group_key}` — the entry in `.../files` this article comes from, `group_key` cased as there. `null` when unknown (not an error) |
| `deprecation` | object \| absent | present only when `resource_code` in the request was a stale/renamed code |

**Errors:** `404` — unknown language, unknown `resource_code`
(with `did_you_mean`), or unknown `content_id` (generic "article not
found" — the `content_id` space is too large to usefully suggest
against). `400` for an unrecognized `content_format`.

**See also:** [Bulk mirroring](#get-resourcesresource_codelanguagearticles) for exporting many articles at once; `.../associations` and `.../available-languages` for an article's links without fetching full content.

---

### `GET /resources/{resource_code}/{language}/articles`

Bulk article listing — the batched form of article detail,
`.../associations` and `.../available-languages`.

**Query params:**

| Name | Type | Default | Notes |
|---|---|---|---|
| `updated_since` | string \| null | `null` | ISO timestamp filter on `updated_at` |
| `include` | string \| null | `null` (all four) | comma list of `content`,`associations`,`localizations`,`acai`; `"none"` or empty → bare metadata; unknown token → 400 |
| `content_format` | string | `"html"` | only applies if `content` is included |
| `page` / `per_page` | int | `1` / `50` | standard pagination |

**Response:**

```json
{
  "resource_code": "AquiferOpenBibleDictionary",
  "language": "eng",
  "updated_since": null,
  "include": ["acai", "associations", "content", "localizations"],
  "content_format": "html",
  "license_info": {"...": "..."},
  "articles": [
    {
      "content_id": "3263",
      "title": "Aaron",
      "reference_id": "2431",
      "media_type": "Text",
      "review_level": "Professional",
      "index_reference": "...",
      "updated_at": "2026-09-01T00:00:00Z",
      "meaning": "index-updated",
      "deleted": false,
      "content": "<p>...</p>",
      "passage_associations": [{"...": "..."}],
      "resource_associations": [{"...": "..."}],
      "localizations": [{"...": "..."}],
      "acai": [{"acai_id": "person:Aaron", "entity_type": "person", "confidence": 1.0}]
    }
  ],
  "pagination": {"page": 1, "per_page": 50, "total": 6120, "has_more": true}
}
```
`"meaning"` is always `"index-updated"`: `updated_at` is when the API
ingested the latest change, not when the source was edited. Rows without `content` in `?include=` simply omit the `content`
key; `content_format` in the response is `null` when `content` wasn't
requested.

**Errors:** `400` for an unknown `include` token or unrecognized
`content_format`. `404` for unknown language/resource_code.

**See also:** [How the endpoints relate](#how-the-endpoints-relate) — `.../associations` and `.../available-languages` are single-item views of the `associations`/`localizations` joins this endpoint exposes in bulk.

---

### `GET /resources/{resource_code}/{language}/articles/{content_id}/associations`

One article's forward links: the passages it is about and the other articles it references.

**Query params:** none.

**Response:**

```json
{
  "resource_code": "AquiferOpenBibleDictionary",
  "language": "eng",
  "content_id": "3263",
  "passage_associations": [
    {
      "start_book": 2, "start_book_code": "EXO", "start_chapter": 4, "start_verse": 14,
      "end_book": 2, "end_book_code": "EXO", "end_chapter": 4, "end_verse": 16
    }
  ],
  "resource_associations": [
    {
      "content_id": "108192",
      "reference_id": "2200",
      "display_name": "Old Testament",
      "resource_code": "BiblicaStudyNotesKeyTerms",
      "language": "eng"
    }
  ]
}
```
Each `resource_associations[]` entry names the target by `resource_code`
and `language` as well as `content_id`, enough to build the link without
another lookup.

**Errors:** `404` — unknown language, unknown `resource_code`, or
"article not found" for an unknown `content_id`.

**See also:** the same rows attach to each item of the bulk
`.../articles?include=associations` response.

---

### `GET /resources/{resource_code}/{language}/articles/{content_id}/available-languages`

One article's sibling translations in other languages.

**Query params:** none.

**Response:**

```json
{
  "resource_code": "AquiferOpenBibleDictionary",
  "language": "eng",
  "content_id": "3263",
  "available_languages": [
    {
      "language": "fra",
      "id": 4,
      "language_name": "French",
      "content_id": "3263",
      "title": "Aaron"
    }
  ]
}
```

**Errors:** `404` — unknown language, unknown `resource_code`, or
"article not found" for an unknown `content_id`.

**See also:** the same rows attach to each item of the bulk
`.../articles?include=localizations` response (there under the key
`localizations`; field names differ slightly, so check both if you share a
client model).

---

### `GET /resources/{resource_code}/{language}/articles/{content_id}/tokens`

Source-text (Greek/Hebrew) tokens aligned to this lexicon article: every
source token whose lexical alignment points at it. The path is nested under
the resource because an article's `content_id` is only unique within its
resource.

This is the reverse of `GET /tokens/{token_id}`, which resolves one token to
its morphology.

**Query params:** `page`, `per_page` (same 50-default/100-max policy as
every other paginated list endpoint).

**Response:**

```json
{
  "resource_code": "BDBHebrewLexicon",
  "language": "eng",
  "content_id": "1234",
  "tokens": [
    {
      "token_id": "o01001001003",
      "corpus": "WLCM",
      "book": 1, "chapter": 1, "verse": 1, "word_num": 3, "part": null,
      "text": "רֵאשִׁית", "lemma": "רֵאשִׁית", "strongs": "7225",
      "gloss": "beginning", "gloss2": "beginning", "pos": "noun", "morph": "Ncfsc"
    }
  ],
  "pagination": { "page": 1, "per_page": 50, "total": 1, "has_more": false }
}
```

A token that cannot be resolved to a source-text entry is still returned,
as a bare `{"token_id": ...}` without morphology, rather than dropped.

**Errors:** `404` — unknown language, unknown `resource_code`, or
"article not found" for an unknown `content_id`. A non-lexicon article
(no lexical alignments) is not an error — it returns an empty `tokens: []`.

**See also:** `GET /tokens/{token_id}` for the reverse direction.

---

## Passages

### `GET /passages/{ref}/articles`

Every article whose passage association overlaps `{ref}`.

**Query params:**

| Name | Type | Default | Notes |
|---|---|---|---|
| `language` | string | `"eng"` | |
| `page` / `per_page` | int | `1` / `50` | standard pagination |
| `include_content` | bool | `false` | opt in to inline HTML `content` |
| `content_format` | string | `"html"` | only applies if `include_content=true` |
| `resource_type` | string, **repeatable** | — | `?resource_type=Translation Guide&resource_type=Study Notes`; case-insensitive, matched **exactly and never split on commas** — v2 type names can contain them (`Images, Maps, Videos`) |
| `resource_code` | string, repeatable | — | repeated or comma-separated; old (renamed) codes resolve. Combine with `resource_type` to AND them |
| `exclude_resource_code` | string, repeatable | — | repeated or comma-separated; removes these resources from whatever the other filters select (counts and paging honor it) |
| `order` | string | `"reference"` | `reference` (canonical order) \| `specificity`; anything else → `400` |

`{ref}` here also accepts **whole chapters and books**: `JHN 3`, `Gen.1`,
`JHN 3-4` (chapters 3–4), `GEN`, `GEN-EXO`. A start with no verse reads a
bare-number end as a chapter (`JHN 3:1-4` is still verses 1–4). They expand
to the `999` open-end sentinel, not a versification lookup.

Neither filter changes the default: with both omitted every overlapping
article is returned, **including the Bibles' own verse rows**
(`resource_type` `Bible`). Filter them out with `resource_type`.

`order=specificity` ranks by **density** — the share of an article's
passage rows that fall inside `{ref}` — highest first, ties in canonical
order then `content_id` (so paging is stable). An article cited 4 times,
all in the passage, scores 1.0; one cited 2,643 times scores ~0.0015.
Each row gains a `density` field (0–1). It is a heuristic over citation
counts, most informative for a verse or chapter; a whole-book request
scores nearly everything 1.0.

By default **no article content is returned**, because a broad range can
match tens of thousands of articles. Fetch text per article via the
article-detail endpoint, or opt in with `include_content=true` for a small
page.

**Response:**

```json
{
  "ref": "GEN 1:1-3",
  "language": "eng",
  "content_format": null,
  "articles": [
    {
      "content_id": "3263",
      "title": "Creation",
      "index_reference": "Genesis 1:1-3",
      "resource_code": "AquiferOpenStudyNotes",
      "start_book": 1, "start_chapter": 1, "start_verse": 1,
      "end_book": 1, "end_chapter": 1, "end_verse": 3
    }
  ],
  "pagination": {"page": 1, "per_page": 50, "total": 4, "has_more": false}
}
```
One row per article (its *earliest* overlapping passage range shown),
even if that article cites the requested range more than once.

**Errors:** `400` for an unrecognized ref or a descending range. `404`
(unknown language) for an unknown `?language=`.

---

### `GET /passages/{ref}/resource-counts`

Per-language counts of articles touching `{ref}`, grouped by resource
type.

**Query params:**

| Name | Type | Default | Notes |
|---|---|---|---|
| `languages` | string \| null | `null` | comma list; omitted → every language |

`{ref}` accepts whole chapters/books (`JHN 3`, `GEN`), same as
`/passages/{ref}/articles`.

**Response:**

```json
{
  "ref": "GEN 1:1",
  "languages": [
    {
      "language": "eng",
      "id": 1,
      "resource_counts": [
        {"resource_type": "Bible Dictionary", "count": 3},
        {"resource_type": "Study Notes", "count": 2}
      ]
    }
  ]
}
```
`resource_type` is the fine-grained type string. Bible
verse articles are excluded — this endpoint is about study resources for
a passage, not the Bible text itself. A language with zero matches is
omitted from the array entirely (not included with an empty list).

**Errors:** `400` for an unrecognized ref or descending range.

**See also:** the same matching as `/passages/{ref}/articles`, but counted
and grouped rather than itemized, and across every language rather than
one `?language=`.

---

### `GET /passages/{ref}/tokens`

The translation-token stream for a passage — every token in reading
order, punctuation included — optionally enriched with the alignment
(interlinear) view.

**Path constraint:** `{ref}` must be within a **single book** (400
otherwise, message: *"a passage for /passages/{ref}/tokens must be within
a single book"*).

**Query params:**

| Name | Type | Default | Notes |
|---|---|---|---|
| `bible_id` | string | **required** | a Bible's `resource_code` |
| `language` | string | `"eng"` | |
| `include` | string \| null | `null` | only valid value: `alignment`; unknown value → 400 |

**Response (bare, no `include`):**

```json
{
  "passage": "GEN 1:1",
  "bible_id": "BereanStandardBible",
  "language": "eng",
  "tokens": [
    {
      "token_id": "01001001001",
      "word_num": 1,
      "book": 1, "chapter": 1, "verse": 1,
      "text": "In",
      "skip_space_after": false,
      "exclude": false,
      "alignment_status": "aligned",
      "record_id": 4821,
      "source_verse": "01001001"
    }
  ]
}
```

Per-token fields:

| Field | Type | Notes |
|---|---|---|
| `token_id` | string | bare (unprefixed) target-token id |
| `word_num` | int | |
| `book`, `chapter`, `verse` | int | |
| `text` | string | |
| `skip_space_after` | bool | for punctuated-text reconstruction: `token.text + ("" if skip_space_after else " ")` |
| `exclude` | bool | true for punctuation tokens |
| `alignment_status` | `"aligned"` \| `"non_equivalent"` \| `"unaligned"` \| `"punctuation"` \| `null` | `null` (not `"unaligned"`) when the token's chapter was never aligned at all — no judgment to report |
| `record_id` | int \| null | non-null only when `alignment_status == "aligned"`; cross-references `alignment[]` |
| `source_verse` | string \| null | the source-text verse this token corresponds to, which can differ from the verse in the token's own id |
| `id_range_end`, `source_verse_range_end` | string \| absent | present only when a token's source correspondence spans a verse range |

**With `?include=alignment`, adds:**

```json
{
  "alignment": [
    {
      "record_id": 4821,
      "alignment_status": "aligned",
      "source": [
        {
          "token_id": "o01001001003",
          "corpus": "WLCM",
          "text": "בְּרֵאשִׁית",
          "lemma": "רֵאשִׁית",
          "strongs": "H7225",
          "gloss": "beginning",
          "gloss2": "beginning",
          "pos": "noun",
          "morph": "Ncfsc",
          "is_primary": true
        }
      ],
      "translation": {"bible_id": "BereanStandardBible", "words": [{"token_id": "01001001001", "text": "In", "is_primary": true}]},
      "lexicon_articles": [{"lexicon": "BDBHebrewLexicon", "content_id": "1234", "title": "רֵאשִׁית"}],
      "acai": []
    }
  ]
}
```
`alignment[]` entry fields: `record_id` (int, null for a target-anchored
unaligned/non-equivalent entry or a source-only non-equivalent — there's
no record), `alignment_status` (`"aligned"`/`"unaligned"`/
`"non_equivalent"`), `source` (array, empty for a target-anchored
unaligned/non-equivalent entry — nothing to show on the source side),
`translation` (`{bible_id, words: [{token_id, text, is_primary}]}`, or
**`null`** for a source-only non-equivalent — there's no target
counterpart at all), `lexicon_articles`, `acai` (both arrays, keyed off
the group's source tokens). One entry per alignment record, plus
target-anchored unaligned/non-equivalent entries and source-only
non-equivalents (a source token explicitly marked as having no target
counterpart, e.g. Hebrew's untranslatable direct-object marker אֵת).
Enough on its own,
together with `tokens[]`, to render a full interlinear from one call.

**Errors:**
- `400` — unrecognized ref, descending range, ref spans more than one
  book, unknown `include` value.
- `404` — unknown language; `bible_id` doesn't name a Bible in that
  language; **`{tokenized: false}`** if the Bible has no word
  tokenization at all —
  this check is whole-edition, so a *partially* tokenised Bible still
  200s with an empty `tokens: []` for a passage it hasn't covered.

`{ref}` bounds the request by passage only (one book at most), not by word
position. To work with a sub-verse span, fetch the verse and filter
`tokens[]` client-side by `word_num`, using `record_id` to find each
token's alignment group.

### No alignment data ≠ 404 — two situations that look similar but aren't

- **The Bible has no tokenization at all** (no tokens for any
  chapter) → **404** `{tokenized: false}` as above, for
  *both* bare `tokens` and `?include=alignment` — this check runs before
  `include` is even inspected.
- **The Bible is tokenized, but the requested chapter has no alignment
  data** (never aligned, or aligned for only some chapters) → **200, not 404**, either way:
  - bare `tokens`: every token in that chapter has `alignment_status: null`
    and `record_id: null` (not the string `"unaligned"` — see the field
    note above).
  - `?include=alignment`: same `tokens[]`, plus **`"alignment": []`** — no
    group is synthesized for a chapter that was never checked, since that
    would claim a judgment ("looked, found nothing") that was never made.
  A passage spanning both an aligned and an unaligned chapter mixes real
  `"aligned"`/`"unaligned"`/`"non_equivalent"` entries with `null`/no-group
  entries in the same response.
- **`has_alignments` (on `GET /bibles` / `GET /resources`) is a whole-resource
  flag**, not a per-passage guarantee — a Bible can report
  `has_alignments: true` and still return `"alignment": []` for a specific
  passage that falls in one of its unaligned chapters.

**See also:** [How the endpoints relate](#how-the-endpoints-relate) — `GET /tokens/{token_id}` is unrelated despite the name.

---

## Bibles

### `GET /bibles`

Bible catalog with filters.

**Query params:**

| Name | Type | Default | Notes |
|---|---|---|---|
| `language` | string \| null | `null` | |
| `has_audio` | bool \| null | `null` | |
| `has_alignments` | bool \| null | `null` | |
| `is_language_default` | bool \| null | `null` | |
| `page` / `per_page` | int | `1` / `50` | standard pagination |

**Response:** `{language, bibles: [...], pagination}` — each entry has
the same shape as a Bible-type row from `/resources` (see that section).
Flat, not grouped by translation family — a Bible's `resource_code` is
inherently one-language.

**Errors:** `404` (unknown language) for an unknown `?language=`.

---

### `GET /bibles/books`

The 66 canonical books (Hebrew OT / Greek NT).

**Query params:** none.

**Response:**

```json
{"books": [{"book": 1, "usfm_code": "GEN", "name": "Genesis", "osis_id": "Gen"}]}
```

**Errors:** none.

---

### `GET /bibles/{bible_id}/texts`

Verse text as plain text (nested `chapters[] → verses[]`, no markup), plus
a batch mode.

`{bible_id}` accepts a **numeric id** (the `id` returned by `/bibles`) or a
`resource_code`.

**Single-range mode — query params:**

| Name | Type | Default | Notes |
|---|---|---|---|
| `ref` | string \| null | `null` | v2-native ref string; mutually exclusive with `book_code`. Accepts whole chapters (`JHN 3`, `JHN 3-4`) and books (`GEN`); multi-book (`GEN-EXO`) → `400`. `?refs=` batch is verse-exact |
| `book_code` | string \| null | `null` | USFM code |
| `start_chapter` / `start_verse` / `end_chapter` / `end_verse` | int \| null | `null` | with `book_code`; omitted end = rest of book |
| `language` | string | `"eng"` | |
| `should_return_audio_data` | bool | `false` | |

Provide either `ref` or `book_code` (400 if neither).

**Response (single-range):**

```json
{
  "id": 1,
  "bible_id": "BereanStandardBible",
  "bible_name": "Berean Standard Bible",
  "bible_abbreviation": "BSB",
  "book_name": "Genesis",
  "book_code": "GEN",
  "language": "eng",
  "chapters": [
    {
      "number": 1,
      "audio": {"mp3": {"url": "...", "size": 1211756}, "webm": {"url": "...", "size": 701865}},
      "verses": [
        {"number": 1, "text": "In the beginning God created the heavens and the earth.", "audio_timestamp": {"start": 3, "end": 7.12}}
      ]
    }
  ]
}
```
`audio`/`audio_timestamp` keys are present only when
`should_return_audio_data=true`; `audio_timestamp` is `null` for a verse
with no narration row.

**Batch mode — query params:**

| Name | Type | Default | Notes |
|---|---|---|---|
| `refs` | string | — | comma-separated ref list, any notation; mutually exclusive with `ref`/`book_code` (400 if combined); max **200** refs (400 if exceeded) |
| `language`, `should_return_audio_data` | — | same as above | |

**Response (batch):**

```json
{
  "id": 1,
  "bible_id": "BereanStandardBible",
  "bible_name": "Berean Standard Bible",
  "bible_abbreviation": "BSB",
  "language": "eng",
  "verses": [
    {"ref": "GEN 1:1", "book_name": "Genesis", "book_code": "GEN", "chapter": 1, "verse": 1, "text": "..."}
  ],
  "missing": ["GEN 99:1"]
}
```
Flat `verses[]` in request order (refs need not be contiguous or share a
book), plus `missing[]` for any requested ref this Bible has no verse
for — silently absent, not a per-ref error (a real Bible can lack a verse
for a versification reason). No pagination in either mode.

**Errors:** `400` — neither `ref` nor `book_code` given (single-range
mode); unrecognized ref or descending range; `refs=` combined with
`ref`/`book_code`; empty or >200-item `refs=` list; any individual ref in
`refs=` fails to parse. `404` — unknown language; `bible_id` doesn't
resolve to a Bible resource in that language (structured error,
notes the actual `resource_type` if it names something else).

---

## ACAI (biblical-entity registry)

The ACAI registry is language-agnostic: every `/acai/*` route returns the
same entities regardless of language; only labels and descriptions are
localized, with an English fallback.
Entities can have **non-primary** forms (e.g. `place:Zion` is a
non-primary form of `place:Jerusalem`) — every id-addressed route below
is an **exact-id lookup**: requesting a non-primary id returns only that
form's own slice, never the primary's aggregate.

### `GET /acai/types`

Per-category entity counts.

**Query params:** none.

**Response:**
```json
{"types": [{"type": "person", "count": 2431, "primary_count": 1980}]}
```

**Errors:** none.

---

### `GET /acai`

The paged entity list — **primary-only by default**.

**Query params:**

| Name | Type | Default | Notes |
|---|---|---|---|
| `type` | string \| null | `null` | one of the 8 entity categories |
| `language` | string | `"eng"` | affects only `preferred_label` |
| `include_variants` | bool | `false` | `true` also lists non-primary name forms |
| `page` / `per_page` | int | `1` / `50` | standard pagination |

**Response:**

```json
{
  "type": null,
  "language": "eng",
  "include_variants": false,
  "entities": [
    {
      "acai_id": "person:Aaron",
      "entity_type": "person",
      "primary_id": "person:Aaron",
      "is_primary": true,
      "preferred_label": "Aaron",
      "label_is_fallback": false
    }
  ],
  "pagination": {"page": 1, "per_page": 50, "total": 6644, "has_more": true}
}
```
`label_is_fallback` is true when the localized label came from English
(the requested `language` has no localization entry) — the ACAI
fallback rule.

**Errors:** none (an unrecognized `type` just returns zero results).

**See also:** this is the only endpoint the primary-only default applies
to; every id-addressed ACAI route below is an exact lookup regardless.

---

### `GET /acai/{acai_id}`

Entity detail — flags, attributes, relations, key references,
geocoordinates, place types, localized label, and counts/links to the
unbounded sub-resources (not inlined).

**Query params:**

| Name | Type | Default | Notes |
|---|---|---|---|
| `language` | string | `"eng"` | affects only the `localization` block |

**Response:**

```json
{
  "acai_id": "person:Aaron",
  "primary_id": "person:Aaron",
  "is_primary": true,
  "entity_type": "person",
  "flags": {"non_biblical": false, "only_mentioned_in_apocrypha": false, "mentioned_in_bible": true},
  "attributes": {"gender": "male", "tribe": "Levi", "is_person": true},
  "localization": {
    "language": "eng", "is_fallback": false,
    "preferred_label": "Aaron", "description": "...",
    "alternate_labels": []
  },
  "relations": [{"relation": "sibling", "target_id": "person:Moses", "reason": null}],
  "key_references": [{"ref": "01021001", "book": 1, "chapter": 21, "verse": 1}],
  "geocoordinates": [],
  "place_types": [],
  "reference_count": 321,
  "speech_count": 4,
  "links": {
    "references": "/acai/person:Aaron/references",
    "speeches": "/acai/person:Aaron/speeches",
    "tokens": "/acai/person:Aaron/tokens",
    "articles": "/acai/person:Aaron/articles"
  }
}
```
`attributes` only carries the keys that apply to this entity's type and
are non-null (`gender`/`tribe`/`deity_type`/`group_type`/`group_origin`/
`keyterm_type`/`is_artifact`/`is_person`). `localization` falls back to English when the requested language has no entry. `key_references` are the entity's most significant references, inlined
(a short list); the full paged list is `.../references`.

**Errors:** `404` for an unknown `acai_id`.

---

### `GET /acai/{acai_id}/references`

Paged Scripture references for this exact entity id.

**Query params:** `page` / `per_page` (standard pagination).

**Response:**
```json
{
  "acai_id": "person:Aaron",
  "references": [{"ref": "01021001", "is_key": true, "book": 1, "chapter": 21, "verse": 1}],
  "pagination": {"page": 1, "per_page": 50, "total": 321, "has_more": true}
}
```
`book`/`chapter`/`verse` are `null` for the very few references that are
not a valid BCV8.

**Errors:** `404` for an unknown `acai_id`.

---

### `GET /acai/{acai_id}/speeches`

Paged speech/quote token spans attributed to this entity.

**Query params:** `page` / `per_page` (standard pagination).

**Response:**
```json
{
  "acai_id": "person:Aaron",
  "speeches": [
    {"corpus": "WLCM", "speech_index": 2, "quote_type": "direct", "token_ids": ["o02003001007", "..."]}
  ],
  "pagination": {"page": 1, "per_page": 50, "total": 4, "has_more": false}
}
```
`token_ids` are ACAI-prefixed source-token ids that you can resolve with
`GET /tokens/{token_id}`; the endpoint does not return the quoted text
itself.

**Errors:** `404` for an unknown `acai_id`.

---

### `GET /acai/{acai_id}/tokens`

Paged word-level mentions of this entity in the Greek/Hebrew source text.

**Query params:** `page` / `per_page` (standard pagination).

**Response:**
```json
{
  "acai_id": "person:Aaron",
  "tokens": [
    {"token_id": "o02003001007", "corpus": "WLCM", "link_type": "explicit", "instance_index": 0, "text": "אַהֲרֹן", "lemma": "אַהֲרֹן", "gloss": "Aaron"}
  ],
  "pagination": {"page": 1, "per_page": 50, "total": 189859, "has_more": true}
}
```
`link_type` is `"explicit"` / `"pronominal"` / `"subject"`.

**Errors:** `404` for an unknown `acai_id`.

---

### `GET /acai/{acai_id}/articles`

Paged articles (in one language) whose forward `acai[]` links tag this
exact entity id.

**Query params:**

| Name | Type | Default | Notes |
|---|---|---|---|
| `language` | string | `"eng"` | one language, not aggregated |
| `page` / `per_page` | int | `1` / `50` | standard pagination |
| `min_confidence` | float | — | 0.0–1.0; drops weaker links (out of range → `400`). `total` reflects the filter |

**Response:**
```json
{
  "acai_id": "person:Aaron",
  "articles": [
    {"resource_code": "AquiferOpenBibleDictionary", "language_code": "eng", "content_id": "3263", "title": "Aaron", "confidence": 1.0}
  ],
  "pagination": {"page": 1, "per_page": 50, "total": 321, "has_more": true}
}
```
Ordered by `confidence DESC`, then `resource_code`, then `content_id` (a stable tie-break). Scores are heavily skewed toward low values, so a threshold such as `min_confidence=0.8` trims the list sharply.

**Errors:** `400` for `min_confidence` outside 0.0–1.0; `404` for an unknown `acai_id`; `404`
(unknown language) for an unknown `?language=`.

---

## Tokens

### `GET /tokens/{token_id}`

Resolve a single Greek/Hebrew source-text token id to its interlinear
row.

**Path params:** `token_id` — bare or ACAI-prefixed (`o`/`n` prefix
stripped automatically).

**Response:**
```json
{
  "corpus": "WLCM",
  "book": 1, "chapter": 1, "verse": 1, "word_num": 3, "part": null,
  "text": "בְּרֵאשִׁית",
  "lemma": "רֵאשִׁית",
  "strongs": "H7225",
  "gloss": "beginning",
  "gloss2": "beginning",
  "pos": "noun",
  "morph": "Ncfsc",
  "token_id": "o01001001003"
}
```
`token_id` in the response is always the ACAI-prefixed form, regardless
of how it was requested.

**Errors:** `404` for an unknown `token_id`.

**See also:** unrelated to `/passages/{ref}/tokens` despite the similar
name — different table, corpus, and id space; see
[How the endpoints relate](#how-the-endpoints-relate).
