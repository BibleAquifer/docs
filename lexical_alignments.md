# Lexical Alignments

Lexical alignments in the Aquifer map Greek and Hebrew words from the original biblical texts to entries in semantic dictionaries. This document describes the structure, format, and usage of lexical alignment data.

## Overview

Lexical alignments connect tokens from source biblical texts (SBLGNT for Greek, WLCM for Hebrew) to specific entries in target dictionary resources (UBS Dictionary of the Greek New Testament, UBS Hebrew Dictionary, and future resources). Unlike textual alignments which map between different Bible translations, lexical alignments support word-level semantic lookup, contextual dictionary navigation, and cross-referencing between texts and reference materials.

### When Lexical Alignments Are Available

Lexical alignments are available for:
- **UBS Dictionary of the Greek New Testament** (NT only, aligned to SBLGNT)
- **UBS Hebrew Dictionary** (OT only, aligned to WLCM)
- Future dictionary resources as they are integrated into the Aquifer

## Alignment JSON Structure

Lexical alignment data is distributed as JSON files compliant with Scripture Burrito v0.4 alignment specification with Aquifer extensions. Each alignment file corresponds to a single biblical book.

### File Organization

```
/{lang}/converted/alignments/
├── 01.alignment.json  (Genesis)
├── 02.alignment.json  (Exodus)
├── ...
├── 39.alignment.json  (Malachi)
├── 40.alignment.json  (Matthew)
├── ...
└── 66.alignment.json  (Revelation)
```

Book numbers follow standard USFM numbering: OT books 01–39, NT books 40–66.

### Top-Level Structure

```json
{
  "format": "alignment",
  "version": "0.4",
  "groups": [...]
}
```

- **`format`**: Always `"alignment"` for Scripture Burrito compliance
- **`version`**: Specification version; currently `"0.4"`
- **`groups`**: Array of alignment groups (typically one per file)

### Groups

Each group defines alignments of a specific type and includes metadata about the source and target documents.

```json
{
  "type": "source-word-to-lexicon",
  "meta": {
    "creator": "convert-semantic-dictionaries",
    "created": "2026-05-12"
  },
  "documents": [...],
  "roles": ["source", "target"],
  "records": [...]
}
```

- **`type`**: `"source-word-to-lexicon"` for dictionary alignments
- **`meta`**: Origin and creation date of the alignment data
- **`documents`**: Array of source and target document definitions
- **`roles`**: Always `["source", "target"]` — first document is source, second is target
- **`records`**: Array of individual alignment units

### Documents

Documents define the token scheme and identifier for each text involved in the alignment.

**Source Document (Original Language Text)**

```json
{
  "scheme": "BCVW|token-string",
  "docid": "SBLGNT"
}
```

- **`docid`**: `"SBLGNT"` for Greek texts, `"WLC"` for Hebrew texts
- **`scheme`**: `"BCVW|token-string"` — Book-Chapter-Verse-Word format with the actual word form
  - Structure: `BBCCCVVVWWW|word_form`
  - Example: `40001001001|Βίβλος` = Matthew 1:1, word 1, the word "Βίβλος"
  - Prefix conventions: none for NT (books 40–66), leading zeros for all positions
  - For OT texts, `docid` would be `"WLC"` with the same scheme

**Target Document (Dictionary)**

```json
{
  "scheme": "ubs-dgnt",
  "docid": "UBSGreekNTDictionary"
}
```

- **`docid`**: Dictionary resource identifier (e.g., `"UBSGreekNTDictionary"`, `"UBSHebrewDictionary"`)
- **`scheme`**: Dictionary-specific token format (see sections below)

#### Target Schemes

**UBS Greek Dictionary (`ubs-dgnt`)**

```
000006000000000
```

This 15-digit identifier uniquely identifies a dictionary entry. The structure is:

```
[entry_id:9 digits][reserved:6 digits]
```

- **Entry ID**: A 9-digit zero-padded entry identifier within the dictionary
- **Reserved**: Six trailing zeros; reserved for future use

Example: `000006000000000` identifies dictionary entry #6 in the UBS Greek Dictionary.

**UBS Hebrew Dictionary (`ubs-dhb`)**

```
000003000000000
```

Similar 15-digit format with:
- **Entry ID**: A 9-digit zero-padded entry identifier
- **Reserved**: Six trailing zeros

To find the corresponding dictionary article, use the entry ID (first 9 digits, zero-padded) as the `content_id` in the dictionary resource's article metadata.

