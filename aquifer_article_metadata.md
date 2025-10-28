# Aquifer Article Metadata JSON

* **Author:** Rick Brannan (`rickb@missionmutual.org`)
* **Date:** 2025-10-16

# Introduction

[an introduction to appear here some time in the future]

# Terminology

This document uses vocabulary with specific meaning in the context of the Aquifer. These terms are:

* **Resource**: A **Resource** is a specific language version of a whole content piece. For example, the _Biblica Study Notes_ in English is a **Resource**. The _Biblica Study Notes_ in Arabic is a different resource.
* **Parent Resource**: A **Parent Resource** is the collection of all available languages of a particular content piece.
* **Resource Type**: The Aquifer presently defines the following **Resource Types**:
  * Study Notes
  * Bible Dictionaries
  * Translation Guides
  * Images
  * Videos
* **Article**: An **Article** is essentially a record within a **Resource**. So the _Biblica Study Notes_ entry for “Genesis 1:1–2:25” is an **Article**.

# JSON

An aquifer content JSON file is a list of dictionaries, where each dictionary in the list contains article-level metadata and the article content.

For canonically-ordered resources, each content file represents the material of a book of the Bible. The files are named like `NN.content.(json|md)` where `NN` is the zero-padded Bible book number. So `01.content.json` is the data in JSON for Genesis, `40.content.md` is content for Matthew in Markdown, etc. 

For other resources (alphabetially ordered and monograph resources), each file, while still a list of articles, only contains information for one article. The files are named like `NNNNNN.content.(json|md)` If the files are sorted in an ascending order, the content of the resource will be in the proper order.

* `version`: The version of the JSON schema `aquifer_article.schema.json` (located in `/schemas`) the metadata has been validated against. This is currently set at `1.0.0`. 
* `content_id`: The `content_id`
* `reference_id`: If the `content_id` is used in a reference in the aquifer, it may use the `reference_id` for navigation.
* `index_reference`: The article sort key, essentially. For canonically-ordered resources, it is an eight-digit string representing the book, chapter, and verse of the Bible reference (`BBCCCVVV`). It may also indicate a range, with two references (`BBCCCVVV`–`BBCCCVVV`). For alphabetically-ordered resources, it is the sort key used (lower-cased article title, typically).
* `media_type`: This is the media of the `content`. If the `content` is essentially a container pointing to an image, the `content` will be `Image`. Below are the supported `media_type` values:
  * `Text`
  * `Image`
  * `Video`
* `language`: A three-letter code compatible with the ISO three-letter language codes.
* `content`: An HTML representation of the article content. Some specialized elements (for linking and sometimes media) do occur; we will offer documentation of this HTML elsewhere. In the Markdown generated from this HTML, Bible references are enabled as links to Logos Bible Software's [ref.ly Bible reference linking system](https://ref.ly). This will probably change to something else, but for the short-term it provides a destination for these links.
* `associations`: This is article-level metadata regarding different types of links or relationships between this article and other articles. The "other articles" may be within the current resource, within a different resource, or to the Bible (by book, chapter, verse reference). Essentially, these are destinations you may be interested in examining based on the article. For _Study Notes_ resources, the reference of the note is usually included as a `passage` association. Many _Study Notes_ resources have related key-term style resources (typically implemented as sidebars or backmatter in print editions); these sorts of references are included as `resource` associations. Still to-do on `resource` associations: Include the resource name of the destination to facilitate easier link generation.
  * `passage`: Each item in the `passage` list has four items.
    * `start_ref`: The `BBCCCVVV` style reference of the starting Bible reference.
	* `start_ref_usfm`: The start reference rendered using standard USFM names.
	* `end_ref`: The `BBCCCVVV` style reference of the ending Bible reference. If the reference range is only one verse (e.g. "GEN 1:1"), then the `start_ref` and the `end_ref` will match.
	* `end_ref_usfm`: The end reference rendered using standard USFM names. If the reference range is only one verse (e.g. "GEN 1:1"), then the `start_ref_usfm` and the `end_ref_usfm` will match.
  * `resource`: Each item in the `resource` list has five items.
    * `reference_id`: An identifier for the unique reference itself.
	* `content_id`: The identifier of the article in the target resource.
	* `label`: A string representing a label for the article. If something clickable is needed for the link, this label can be used as the clickable content.
	* `language`: ISO three-letter code representing the language. This, plus the `resource_code` allow the destination to be easily specified in a manner consistent with how the material is laid out in the github repositories.
	* `resource_code`: The `resource_code` uniquely identifies the resource and is also the repository name for the resource in github. 