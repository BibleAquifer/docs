# Lexical Alignments

Lexical alignments map individual words from the original biblical texts to entries in semantic dictionary resources. Each alignment record connects a source token from the Greek or Hebrew text to a specific dictionary article, enabling word-level semantic lookup, contextual dictionary navigation, and cross-referencing between biblical text and reference material.

Lexical alignments are currently available for:

| Resource | Source Text | Source `docid` | Target Scheme |
|---|---|---|---|
| UBS Dictionary of the Greek New Testament | SBLGNT | `SBLGNT` | `ubs-dgnt` |
| UBS Hebrew Dictionary | WLCM | `WLC` | `ubs-dbh` |

## File Organization

Alignment files are located at `/{lang}/alignments/` within each dictionary repository. One file covers one biblical book; the filename number matches standard USFM book numbering (OT books 01–39, NT books 40–66).

```
/{lang}/alignments/
├── 01.alignment.json   (Genesis)
├── 02.alignment.json   (Exodus)
│   ...
├── 39.alignment.json   (Malachi)
├── 40.alignment.json   (Matthew)
│   ...
└── 66.alignment.json   (Revelation)
```

Which books have alignment data is indicated in `/{lang}/metadata.json` under `scripture_burrito.ingredients` (see [Metadata](#metadata) below). Not all biblical books are necessarily covered for every dictionary.

## JSON Structure

Files conform to the Scripture Burrito v0.4 alignment specification.

### Top Level

```json
{
  "format": "alignment",
  "version": "0.4",
  "groups": [...]
}
```

### Group

Each file contains one group.

```json
{
  "type": "source-word-to-lexicon",
  "meta": {
    "creator": "convert-semantic-dictionaries",
    "created": "2026-05-12"
  },
  "documents": [
    { "scheme": "BCVW|token-string",  "docid": "SBLGNT" },
    { "scheme": "ubs-dgnt",           "docid": "UBSGreekNTDictionary" }
  ],
  "roles": ["source", "target"],
  "records": [...]
}
```

- **`type`**: Always `"source-word-to-lexicon"` for lexical alignments
- **`documents[0]`**: The source (original language) text
- **`documents[1]`**: The target (dictionary) resource
- **`roles`**: First document is `"source"`, second is `"target"`

### Records

Each record aligns one or more source tokens to one or more dictionary entries.

```json
{
  "references": [
    ["40001001008|Ἀβραάμ"],
    ["000011000000000"]
  ]
}
```

- **`references[0]`**: Array of source token identifiers (see [Source Token Format](#source-token-format))
- **`references[1]`**: Array of target dictionary entry identifiers (see [Target Entry Identifiers](#target-entry-identifiers))

Most records have a single source token and a single target entry. Multiple source tokens occur when a Hebrew word is morphologically segmented into parts (see [Hebrew Morphological Parts](#hebrew-morphological-parts)).

## Source Token Format

Source tokens combine a positional identifier with the word form:

```
{token_id}|{word_form}
```

### Greek (SBLGNT)

Scheme: `BCVW|token-string`

Token IDs are 11 digits: `BBCCCVVVWWW`

| Field | Digits | Description |
|---|---|---|
| BB | 2 | Book number (40–66 for NT) |
| CCC | 3 | Chapter |
| VVV | 3 | Verse |
| WWW | 3 | Word position within verse |

**Examples**

- `40001001001|Βίβλος` — Matthew 1:1, word 1, "Βίβλος"
- `43003016016|οὕτως` — John 3:16, word 16, "οὕτως"

### Hebrew (WLCM)

Scheme: `BCVWP|token-string`

Token IDs are 12 digits: `BBCCCVVVWWWP`

| Field | Digits | Description |
|---|---|---|
| BB | 2 | Book number (01–39 for OT) |
| CCC | 3 | Chapter |
| VVV | 3 | Verse |
| WWW | 3 | Word position within verse |
| P | 1 | Morphological part (1 or 2) |

**Examples**

- `010110290161|אֲבִי` — Genesis 11:29, word 16, part 1, "אֲבִי"
- `010020240061|אָבִי` + `010020240062|ו` — Genesis 2:24, word 6, parts 1 and 2

### Hebrew Morphological Parts

The WLCM tokenization segments some Hebrew words into morphological constituents. For example, a prefixed conjunction (ו, "and") attached to a root is represented as two part tokens. When a single lexeme alignment spans multiple parts, all part tokens appear in `references[0]`:

```json
{
  "references": [
    [
      "010020240061|אָבִי",
      "010020240062|ו"
    ],
    ["000003000000000"]
  ]
}
```

Both tokens together represent one dictionary headword. The majority of Hebrew records involve a single part token; roughly 15% involve two parts.

## Target Entry Identifiers

Target identifiers are 15-digit strings that uniquely identify a dictionary entry:

```
000003000000000
```

The first 9 digits are the zero-padded entry number; the trailing 6 digits are zeros reserved for future use. This full 15-digit string is used directly as the `content_id` for article lookup — no extraction is needed.

The target scheme encodes which dictionary the entry belongs to:

| Dictionary | `docid` | `scheme` |
|---|---|---|
| UBS Greek NT Dictionary | `UBSGreekNTDictionary` | `ubs-dgnt` |
| UBS Hebrew Dictionary | `UBSHebrewDictionary` | `ubs-dbh` |

## Looking Up Dictionary Articles

The 15-digit target identifier is the `content_id` used throughout the dictionary resource. To retrieve the corresponding article:

1. Take the target identifier from the alignment record (e.g., `"000003000000000"`)
2. Look it up as a key in `article_metadata` in the dictionary's `metadata.json`
3. The entry provides the headword (`index_reference`) and localized titles

```json
"000003000000000": {
  "content_id": "000003000000000",
  "reference_id": "000003000000000",
  "index_reference": "אָב",
  "localizations": {
    "spa": { "content_id": "000003000000000", "language": "spa", "title": "אָב" },
    "fra": { "content_id": "000003000000000", "language": "fra", "title": "אָב" }
  }
}
```

The article content file is at `json/{content_id}.json` within the language folder.

## Metadata

The dictionary resource's `metadata.json` lists alignment files under `scripture_burrito.ingredients`:

```json
{
  "scripture_burrito": {
    "ingredients": {
      "alignments/01.alignment.json": {
        "mimeType": "text/json",
        "size": 4521389,
        "scope": { "GEN": [] }
      },
      "alignments/40.alignment.json": {
        "mimeType": "text/json",
        "size": 1234567,
        "scope": { "MAT": [] }
      }
    }
  }
}
```

- **Key**: Path to the alignment file relative to the language folder
- **`scope`**: USFM book name indicating which book the file covers
- **`size`**: File size in bytes

Enumerate `scripture_burrito.ingredients` to discover which books have alignment data before loading alignment files.

## Relation to Source Text Datasets

The token IDs in lexical alignments use the same tokenization as Biblica's Macula datasets:

- **Greek** (SBLGNT): [github.com/Clear-Bible/macula-greek](https://github.com/Clear-Bible/macula-greek)
- **Hebrew** (WLCM): [github.com/Clear-Bible/macula-hebrew](https://github.com/Clear-Bible/macula-hebrew)

These datasets provide morphological tagging, lemmas, Strong's numbers, and other word-level data keyed to the same token IDs. Implementors who want to display morphological information alongside dictionary entries can join on the token ID.

The [Clear Bible Alignments repository](https://github.com/Clear-Bible/Alignments) contains the upstream source TSVs (`WLCM.tsv`, `SBLGNT.tsv`) from which the enriched token strings in these files are derived.

## Integration Example

```python
import json

# Load alignment file
with open("40.alignment.json", encoding="utf-8") as f:
    data = json.load(f)

# Load dictionary article metadata
with open("metadata.json", encoding="utf-8") as f:
    metadata = json.load(f)
article_meta = metadata["article_metadata"]

for group in data["groups"]:
    for record in group["records"]:
        source_tokens = record["references"][0]   # e.g. ["40001001008|Ἀβραάμ"]
        target_entries = record["references"][1]  # e.g. ["000011000000000"]

        for token in source_tokens:
            token_id, word_form = token.split("|", 1)
            # token_id encodes book/chapter/verse/word (and part for Hebrew)

        for entry_id in target_entries:
            article = article_meta.get(entry_id, {})
            headword = article.get("index_reference", "")
            # headword is the dictionary entry's lemma form
```

## Related Documentation

- [Aquifer Alignments](aquifer_alignments.md) — textual alignments between Bible translations
- [Aquifer Resource Metadata](aquifer_resource_metadata.md) — full `metadata.json` structure
- [Aquifer Article Metadata](aquifer_article_metadata.md) — article content and metadata structure