### Records

Records are the individual alignment units. Each record maps one or more source tokens to one or more target dictionary entries.

```json
{
  "references": [
    ["40001001008|Ἀβραάμ"],
    ["000011000000000"]
  ]
}
```

**Structure**

- **`references`**: Array with exactly two elements
  - **`references[0]`**: Array of source token identifiers (typically one; may be multiple for phrases)
  - **`references[1]`**: Array of target dictionary entry identifiers (typically one per unique lexeme)

**Source Token Format**

Source tokens include both the token ID and the word form:

```
BBCCCVVVWWW|word_form
```

Example: `40001001008|Ἀβραάμ`

**Many-to-Many Alignments**

While most records align a single Greek/Hebrew word form to a single dictionary entry, the structure supports many-to-many relationships:

```json
{
  "references": [
    ["40001001008|Ἀβραάμ", "40001001009|Ἀβραὰμ"],
    ["000011000000000"]
  ]
}
```

This would indicate that two different forms (with different diacritics) map to the same dictionary entry, which is typical for inflected languages.

## Token ID Schemes

### SBLGNT (Greek New Testament)

Token IDs use the format: `BBCCCVVVWWW`

- **BB**: Book number (40–66 for NT)
- **CCC**: Chapter (001–150+)
- **VVV**: Verse (001–999, though typically ≤chapter verse limit)
- **WWW**: Word position within the verse (001–999, typically ≤50)

**Examples**

- `40001001001` = Matthew 1:1, word 1
- `42003016008` = Luke 3:16, word 8
- `46007004012` = 1 Corinthians 7:4, word 12

### WLCM (Hebrew Old Testament)

Token IDs use the same format as SBLGNT: `BBCCCVVVWWW`

- **BB**: Book number (01–39 for OT)
- **CCC**: Chapter
- **VVV**: Verse
- **WWW**: Word position

**Examples**

- `01001001001` = Genesis 1:1, word 1
- `19023005003` = Psalms 23:5, word 3

## Relating Alignments to Source Texts

### Accessing SBLGNT Source Tokens

The SBLGNT is based on Biblica's Macula Greek dataset. The complete token list with morphological and linguistic data is available at:

```
https://github.com/Clear-Bible/macula-greek
```

The tokenization used in Aquifer alignments matches this dataset exactly. You can:

1. Load the Macula Greek TSV/data files
2. Construct token IDs using the `BBCCCVVVWWW` format
3. Look up morphological tags, part-of-speech, lemmas, and Strong's numbers

### Accessing WLCM Source Tokens

The WLCM is based on Biblica's Macula Hebrew dataset:

```
https://github.com/Clear-Bible/macula-hebrew
```

The tokenization and token ID format is identical to the Greek alignment tokenization.

### Word Form Enrichment

Alignment records include the actual word form in the token string (e.g., `40001001001|Ἀβραάμ`). This allows direct matching against the text without requiring external lookups, though full morphological data is available in the Macula datasets.

## Relating Alignments to Dictionary Resources

### UBS Dictionary of the Greek New Testament

Dictionary entries are identified by a 9-digit `content_id` in the target identifier's first 9 digits:

```
000011000000000  → entry ID: 000011
```

To retrieve the dictionary article:

1. Extract the first 9 digits from the target identifier
2. Use zero-padding to ensure 9 digits (already done in the alignment file)
3. Look up this `content_id` in the dictionary resource's `article_metadata`
4. The `localizations` section contains the article title and localized content IDs

**Example Lookup**

Given alignment record:
```json
{
  "references": [
    ["40001001008|Ἀβραάμ"],
    ["000011000000000"]
  ]
}
```

Steps:
1. Extract entry ID: `000011`
2. Retrieve `article_metadata["000011"]` from the dictionary's `metadata.json`
3. Access the article in the JSON files at `json/000011.json`
4. The article's `title` field contains the dictionary headword (e.g., "Ἀβραάμ")

### UBS Hebrew Dictionary

Same process as Greek dictionary but using `content_id` values from Hebrew dictionary resources and entry IDs from the Hebrew target identifiers.

## Metadata Integration

### Alignment Metadata in `metadata.json`

The dictionary resource's root `metadata.json` contains alignment information:

```json
{
  "scripture_burrito": {
    "ingredients": {
      "alignments/40.alignment.json": {
        "mimeType": "text/json",
        "size": 1234567,
        "scope": {"MAT": []}
      }
    }
  }
}
```

