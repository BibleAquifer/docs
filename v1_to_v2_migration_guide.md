# Aquifer API: migrating from v1 to v2

A practical guide for consumers moving off the retired v1 API
(`https://api.aquifer.bible`, `swagger/v1`). Every v1 endpoint a consumer
actually called has a v2 equivalent — this doc is the field-by-field
mapping and the list of behaviour changes you have to account for.

---

## 1. Global changes (apply to every endpoint)

| area | v1 | v2 | what you change |
|---|---|---|---|
| **Field casing** | `camelCase` (`bibleId`, `startChapter`) | `snake_case` (`bible_id`, `start_chapter`) | rename every field and query param |
| **List responses** | bare JSON array | `{ "<name>": [...], "pagination": {...} }` | unwrap: read `body.languages`, `body.bibles`, `body.results`, … |
| **Pagination params** | `?limit=&offset=` | `?page=&per_page=` (default 50, max 100 — **`/resources` caps at 500**, since a catalog listing is small and bounded) | convert `offset` → `page` (`page = offset/limit + 1`); `/resources/updates` keeps `limit`/`offset` |
| **Pagination response** | `totalItemCount` / `returnedItemCount` / `offset` | `pagination: { page, per_page, total, has_more }` | `total` replaces `totalItemCount`; `has_more` replaces the `returned < total` check |
| **Script direction** | `scriptDirection: "LTR"\|"RTL"` | `is_rtl: boolean` | `is_rtl` ⇔ `scriptDirection === "RTL"` |
| **License block** | `licenseInfo { title, copyright{...}, licenses:[...], showAdaptationNotice* }` | `license_info { ... }` — same shape, snake-cased (`show_adaptation_notice_for_english` / `_for_non_english`), **plus `adaptation_notice`** (the HTML the flags gate; v1 never returned it) | rename keys; `licenses` carries only the resource's own language entry |
| **Language on resource/article endpoints** | implied by numeric id | explicit `?language=` (default `eng`) where a v2 route is language-scoped | add `?language=` |
| **Authentication** | `api-key` header, required | same — `api-key` header, required | nothing — see below |

### Identity: numeric ids

v1 used numeric ids everywhere. v2 is string-keyed (`resource_code`,
ISO-639-3 language code, `content_id`) **except** where a v1 consumer has
stored numeric values worth preserving:

- **Bibles** keep their v1 numeric `id` — served on `GET /bibles` and Bible
  rows of `GET /resources`, and accepted by `GET /bibles/{bible_id}/texts`
  (numeric id **or** `resource_code`).
- **Languages** keep their v1 numeric `id` (1–36; source-text languages
  `grc`/`heb`/`arc` = 37/38/39) — served on `GET /languages` and other
  language-keyed responses.
- **Collections (now "resources")** never had a numeric id in v1 and still
  don't — use `resource_code`.
- **Resources/items (now "articles")** are keyed by `content_id` (a
  string), scoped by `(resource_code, language)`. A stored numeric
  `contentId` from v1 **does not map** to anything in v2 — you must
  re-resolve articles by `resource_code` + `language` + `content_id`, or
  find them via passage / search / association responses.

### Terminology (the two nouns swapped)

| concept | v1 term | v2 term |
|---|---|---|
| the container (Berean Standard Bible, Tyndale Bible Dictionary) | **collection** | **resource** — `resources` table, `resource_code` |
| a single item inside it (one dictionary entry, one verse) | **resource** | **article** — `articles` table, `content_id` |

So v1's `/resources/search` and `/resources/{contentId}` are *article*
endpoints in v2 language, while v1's `/resources/collections` and
`/resources/collections/{code}` are *resource* endpoints.

### `resource_code` renames

Seven source repos were renamed upstream; v1 shipped the old codes. v2
accepts either — a stale code resolves transitively and the response
carries `Deprecation: true`, a `Link: rel="successor-version"` header, and
(on article detail) a `deprecation` body block. Update your stored codes
when convenient; they keep working.

