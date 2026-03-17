# Aquifer Alignments

Documentation on textual alignment data for Bibles available in the Aquifer.

## Alignment information in `/[lang]/metadata.json`

### `scripture_burrito/ingredients/`

If a Bible translation has associated alignment metadata, paths to the files will be included in the `scripture_burrito/ingredients/` metadata. For example, the Berean Standard Bible metadata will have Scripture Burrito ingredients like:

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
  },

```

This information positively associates an alignment file (e.g. `alignments/01.alignment.json`) with a book of the Bible (here `GEN` which is the [standard USFM book name](https://ubsicap.github.io/usfm/identification/books.html) for Genesis). In this way one can better understand exactly what alignment data is available for a Bible and customize implementations that utilize alignment data.


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

The alignment data provided is compliant with v0.3 of the Scripture Burrito textual alignment specification. The specification itself is incredibly flexible; the implementation of that specification generally follows the implementation practices of Biblica (see their [available alignment data repository](https://github.com/Clear-Bible/Alignments) ). Very brief documentation of the Aquifer implemention of the Scripture Burrito spec follows.

### Files

Each alignment json file represents a book of the Bible.

### `documents`

The following is from the Berean Standard Bible's `40.alignments.json` file, which represents Matthew. 

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

In this representation, the first "document" in the list represents the alignment "Source" (here the SBLGNT), the second 
represents the "Target" (here the BSB or Berean Standard Bible). Each document also specifies a `scheme` it uses to represent units. 

The SBLGNT uses a `BCVWP|token-string` scheme. In this alignment, a source unit is specified by a `BCVWP` (testament-Book-Chapter-Verse-Word-Part) string; so `n40001001001` for the first word token found in Matthew 1:1 (book 40, chapter 1, verse 1, word 1). The pipe (`|`) is simply a delimiter, and the `token-string` is the actual word from the edition (here `Βίβλος`). The combined token representing the first word of Matthew 1:1 in the SBLGNT is `n40001001001|Βίβλος`.

The BSB uses a `BCVW|token-string` scheme. In this alignment, a target unit is specified by a `BCVW` (Book-Chapter-Verse-Word) scheme. So `40001001001` represents Matthew 1:1, word 1, or "This". The combined token would be `40001001001|This`. 

### `records`

A **record** is essentially a list of **alignment units**.

An **alignment unit** associates a set of **Source** token identifiers with set of **Target** token identifiers.

Because alignments support a many-to-many relationship, we align a list or set of source token identifiers with a list or set of target token identifiers. An example is below.

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
    },
```

For alignment units we provide both the source token identifiers as well as the actual word strings involved. This allows implementors to use this information to align with either the text itself from the aquifer, or from an edition of the text the implementor has already integrated into their system or application.

The `meta` block simply provides an identifier to the alignment unit as well as some basic metadata of how the alignment came about. The `origin` and `status` will likely be consistent for the entire translation, and they will usually be of little interest to the implementor.


## Suggestions on Display of Alignment data

The Aquifer provides the following textual data:

* The text of the Bible, ordered by verse
* Alignment units available for the Bible
* Linkages in metadata to the Old Testament basis and New Testament basis (the original language texts the Bible is aligned to)

With this information, displays that provide the translated text in reading order with Greek or Hebrew word assocation information below is possible, like the following example from Matthew 1:1 of the Berean Standard Bible:

<blockquote>
<p style='display: block'><b>MAT 1:1:</b> <div style='display: inline-block; padding: 1 1 1 1'><div>This</div><div>Βίβλος <sub style='font-size: 8pt'>1</sub></div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>is</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>the</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>record</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>of</div><div>γενέσεως <sub style='font-size: 8pt'>2</sub></div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>the</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>genealogy</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>of</div><div>Ἰησοῦ <sub style='font-size: 8pt'>3</sub></div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>Jesus</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>Christ</div><div>χριστοῦ <sub style='font-size: 8pt'>4</sub></div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>,</div><div> </div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>the</div><div>υἱοῦ <sub style='font-size: 8pt'>5</sub></div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>son</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>of</div><div>Δαυὶδ <sub style='font-size: 8pt'>6</sub></div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>David</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>,</div><div> </div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>the</div><div>υἱοῦ <sub style='font-size: 8pt'>7</sub></div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>son</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>of</div><div>Ἀβραάμ <sub style='font-size: 8pt'>8</sub></div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>Abraham</div><div>&larr;</div></div> <div style='display: inline-block; padding: 1 1 1 1'><div>:</div><div> </div></div> </p>
</blockquote>

Here the linkages are evident. The left-pointing arrows indicate the available alignment units; so in this instance "This is the record" aligns with "Βίβλος"; "of the genealogy" aligns with "γενέσεως", and such. The digits after the Greek word represent the reading order of the Greek text. Be aware, however, that in many instances the reading order of one language does not necessarily follow the reading order of another. In these cases providing the word numbers of the original language can be useful.

Further, with the information available in the textual alignments (speaking specifically of the token IDs of the original language portions), one can bring in information from alternate datasets that support the same tokenization. In our case, the [ACAI dataset](https://github.com/BibleAquifer/ACAI) supports this tokenization. This means for a given original language token, the ACAI entity/concept that provides annotation of that token can be imported at the word level for an association. One possible visualization of that information is below:

<blockquote>
<p style='line-height:2.5' dir='l2r'><b>MAT 1:1:</b> This is the record of the genealogy <span style='background: gainsboro;padding: 0.45em 0.6em; margin: 0 0.25em;line-height:2;border-radius 0.35em;'>of Jesus<span style='background: #aa9cfc;font-size: 60%; font-family: Arial;line-height: 2;border-radius: 0.35em; text-transform: uppercase;vertical-align: super;margin-left: 0.5rem'><a href='https://github.com/BibleAquifer/ACAI/blob/main/people/md/Jesus.2.md' target='_blank'>person:Jesus.2</a></span></span> <span style='background: gainsboro;padding: 0.45em 0.6em; margin: 0 0.25em;line-height:2;border-radius 0.35em;'>Christ<span style='background: #aa9cfc;font-size: 60%; font-family: Arial;line-height: 2;border-radius: 0.35em; text-transform: uppercase;vertical-align: super;margin-left: 0.5rem'><a href='https://github.com/BibleAquifer/ACAI/blob/main/people/md/Christ.md' target='_blank'>person:Christ</a></span></span>  the son <span style='background: gainsboro;padding: 0.45em 0.6em; margin: 0 0.25em;line-height:2;border-radius 0.35em;'>of David<span style='background: #aa9cfc;font-size: 60%; font-family: Arial;line-height: 2;border-radius: 0.35em; text-transform: uppercase;vertical-align: super;margin-left: 0.5rem'><a href='https://github.com/BibleAquifer/ACAI/blob/main/people/md/David.md' target='_blank'>person:David</a></span></span>  the son <span style='background: gainsboro;padding: 0.45em 0.6em; margin: 0 0.25em;line-height:2;border-radius 0.35em;'>of Abraham<span style='background: #aa9cfc;font-size: 60%; font-family: Arial;line-height: 2;border-radius: 0.35em; text-transform: uppercase;vertical-align: super;margin-left: 0.5rem'><a href='https://github.com/BibleAquifer/ACAI/blob/main/people/md/Abraham.md' target='_blank'>person:Abraham</a></span></span> </p>

</blockquote>

