# Aquifer Documentation

Documentation for JSON schemas and other material

## Available Documentation

* [Aquifer Resource Metadata](aquifer_resource_metadata.md): Information about the JSON used to encode resource-level metadata for Aquifer resources
* [Aquifer Article Metadata](aquifer_article_metadata.md): Information about the JSON used to encode article-level metadata and content for Aquifer resources.

## JSON schemas

* `schemas/aquifer_resource.schema.json`: Currently version `1.0.0`. The resource metadata JSON file `metadata.json` should conform to the current version of this schema.
* `schemas/aquifer_article.schema.json`: Currently version `1.0.0`. The content JSON files (e.g. `01.content.json` for canonically-ordered resources, `000001.content.json` for others) should conform tot he current version of this schema.

## Forthcoming

This list is not in any particular order and order does not imply priority of item.

* Aquifer Linking Specification / Documentation
  * specifying Bible references
  * specifying links to other Aquifer resources
  * specifying ACAI links
* Aquifer Resource Registry
* Images, Audio, and other media
  * currently only have (usable, valid) links to the media item
  * intend to also provide media item within relevant respositories