This provides:
- **Location**: Path to the alignment file
- **Size**: File size in bytes (useful for download planning)
- **Scope**: Which biblical books are covered (keyed by USFM book name)

### Entry Point for Third Parties

To discover lexical alignments:

1. Load the dictionary resource's `metadata.json`
2. Iterate through `scripture_burrito.ingredients`
3. Filter for keys matching `alignments/*.alignment.json`
4. Extract the scope to understand which books have alignment data

## Use Cases

### 1. Word-Level Dictionary Lookup

Display the Greek/Hebrew word alongside its dictionary entry:

```
Matthew 1:1 (SBLGNT): Ἀβραάμ
  ↓ [via alignment]
UBS Dictionary entry 11: Ἀβραάμ (Abraham)
```

### 2. Cross-Reference Navigation

Use alignment data to hyperlink from Bible text to dictionary entries, enabling readers to quickly jump to reference material without manual searching.

### 3. Morphological Analysis

Combine alignment token IDs with Macula datasets to provide:
- Lemma information
- Part-of-speech tagging
- Morphological breakdowns
- Strong's numbers (where available)

### 4. Lexical Analysis

Aggregate all occurrences of a lexeme across the NT/OT via alignments to identify patterns, frequency data, and contextual usage.

### 5. Multi-Language Dictionary Support

Use alignments as a stable anchor point for translating dictionary entries across language boundaries. Since alignments are keyed to the original language text, they provide consistent reference points regardless of the target language dictionary version.

## Examples

### Example 1: Single Word to Single Entry

```json
{
  "references": [
    ["40001001001|Βίβλος"],
    ["000001000000000"]
  ]
}
```

The Greek word "Βίβλος" at Matthew 1:1, word 1 aligns to dictionary entry 1.

### Example 2: Variant Forms to Same Entry

```json
{
  "references": [
    ["40001001008|Ἀβραάμ", "40001001009|Ἀβραὰμ"],
    ["000011000000000"]
  ]
}
```

Two variant spellings of the same name (nominative and accusative with different accentuation) both map to the same dictionary entry. This is common in Greek and Hebrew where morphological variation is expected.

### Example 3: Locating the Dictionary Article

Given this alignment:

```json
{
  "references": [
    ["42003016008|λόγος"],
    ["000256000000000"]
  ]
}
```

To retrieve the dictionary article:

1. Load the dictionary's `metadata.json`
2. Find `article_metadata["000256"]`
3. Retrieve the article:
   - English: `article.json/000256.json` or `article_metadata["000256"].localizations.eng`
   - Spanish: `article.json/000256.json` with `article_metadata["000256"].localizations.spa`
4. The article contains the full definition and examples of how "λόγος" is used

## Integration Guidelines

### Loading Alignment Data

```python
import json
from pathlib import Path

alignment_file = Path("40.alignment.json")
with open(alignment_file) as f:
    alignment_data = json.load(f)

for group in alignment_data.get("groups", []):
    source_doc = group["documents"][0]
    target_doc = group["documents"][1]
    
    for record in group.get("records", []):
        source_tokens = record["references"][0]
        target_entries = record["references"][1]
        
        # Process alignment...
```

### Matching Tokens to Text

The `token-string` portion of source tokens (after the `|`) can be matched directly against rendered Bible text, accounting for Unicode normalization and spacing differences.

### Handling Multiple Targets

When a source token aligns to multiple target entries, treat as separate alignments unless explicitly grouped. Multiple targets typically indicate ambiguous or polysemous usage requiring context-dependent selection.

### Scope Filtering

Use the `scope` information from `scripture_burrito.ingredients` to determine which books have alignment data before attempting to load alignment files.

## Technical Specifications

- **File Format**: JSON (UTF-8 encoding)
- **Specification Version**: Scripture Burrito v0.4
- **Token ID Format**: Zero-padded integers with pipe-delimited word forms
- **Dictionary Entry IDs**: 9-digit zero-padded identifiers
- **Source Texts**: SBLGNT (Greek), WLCM (Hebrew) tokenization

## Related Resources

- **Macula Greek**: https://github.com/Clear-Bible/macula-greek
- **Macula Hebrew**: https://github.com/Clear-Bible/macula-hebrew
- **Scripture Burrito Alignment Specification**: https://github.com/bible-technology/alignment-spec/blob/main/spec.md

