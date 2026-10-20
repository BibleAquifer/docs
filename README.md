# Aquifer Documentation

Documentation for the Aquifer data formats (JSON schemas, alignments) and for the Aquifer API, which serves that data.

## Aquifer API

The Aquifer API serves Bible study resources (dictionaries, study notes, translation guides, lexicons, images and video), Bible texts with Greek/Hebrew word alignment, passage lookups, and the ACAI registry of people, places and other entities in the Bible. The API is at <https://api2.aquifer.bible>. Every request needs an `api-key` header; to get a key, sign up at <https://www.aquifer.bible/apiaccess>.

* [Interactive API explorer](https://api2.aquifer.bible/docs): Interactive browser of every endpoint and its parameters; the OpenAPI document is at <https://api2.aquifer.bible/openapi.json>. No API key is needed to view either.
* [API Reference](api_reference.md): Every endpoint, with conventions (pagination, passage reference notations, content formats, authentication and rate limits), query parameters, response shapes and errors.
* [v1 to v2 Migration Guide](v1_to_v2_migration_guide.md): Field-by-field mapping and behavior changes for consumers moving from the retired v1 API to v2.

## Data formats

Documentation for the JSON that Aquifer resource repositories publish.

* [Aquifer Resource Metadata](aquifer_resource_metadata.md): Information about the JSON used to encode resource-level metadata for Aquifer resources.
* [Aquifer Article Metadata](aquifer_article_metadata.md): Information about the JSON used to encode article-level metadata and content for Aquifer resources.
* [Aquifer Alignments](aquifer_alignments.md): Information about alignment data and suggestions on how to apply the data to textual annotation and display.
* [Lexical Alignments](lexical_alignments.md): Comprehensive documentation on lexical alignment structure and integration, including how alignments relate to source texts (SBLGNT, WLCM) and target dictionary resources.
* [Aquifer Resource Inventory](aquifer_full_inventory.md): A snapshot of the available resources with their citations, copyright and licenses (accessed 2026-01-06).

### JSON schemas

* `schemas/aquifer_resource.schema.json`: Currently version `1.1.2`. The resource metadata JSON file `metadata.json` should conform to the current version of this schema.
* `schemas/aquifer_article.schema.json`: Currently version `1.0.4`. The content JSON files (e.g. `01.content.json` for canonically-ordered resources, `001.content.json` for alphabetically-ordered resources, and `000001.content.json` for others) should conform to the current version of this schema.

## Forthcoming

This list is not in any particular order and order does not imply priority of item.

* Aquifer Linking Specification / Documentation
  * specifying Bible references
  * specifying links to other Aquifer resources
* Images, Audio, and other media
  * currently only have (usable, valid) links to the media item
  * intend to also provide media item within relevant respositories