| v1 code | v2 `resource_code` |
|---|---|
| `TyndaleBibleDictionary` | `AquiferOpenBibleDictionary` |
| `TyndaleStudyNotes` | `AquiferOpenStudyNotes` |
| `TyndaleStudyNotesBookIntros` | `AquiferOpenStudyNotesBookIntros` |
| `TyndaleStudyNotesBookIntroSummaries` | `AquiferOpenStudyNotesBookIntroSummaries` |
| `TyndaleStudyNotesProfiles` | `AquiferOpenStudyNotesProfiles` |
| `TyndaleStudyNotesThemes` | `AquiferOpenStudyNotesThemes` |
| `CBBTER` | `FIATranslationGuide` |
| `UbsImages` | `UBSImages` (casing fix) |

Browser clients: to read the deprecation headers cross-origin the server
must send `Access-Control-Expose-Headers: Deprecation, Link`. The body
signal on article detail needs no CORS config.

### Authentication

**No change** — v2 uses the same `api-key` header v1 did, on every
request. Your existing v1 key keeps working unchanged; there's nothing to
re-issue or migrate on your end.

What did change (doesn't affect a working request): v1's access-level
`Scope` value (1–5) is not carried over — v2 has one API surface and
identification-only auth: a key never unlocks some resources and not
others. A revoked key gets `401`, same as an unrecognized one.

**New: per-key rate limiting.** Each key has a rate tier (currently `standard`:
25 requests/s, burst 100; `elevated`: 50/s, burst 200 — subject to tuning), and heavier routes
(`/passages/{ref}/tokens`, `/resources/updates`, bulk article lists) cost
more than one request. Responses carry `RateLimit-Limit`,
`RateLimit-Remaining` and `RateLimit-Reset`. The limiter starts in a
log-only mode that rejects nothing; once enforced, an over-limit request gets
`429` with `Retry-After` (seconds) — back off and retry.

---

## 2. Endpoint-by-endpoint

### `GET /languages`

```
v1:  GET /languages                       → [ Language, … ]
v2:  GET /languages                       → { languages: [ … ] }
```

| v1 field | v2 field |
|---|---|
| `id` | `id` |
| `code` | `language` |
| `englishDisplay` | `name` |
| `localizedDisplay` | `localized_name` (null when no native name is available) |
| `scriptDirection` | `is_rtl` |
| — | `resource_count`, `iso639_1`, `word_locale` (new) |

### `GET /resources/collections` and `/resources/collections/{code}`

v1's collection list and collection detail map to:

```
v1:  GET /resources/collections          →  v2:  GET /resources
v1:  GET /resources/collections/{code}   →  v2:  GET /resources/{resource_code}              (language-agnostic; available_languages rollup)
                                                 GET /resources/{resource_code}/{language}   (one language's header)
```

| v1 field | v2 field | note |
|---|---|---|
| `code` | `resource_code` | old codes auto-resolve |
| `displayName` | — | dropped (v1-database collection branding, e.g. `"Study Notes (Tyndale)"`); use `name` |
| `shortName` | `abbreviation` | |
| `sliCategory` | `resource_type` | **same value** (`"Bible Dictionary"`, `"Semantic Lexicon"`, …) |
| `resourceType` | `aquifer_type` | **same value** (`Dictionary`/`Guide`/`StudyNotes`/…) |
| `sliLevel` | — | dropped (not in source data) |
| `licenseInfo` | `license_info` | + `adaptation_notice` — see below |
| `availableLanguages[].{languageId,languageCode,displayName,resourceItemCount}` | `available_languages[].{id,language,display_name,article_count,localization,download_urls,formats}` | fat form only on `/resources/{rc}`; the `/{language}` route returns a bare code list |
| — | `content_type` (`Html`/`Json`), `content_order` (`canonical`/`alphabetical`/`monograph`), `version`, `has_audio`, `has_alignments`, `localization`, `download_urls`, `formats` | new |
| — (language-scoped `/resources/{rc}/{language}` only) | `download_url`, `release_tag` | new — the whole-language release zip URL(s) and the release tag they came from |

**Release downloads and per-file formats** (new) — a resource's
release carries more than one zip per language: a combined archive and,
per format (`json`/`md`/`pdf`/`docx`/`alignments`), a format-specific zip.
`download_urls` is `{ kind: [ { url, size_bytes } ] }` keyed by format
(`all` for the combined archive); `formats` is the sorted list of
downloadable content formats for that language, excluding
`alignments`/`audio`. `?format=<kind>` on `GET /resources/{rc}/{language}`
narrows `download_urls` to that one kind (`{}` if the release has none —
not a 404). `by_book` (canonical-order resources, `/resources/{rc}/{language}`
only) also carries a `files` key per book, drawn from the file inventory
returned by `GET /resources/{rc}/{language}/files` (see §4). None of this
has a v1 analog.

**`license_info.adaptation_notice`** (new) — the per-language HTML notice
that the `show_adaptation_notice_for_english` / `_for_non_english` booleans
gate. v1 never returned it, so a v1 consumer only had the flags. `null` when the source has none (e.g. all
Bibles). Serve it when the relevant flag is `true` for the reader's
language.

**`localization`** (new, non-Bible resources) — how complete a
localization is, as `{ percent, band }` per language, banded `Initial` /
`Early` / `Developing` / `Substantial` / `Near Completion` (0–20 … 81–100)
by `reference_id` coverage of the English source. The metric never
computes `Complete` — that band is asserted, not derived; when set, the
computed `percent` is still shown. `GET /resources/{rc}/{language}` carries
a fuller block (`reference_language`, `computed_at`, and `by_book` for
canonical-order resources).

### `GET /resources/types`

```
v1:  GET /resources/types                                    → [ { type, collections:[…] } ]
v2:  GET /resources/types?group_by=resource_type|aquifer_type → { group_by, types:[ { type, resource_count, resources:[…] } ] }
```

**v2 defaults to grouping by `resource_type`** (the fine-grained
SLI-derived taxonomy — `"Bible Dictionary"`, `"Semantic Lexicon"`,
`"Translation Guide"`, …). Pass `?group_by=aquifer_type` to get v1's
collapsed 5-value grouping (`Dictionary`/`Guide`/`StudyNotes`/`Bible`/…).
Every resource entry carries **both** `resource_type` and `aquifer_type`,
so you can group client-side on whichever you need. Neither field is a
closed enum — new values appear as new resources are published.

Per-resource fields: `resource_code`, `name`, `display_name`,
`resource_type`, `aquifer_type`, `content_type`, `license_info`.

### `GET /languages/available-resources`

```
v1:  GET /languages/available-resources?bookCode=GEN&startChapter=1&…&languageCodes=eng,arb
v2:  GET /passages/{ref}/resource-counts?languages=eng,arb
```

`{ref}` accepts BCV8 (`01001001`), USFM (`GEN 1:1`), OSIS (`Gen.1.1`), and
ranges of each. Response: `{ ref, languages: [ { language, id,
resource_counts: [ { resource_type, count } ] } ] }`.

- `resource_type` is v2's string, not v1's collapsed enum.
- Bible verse articles are excluded (as in v1).
- `?languages=` omitted → every available language.

### `GET /resources/search` — **title search**

v1's own docs: *"Currently only searches against content names (not inside
content)."* v2 matches that exactly.

