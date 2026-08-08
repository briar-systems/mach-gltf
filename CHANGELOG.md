# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.3.0] - 2026-08-07

A `Document` now owns its strings, so it no longer borrows the JSON buffer it
was parsed from.

### Changed
- **doc: every string field is an owned, null-terminated `str` instead of a borrowed `StrView`** (#7, #8). Nothing in a parsed `Document` points into the source JSON, so a caller may free or reuse those bytes the moment `parse_document` returns. Every std string function now applies directly to a name or URI, and `view_eq` — the byte comparison this library hand-rolled because `str` functions could not reach a view — is gone in favour of `str_equals`. An absent member is an owned empty string rather than nil, so consumers never nil-check before comparing or formatting.
- parse: the eight parse functions that newly allocate (`parse_asset`, `parse_buffer`, `parse_buffer_view`, `parse_accessor`, `parse_material`, `parse_texture`, `parse_image`, `parse_sampler`) take the allocator and return `Result`, matching the rule the file already followed for the array parsers.
- manifest: Re-touched `mach.toml` to RFC-exact totality per mach#1964/mach#1979.

### Fixed
- Builds against mach-std 0.25.x. `StrView` moved to `std.types.view.View` in mach-std `25b7ba0`; the borrowed design is what turned that move into a breakage, so this release removes the coupling rather than chasing the rename.

### Lifetime discipline, unchanged
An arena freed wholesale, matching `std.data.json`'s own Value tree. There is no
per-field teardown to call and none was added.

### Verification
- 26/26 tests against mach-std 0.25.1.
- A new test parses a document, overwrites **every byte** of the source buffer with `0xAA`, then reads the strings back. It fails against the borrowed implementation (23 passed, 3 failed) and passes against this one, so it is not vacuous.

## [0.2.0] - 2026-07-07

Updates the project's manifest (`mach.toml`) to the V2 manifest format and points dependencies to git URLs.

### Changed
- manifest: Migrated manifest layout and dependencies to comply with the V2 manifest spec.
