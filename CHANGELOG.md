# Changelog

## [Unreleased]

### Fixed

- A query with both `title` and `text` searches the words of both in the titles, where `text`
  was dropped without a word.
- A `language` that is not an official language of the Union is an `InvalidConfigException` (an
  `\InvalidArgumentException` still), which `Registry::usable()` leaves out.
- `content` given as a string reads `"false"` and `"0"` as false (`Config::bool()`), where
  `(bool) "false"` fetched the text. Requires omnisense/omnilex with `Config::bool()`.
