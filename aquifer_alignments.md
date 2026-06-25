# Aquifer Alignments

Documentation on textual alignment data for Bibles available in the Aquifer.

## Alignment information in `/[lang]/metadata.json`

### `scripture_burrito/ingredients/`

If a Bible translation has associated alignment metadata, paths to the files will be included in the `scripture_burrito/ingredients/` metadata. Two filename patterns are used depending on the alignment format version (see [Alignment Format Versions](#alignment-format-versions) below):

**Book-level entries (v0.3)** — a single file covers an entire book; `scope` is an empty array:

```json
  "alignments/01.alignment.json": {
    "mimeType": "text/json",
    "size": 5362569,
    "scope": {
      "GEN": []
    }
  },
  "alignments/02.alignment.json": {
    "mimeType": "text/json",
    "size": 4349395,
    "scope": {
      "EXO": []
    }
  }
```

**Chapter-level entries (v0.4)** — one file per chapter; `scope` lists the single chapter number:

```json
  "alignments/40-001.alignment.json": {
    "mimeType": "text/json",
    "size": 98432,
    "scope": {
      "MAT": ["1"]
    }
  },
  "alignments/40-002.alignment.json": {
    "mimeType": "text/json",
    "size": 112045,
    "scope": {
      "MAT": ["2"]
    }
  }
```

For any given book you will find either a single book-level file (v0.3) or a set of chapter-level files (v0.4), never both. The `scope` value distinguishes them: an empty array (`[]`) means the file covers the whole book; a one-element array (e.g. `["1"]`) means it covers that chapter only. This information lets implementors determine exactly what alignment data is available for a Bible and load only what is needed.


### `alignment_metadata`

If a Bible includes data that supports textual alignment, the Bible's `metadata.json` will have an `alignment_metadata` section. This section includes information on the sources and license of the alignment data. An example from the Berean Standard Bible is below:

```json
`"alignment_metadata": {
	"alignment_source": "https://github.com/Clear-Bible/alignments-eng/tree/main/data/alignments/BSB",
	"alignment_ot_basis": "https://github.com/Clear-Bible/Alignments/tree/main/data/sources/WLCM.tsv",
	"alignment_nt_basis": "https://github.com/Clear-Bible/Alignments/tree/main/data/sources/SBLGNT.tsv",
	"licenses": [
	  {
		"eng": {
		  "name": "CC BY-SA 4.0",
		  "url": "https://creativecommons.org/licenses/by-sa/4.0/"
		}
	  }
	]
```

As mentioned in [Aquifer Resource Metadata](aquifer_resource_metadata.md), the keys are defined as follows:

* `alignment_source`: A URL string indicating the source (github repository) of the alignment data. This repository may or may not be publicly available.
* `alignment_ot_basis`: A URL indicating the edition/source/basis for the Old Testament alignment data. At present there is one possible value:
  * `WLCM`: A URL pointing to a TSV representing the Westminster Leningrad Codex as implemented in Biblica's [Macula Hebrew](https://github.com/Clear-Bible/macula-hebrew) dataset.
* `alignment_nt_basis`: A URL indicating the edition/source/basis for the New Testament alignment data. At present there is one possible value:
  * `SBLGNT`: A URL pointing to a TSV representing _The SBL Greek New Testament_ as implemented in Biblica's [Macula Greek](https://github.com/Clear-Bible/macula-greek) dataset.
* `licenses`: A list of dictionaries that provide a URL for each supported license.

The TSV data referred to as underlying basis utilize the same tokenization identifiers as the alignments and may contain additional word-level data (morphology / part of speech, strongs numbers, lemmas, etc.) that may be useful for implementors.

## Alignment Information in `/[lang]/alignments/`

Aquifer alignment data uses two versions of the Scripture Burrito textual alignment specification. Both follow implementation practices established by Biblica (see their [available alignment data repository](https://github.com/Clear-Bible/Alignments)).

### Alignment Format Versions

**v0.3** is the legacy format. It is being phased out as higher-quality v0.4 data becomes available. v0.3 files cover one book per file and use a flat top-level structure (`documents`, `meta`, `roles`, `type`, `records`).

**v0.4** is the current format, produced by LLM-assisted refinement. v0.4 files cover one chapter per file and wrap the same data inside a `groups` array, which enables richer per-record and per-group metadata (see [records](#records) below).

For a given book, only one format will be present: v0.4 where the entire book has been refined; v0.3 otherwise.

### Files

Book-level files (`{BB}.alignment.json`, e.g. `01.alignment.json`) are v0.3 and cover a whole book. Chapter-level files (`{BB}-{CCC}.alignment.json`, e.g. `40-001.alignment.json`) are v0.4 and cover a single chapter.

### `documents`

The `documents` array identifies the source and target texts and the token ID scheme each uses. The first entry is always the source (original language); the second is the target (translation).

#### v0.3 `documents` example

The following is from the Berean Standard Bible's `40.alignment.json` file (Matthew, v0.3 format):

```json
"documents": [
  {
    "docid": "SBLGNT",
    "scheme": "BCVWP|token-string"
  },
  {
    "docid": "BSB",
    "scheme": "BCVW|token-string"
  }
],
```

In v0.3, the SBLGNT source uses `BCVWP` (prefix-Book-Chapter-Verse-Word-Part). The prefix identifies the testament: `n` for New Testament, `o` for Old Testament. So `n40001001001` is Matthew 1:1 word 1, and `o010010010011` is Genesis 1:1 word 1 part 1. The BSB target uses `BCVW` (Book-Chapter-Verse-Word, no prefix), so `40001001001` is Matthew 1:1 word 1.

#### v0.4 `documents` example

The following is from the Berean Standard Bible's `40-001.alignment.json` file (Matthew 1, v0.4 format). In v0.4, `documents` is nested inside `groups[0]`:

```json
"groups": [
  {
    "type": "translation",
    "documents": [
      {
        "docid": "SBLGNT",
        "scheme": "BCVWP|token-string"
      },
      {
        "docid": "BSB",
        "scheme": "BCVWP|token-string"
      }
    ]
  }
]
```

In v0.4, **both** source and target use the `BCVWP` scheme and neither carries a testament prefix. So the first source word of Matthew 1:1 in the SBLGNT is `40001001001|Βίβλος` and a target token in the BSB is `40001001003|This`.

In both formats the pipe (`|`) is a delimiter and the `token-string` portion is the surface form of the word from that edition.

### `records`

A **record** is an **alignment unit**: a many-to-many association between a set of source token identifiers and a set of target token identifiers. Both source and target include the surface form after the `|` delimiter so implementors can match against the text directly without a separate lookup.

#### v0.3 records

```json
"records": [
  {
    "source": [
      "n40001001001|Βίβλος"
    ],
    "target": [
      "40001001001|This",
      "40001001002|is",
      "40001001003|the",
      "40001001004|record"
    ],
    "meta": {
      "id": "40001001.001",
      "origin": "manual",
      "status": "created"
    }
  }
]
```

The `meta` block carries a record identifier and provenance fields. These are typically uniform across a translation and are of little interest to most implementors.

#### v0.4 records

v0.4 records carry richer metadata. Source token IDs have no testament prefix; both source and target use the `BCVWP` scheme.

```json
"records": [
  {
    "source": [
      "40001001001|Βίβλος"
    ],
    "target": [
      "40001001003|This",
      "40001001004|is",
      "40001001005|the",
      "40001001006|record"
    ],
    "meta": {
      "secondary": {
        "source": [],
        "target": [
          "40001001004|is",
          "40001001005|the"
        ]
      }
    }
  },
  {
    "source": [
      "40001005003|λέγων",
      "40001005004|Ποῦ"
    ],
    "target": [
      "40001005007|asking",
      "40001005008|where"
    ],
    "meta": {
      "is_idiom": true
    }
  }
]
```

Record-level `meta` fields:

| Field | Type | Meaning |
|---|---|---|
| `meta.secondary.source` | string[] | Source tokens in this record that are secondary (grammatically implied, not a direct lexical equivalent) |
| `meta.secondary.target` | string[] | Target tokens in this record that are secondary |
| `meta.is_idiom` | bool | The record represents a phrase-to-phrase idiomatic alignment; neither side can be decomposed word-by-word |

Tokens absent from `secondary.source` / `secondary.target` are primary alignments. `is_idiom` and `secondary` are mutually exclusive on any given record.

#### v0.4 group-level `meta`

The group `meta` object carries three kinds of information:

**Non-equivalent tokens** — token IDs positively confirmed to have no counterpart in the other language:

```json
"meta": {
  "nonEquivalent": {
    "source": [
      "40001002006|Ἰακώβ"
    ],
    "target": [
      "40001001001|This",
      "40001001002|is"
    ]
  }
}
```

Source tokens in `nonEquivalent.source` are original-language words with no translation equivalent in this passage. Target tokens in `nonEquivalent.target` are translation words with no corresponding original-language word. These are distinct from simply unaligned tokens — they represent a positive determination of non-equivalence.

**LLM provenance** — identifies the model that produced and (optionally) refined the alignment:

```json
"meta": {
  "llm": {
    "provider": "gloo",
    "model": "gloo-google-gemini-3.5-flash",
    "reasoning_effort": "high"
  },
  "retry_llm": {
    "provider": "gloo",
    "model": "gloo-anthropic-claude-opus-4.7",
    "reasoning_effort": "high"
  }
}
```

`retry_llm` is only present when a retry pass has been run. These fields are informational; implementors do not need to act on them.


## Suggestions on Display of Alignment data

**Note:** The Github markdown rendering does not display the sample HTML as desired; you may need to view this file in an editor of your choosing.

The Aquifer provides the following textual data:

* The text of the Bible, ordered by verse
* Alignment units available for the Bible
* Linkages in metadata to the Old Testament basis and New Testament basis (the original language texts the Bible is aligned to)

With this information, displays that provide the translated text in reading order with Greek or Hebrew word assocation information below is possible, like the following example from Matthew 1:1 of the Berean Standard Bible:
<style>
body  { font-family: serif; font-size: 14px; }
.cell { display: inline-block; vertical-align: top; padding: 1px 4px;
        text-align: center; min-width: 1.2em; }
.tgt  { display: block; margin-bottom: 2px; min-height: 1.2em; }
.src  { display: block; font-size: 90%; }
.neq  { color: #bbb; }
.idiom{ font-style: italic; }

.sub  { font-size: 60%; }
.tri  { font-size: 90%; vertical-align: 1px; }
.acai-hl  { background: #d0e8ff; border-radius: 2px; padding: 0 1px; }
.acai-tag { font-size: 55%; font-family: Arial, sans-serif;
            text-transform: uppercase; position: relative; top: -0.6em;
            margin-left: 0.4em; color: #446; }
.file-meta { font-family: Arial, sans-serif; font-size: 12px; color: #666;
             margin: 2px 0 14px 0; letter-spacing: 0.01em; }
.file-meta .meta-edition { font-weight: bold; color: #333; }
.file-meta .meta-sep { color: #bbb; margin: 0 7px; }
</style>
<blockquote>
<p style='display: block'><b>MAT 1:1:</b> <div style='display: inline-block; padding: 1 1 1 1'><div>This</div><div>Βίβλος <sub style='font-size: 8pt'>1</sub></div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>is</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>the</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>record</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>of</div><div>γενέσεως <sub style='font-size: 8pt'>2</sub></div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>the</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>genealogy</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>of</div><div>Ἰησοῦ <sub style='font-size: 8pt'>3</sub></div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>Jesus</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>Christ</div><div>χριστοῦ <sub style='font-size: 8pt'>4</sub></div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>,</div><div> </div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>the</div><div>υἱοῦ <sub style='font-size: 8pt'>5</sub></div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>son</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>of</div><div>Δαυὶδ <sub style='font-size: 8pt'>6</sub></div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>David</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>,</div><div> </div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>the</div><div>υἱοῦ <sub style='font-size: 8pt'>7</sub></div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>son</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>of</div><div>Ἀβραάμ <sub style='font-size: 8pt'>8</sub></div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>Abraham</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>:</div><div> </div></div> </p>
</blockquote>

Here the linkages are evident. The left-pointing arrows indicate the available alignment units; so in this instance "This is the record" aligns with "Βίβλος"; "of the genealogy" aligns with "γενέσεως", and such. The digits after the Greek word represent the reading order of the Greek text. Be aware, however, that in many instances the reading order of one language does not necessarily follow the reading order of another. In these cases providing the word numbers of the original language can be useful.

Further, with the information available in the textual alignments (speaking specifically of the token IDs of the original language portions), one can bring in information from alternate datasets that support the same tokenization. In our case, the [ACAI dataset](https://github.com/BibleAquifer/ACAI) supports this tokenization. This means for a given original language token, the ACAI entity/concept that provides annotation of that token can be imported at the word level for an association. One possible visualization of that information is below:

<blockquote>
<p style='line-height:2.5' dir='l2r'><b>MAT 1:1:</b> This is the record of the genealogy <span style='background: gainsboro;padding: 0.45em 0.6em; margin: 0 0.25em;line-height:2;border-radius 0.35em;'>of Jesus<span style='background: #aa9cfc;font-size: 60%; font-family: Arial;line-height: 2;border-radius: 0.35em; text-transform: uppercase;vertical-align: super;margin-left: 0.5rem'><a href='https://github.com/BibleAquifer/ACAI/blob/main/people/md/Jesus.2.md' target='_blank'>person:Jesus.2</a></span></span> <span style='background: gainsboro;padding: 0.45em 0.6em; margin: 0 0.25em;line-height:2;border-radius 0.35em;'>Christ<span style='background: #aa9cfc;font-size: 60%; font-family: Arial;line-height: 2;border-radius: 0.35em; text-transform: uppercase;vertical-align: super;margin-left: 0.5rem'><a href='https://github.com/BibleAquifer/ACAI/blob/main/people/md/Christ.md' target='_blank'>person:Christ</a></span></span>  the son <span style='background: gainsboro;padding: 0.45em 0.6em; margin: 0 0.25em;line-height:2;border-radius 0.35em;'>of David<span style='background: #aa9cfc;font-size: 60%; font-family: Arial;line-height: 2;border-radius: 0.35em; text-transform: uppercase;vertical-align: super;margin-left: 0.5rem'><a href='https://github.com/BibleAquifer/ACAI/blob/main/people/md/David.md' target='_blank'>person:David</a></span></span>  the son <span style='background: gainsboro;padding: 0.45em 0.6em; margin: 0 0.25em;line-height:2;border-radius 0.35em;'>of Abraham<span style='background: #aa9cfc;font-size: 60%; font-family: Arial;line-height: 2;border-radius: 0.35em; text-transform: uppercase;vertical-align: super;margin-left: 0.5rem'><a href='https://github.com/BibleAquifer/ACAI/blob/main/people/md/Abraham.md' target='_blank'>person:Abraham</a></span></span> </p>

</blockquote>

