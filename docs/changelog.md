---
currentSection: overview
currentItem: changelog
pageflow_prev_url: multivariant.html
pageflow_prev_text: Multi-Variant
pageflow_next_url: contributing.html
pageflow_next_text: Contributing
---
# CHANGELOG

## develop branch

Nothing yet.

## v1.2016060501

Released Sun 5th June 2016.

### New

* Added support for mapping types onto method names:
  - added `TypeMapper` interface
  - added `MapDuckTypeToMethodName` class
  - added `MapStrictTypeToMethodName` class
* Added support for improving performance of type mapping using intelligent caching dispatch tables:
  - added `DispatchTable` interface
  - added `AllPurposeDispatchTable` class
  - added `ObjectsOnlyDispatchTable` class
  - added `TypeOnlyDispatchTable` class