```
v1:  GET /resources/search?query=Aaron&bookCode=GEN&resourceType=Dictionary&limit=10&offset=0
v2:  GET /resources/search?query=Aaron&book_code=GEN&aquifer_type=Dictionary&per_page=50
```

| v1 param | v2 param | note |
|---|---|---|
| `query` (≥3 chars) | `query` (≥3 chars) | accent/case-folded; last term is prefix-matched (`abr` → `Abraham`) |
| `bookCode` + `startChapter`/`endChapter`/`startVerse`/`endVerse` | `book_code` + `start_chapter`/`end_chapter`/`start_verse`/`end_verse` | omitted end = rest of book |
| `languageId` / `languageCode` | `language` | omitted → search every language partition, results merged by score |
| `resourceType` | `aquifer_type` | v1's collapsed enum |
| — | `resource_type` | **new** — filter on the fine-grained SLI string instead |
| — | `content_order` | **new** — comma list of `canonical`/`alphabetical`/`monograph`. **Defaults to `alphabetical,monograph`**: this is a title search and `canonical` resources title articles by passage ref, not headword, so they are dropped by default. **Pass `content_order=all` for the old v1 behaviour** (search every resource, including `canonical` and unclassified/NULL). |
| `resourceCollectionCode` | `resource_code` | old codes auto-resolve |
| `limit` / `offset` | `per_page` / `page` | |

