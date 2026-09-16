# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Changed
- ci: CI runs the family pipeline (`briar-systems/.github` `mach-lib.yml`) on the pinned, checksum-verified mach seed: debug and release build and test, `mach fmt --check` and an all-targets release build on x86_64-linux for pull requests into dev, plus native aarch64-linux, windows and darwin legs for pull requests into main. A `gate` job is the one required check.
- **Builds with Mach 5.0 and std 2.1.** The dependency is `[dep.std]`, pinned by the committed `dep/std` gitlink, and `mach.lock` is gone. Every profile states its full field set and the linux-x86_64 target, debug profile and library artifact are the defaults.
- **Failures are closed error tags, not strings.** `Result[T, str]` becomes `res[T, E]`: `read_header`, `chunk_iter` and `parse_glb` fail with `ContainerError`; `parse_document` with `DocumentError` (`json`, `alloc`, `version`, `missing: Site`, `invalid: Site`); `load_glb` with `LoadError` (`container`, `document`); `plan` with `PlanError`; `read_f32`, `read_u32`, `read_u16` and `read_u8` with `ReadError` (`component_type`, `capacity`).
- **`chunk_next(it)` returns `res[opt[Chunk], ContainerError]`** instead of filling an out parameter and returning a boolean.
- **doc: the `NONE` sentinel is gone.** Indices are `usize`, and an optional reference (`scene`, `mesh`, `skin`, `camera`, `indices`, `material`, `buffer_view`, `source`, `sampler`, `skeleton`, `inverse_bind_matrices`, `target_node`) is `opt[usize]`. A material texture reference is `opt[TextureRef]`.
- **doc: every glTF enumeration is a tag.** `ComponentType`, `AccessorType`, `PrimitiveMode`, `Interpolation`, `TargetPath` and `AlphaMode` replace their integer aliases and `COMPONENT_*`, `TYPE_*`, `MODE_*`, `INTERP_*`, `PATH_*` and `ALPHA_*` constants, and the new `BufferTarget`, `MagFilter`, `MinFilter` and `WrapMode` type the buffer view target and sampler fields. `component_count` and `component_size` are total and no longer return 0.
- **parse: the input is held to the specification's shapes.** A required member that is absent is `missing`, and a wrong JSON type, a negative index, a non-integer index or a code outside its enumeration is `invalid`, where it used to fall back to a default or `NONE`. A document whose `asset.version` is not 2.x, or whose `minVersion` is above 2.0, is refused with `version`.

### Fixed
- String members are decoded. Names, URIs and MIME types used to be copied with their JSON escapes intact, so `"a\/b"` read back as `a\/b`.

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