At least one of `query` / `book_code` / `resource_type` / `aquifer_type` /
`resource_code` is required (400 otherwise).

Response:
```jsonc
{
  "query": "Aaron",
  "results": [
    {
      "content_id": "3263",           // was items[].id (numeric)
      "title": "Aaron",               // was name / localizedName
      "reference_id": "2431",
      "review_level": "Professional", // was reviewLevel
      "media_type": "Text",           // was mediaType
      "language_code": "eng",
      "resource_code": "AquiferOpenBibleDictionary",  // was grouping.collectionCode
      "resource_name": "Aquifer Open Bible Dictionary", // was grouping.collectionTitle / name
      "resource_type": "Bible Dictionary",
      "aquifer_type": "Dictionary",   // was grouping.type
      "content_order": "alphabetical", // canonical | alphabetical | monograph
      "score": -12.83                 // bm25 rank, lower = better (query mode only)
    }
  ],
  "pagination": { "page": 1, "per_page": 50, "total": 7, "has_more": false }
}
```

**Not available:** full-text search over article *bodies*. v1 did not
offer it either.

### `GET /resources/{contentId}` → article detail

```
v1:  GET /resources/3261
     GET /resources/3261/by-language/eng
v2:  GET /resources/{resource_code}/{language}/articles/{content_id}
```

There is **no bare-numeric-id lookup** in v2 and no `by-language` variant —
you need the `(resource_code, language, content_id)` triple. Get it from a
search / passage / association / updates response.

| v1 field | v2 field | note |
|---|---|---|
| `id` | `content_id` (string) + path | |
| `referenceId` | `reference_id` | stable cross-language article id |
| `name` / `localizedName` | `title` | one field; it's the localized title |
| `reviewLevel` / `mediaType` | `review_level` / `media_type` | |
| `content` (`list[str]`, one entry per section) | `content` (one `string`, or a `doc` object) | see below |
| `grouping.type` | parent `resource_type` | |
| `grouping.collectionCode` / `collectionTitle` | `resource_code` / `resource_name` | |
| `grouping.associatedResources` | — moved to `…/associations` (`resource_associations`) | |
| `language.{id,displayName,scriptDirection}` | `language_code` only | thinner |
| — | `acai[]`, `content_hash`, `updated_at`, `index_reference`, `source_file` | new |

**Content format** — `?content_format=html|markdown|json` (default `html`):

- `html` — one string (not v1's section array). Bible references are
  `<data class="bible-ref" data-start-ref="BCV8" data-end-ref="BCV8">`;
  article-to-article references are `<data class="resource-ref"
  data-content-id=… data-resource-code=…>`. (v1's `data-bnType` spans are
  gone — same information, different element.)
- `markdown` — GitHub-flavoured Markdown.
- `json` — a single ProseMirror/TipTap `doc` object. `doc.attrs.dir`
  (`ltr`/`rtl`) carries the language's text direction. It uses its own
  node set rather than TipTap's StarterKit schema.

**Bulk mirroring** — instead of `get_resource` + `get_resource_associations`
+ `get_resource_available_languages` per article, use:

```
GET /resources/{resource_code}/{language}/articles?include=content,associations,localizations,acai&page=1&per_page=100
```

`?include=` omitted → all four; `?include=none` → bare metadata rows;
`?updated_since=<iso>` filters on each article's `updated_at`. A full export is `⌈N/page⌉`
calls.

### `GET /resources/{contentId}/associations`

```
v1:  GET /resources/3261/associations
v2:  GET /resources/{resource_code}/{language}/articles/{content_id}/associations
```

| v1 | v2 | note |
|---|---|---|
| `passageAssociations[].{startBookCode,startChapter,startVerse,endBookCode,endChapter,endVerse}` | `passage_associations[].{start_book,start_book_code,start_chapter,start_verse,end_book,end_book_code,end_chapter,end_verse}` | v2 adds the integer book number alongside the USFM code |
| `resourceAssociations[].{contentId,displayName,referenceId}` | `resource_associations[].{content_id,display_name,reference_id,resource_code,language}` | v2 adds the target resource + language, so you can build the cross-reference without another lookup |

Flat and unpaginated, as in v1.

### `GET /resources/{contentId}/available-languages`

```
v2:  GET /resources/{resource_code}/{language}/articles/{content_id}/available-languages
```

Returns the sibling articles in other languages
(`available_languages[].{language, content_id, title}`).

### `GET /bibles`

```
v1:  GET /bibles?languageCode=eng&hasAudio=true                → [ Bible, … ]
v2:  GET /bibles?language=eng&has_audio=true                    → { language, bibles:[…], pagination }
```

| v1 field | v2 field | note |
|---|---|---|
| `id` | `id` (+ `resource_code`) | stored `bibleId` still works |
| `name` / `abbreviation` | `name` / `abbreviation` | |
| `languageId` | `language_code` | |
| `isLanguageDefault` | `is_language_default` | |
| `hasAudio` | `has_audio` | |
| `hasGreekAlignment` | `has_alignments` | broader — true for any Bible with word alignment (Hebrew OT or Greek NT) |
| `licenseInfo` | `license_info` | |
| `hasAudio` / `hasGreekAlignment` / `isLanguageDefault` filters | `has_audio` / `has_alignments` / `is_language_default` query params | booleans, optional; `languageId` filter → `language` (code) |
| — | `version`, `resource_code` | new |

### `GET /bibles/{bibleId}/texts`

```
v1:  GET /bibles/1/texts?bookCode=GEN&startChapter=1&startVerse=1&endChapter=1&endVerse=2&shouldReturnAudioData=true
v2:  GET /bibles/1/texts?book_code=GEN&start_chapter=1&start_verse=1&end_chapter=1&end_verse=2&should_return_audio_data=true&language=eng
```

Response shape is unchanged from v1 (nested `chapters[] → verses[] →
{number, text}`, `text` is plain, no markup) — only the field casing
differs and you add `?language=`.

| v1 | v2 |
|---|---|
| `bibleId` / `bibleName` / `bibleAbbreviation` | `id` / `bible_name` / `bible_abbreviation` (plus `bible_id` = `resource_code`) |
| `bookName` / `bookCode` | `book_name` / `book_code` |
| `verses[].audioTimestamp` | `verses[].audio_timestamp` (null when a verse has no narration) |
| `chapters[].audio.{mp3,webm}.{url,size}` | same, snake-cased path |

`{bibleId}` accepts the numeric id **or** the `resource_code`. Omitted end
chapter/verse = rest of the book. You can also pass a v2-native `?ref=`
string instead of `book_code` + ints.

**`?refs=` batch mode (new, no v1 analog)** — a comma-separated list of
refs (any notation, up to 200), mutually exclusive with `ref`/`book_code`.
Fetches a discontiguous list of verses (e.g. every reference an ACAI
entity's `key_references` cites) in one call instead of one request per
verse. Response shape differs from single-range mode: flat `verses[]` in
request order (each carrying its own `book_name`/`book_code`, since refs
need not share a book), plus `missing[]` for any requested ref this
Bible/language has no verse for (silently absent, not a per-ref error).
`should_return_audio_data` works the same as single-range mode.

### `GET /resources/updates`

```
v1:  GET /resources/updates?startTimestamp=…&languageCode=fra&resourceCollectionCode=…&limit=1000&offset=0
v2:  GET /resources/updates?start_timestamp=…&language_code=fra&resource_code=…&limit=1000&offset=0
```

Keeps v1's `limit`/`offset` (not `page`/`per_page`).

| v1 field | v2 field | note |
|---|---|---|
| `startTimestamp` / `endTimestamp` | `start_timestamp` / `end_timestamp` | end is exclusive |
| `languageCode` / `resourceCollectionCode` | `language_code` / `resource_code` | old codes resolve |
| `items[].resourceId` (numeric article id) | `items[].content_id` + `resource_code` + `language_code` | v2 articles aren't numerically keyed |
| `items[].updateType` (`New` / `Updated`) | `items[].update_type` (`updated` / `deleted`) | v2 has no "new"; v2 adds tombstones (`deleted`), which v1 could not report |
| `{returnedItemCount, totalItemCount, offset, items}` | same, snake-cased | offset-based, oldest first |

### `GET /bibles/books`

Unchanged in shape (`GET /bibles/books`).

---

## 3. What's gone (not available in v2)

| v1 endpoint / feature | why | do instead |
|---|---|---|
| `GET /resources/{contentId}/by-language/{languageCode}` | no bare-numeric-`contentId` resolution in v2 | `GET /resources/{rc}/{language}/articles/{cid}` |
| `GET /bibles/{bibleId}/alignments/greek` | alignment data is served passage-scoped, not as a whole-Bible dump | `GET /passages/{ref}/tokens?include=alignment` |
| `GET /clients/{cs,java,py,ts}` | generated SDK downloads | — |
| Full-text search over article bodies | never existed in v1 either | title search (`/resources/search`) |
| `sliLevel`, `collectionTitle`, numeric ids for non-Bibles/articles | v1-internal / not in source data | use `resource_type` / `name` / string keys |
| v1's section-array `content` | — | one HTML string, or `?content_format=json` |

---

## 4. New in v2 (no v1 analog)

You don't have to use these, but v2 returns reference ids that point at
them:

- `GET /tokens/{token_id}` — resolve a Greek/Hebrew interlinear token id.
- `GET /acai/*` — the ACAI biblical-entity registry (`/acai/types`,
  `/acai`, `/acai/{id}`, `/acai/{id}/tokens|articles|references|speeches`).
- `GET /passages/{ref}/tokens?bible_id=&language=` — the token stream for a
  passage: every translation token in reading order, punctuation included,
  each with `text` / `skip_space_after` (bools) / `exclude` /
  `alignment_status` (`aligned`\|`non_equivalent`\|`unaligned`\|`punctuation`)
  / `record_id`. Enough on its own to reconstruct the punctuated verse text
  (`token.text + ("" if token.skip_space_after else " ")`). One book
  maximum per request; needs a Bible that has word tokenization —
  against one that doesn't it returns **404** `{tokenized: false}`, so
  check `has_alignments` on the Bible first.
  **`?include=alignment`** additionally returns `alignment[]` — one entry
  per alignment record (source-side morphology, lexicon articles, ACAI
  mentions), plus target-anchored unaligned/non-equivalent entries and
  source-only non-equivalents — cross-referenced from each token's
  `record_id`. Enough together to build a full interlinear in one call.
- `GET /passages/{ref}/articles` — every article touching a passage (the
  keyword-free counterpart to passage-scoped search).
- `GET /resources/{rc}/{language}/articles` — bulk article retrieval with
  `?include=`.
- `GET /resources/{rc}/{language}/files` — per-file download inventory for
  a resource+language: each entry carries `path`, `format`, `mime_type`,
  `size_bytes`, `books` (book scope), `group_key` and `download_url`.
  `?book_code=`, `?format=` and `?group=` narrow the list; all are
  optional.
- `GET /resources/{rc}/{language}/articles/{content_id}/tokens` — for a
  lexicon article, every Greek/Hebrew source token whose alignment record
  points at it (the reverse of `GET /tokens/{token_id}`). Paginated.
- `GET /bibles/{bibleId}/texts?refs=` — batch mode: a discontiguous list
  of refs (any notation, up to 200) in one call instead of one request per
  verse. See §2 above.

New **fields** on endpoints you already call (see §2):

- `license_info.adaptation_notice` — the HTML adaptation notice, per
  language. In the source, exposed by neither v1 endpoint.
- `localization` (`{percent, band}`) — per-language completeness of a
  non-Bible resource against its English source; the scoped
  `/resources/{rc}/{language}` adds `by_book` for canonical-order
  resources.
- `content_order` — `canonical` / `alphabetical` / `monograph`.
- `source_file` (article detail) — `{path, group_key}`, the entry in the
  resource's `.../files` listing that the article comes from. `null` when
  unknown, not an error.
- `download_urls` / `formats` / `download_url` / `release_tag` — release
  and per-format download inventory on `/resources/{rc}` and
  `/resources/{rc}/{language}` (see §2 above); `?format=` filters.
