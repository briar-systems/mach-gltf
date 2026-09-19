# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.5.0] - 2026-09-19

Builds against std 5.7.1 and releases through the family CD workflow. Requires mach 5.5.2 or later.

### Changed
- **Builds against std 5.7.1** (#33). `[dep.std]` moves from `tag/v4.0.0` to `tag/v5.7.1`, so this library resolves alongside a root project that pins std 5.x. Verified from a clean build: no source change was needed, since nothing here uses the clocks, cancellation scopes or `buffers.Source` that std 5 changed. `[project].mach` rises from `^5.3` to `^5.5.2`, which is what std 5.7.1 itself requires, and no higher.
- ci: the tag-triggered workflow file is `cd.yml`, the family-wide name (#31). Its content and `Release` name are unchanged.
- ci: release runs are serialized per tag with a `concurrency` group, as the shared release workflow now requires, so a duplicate tag-push delivery waits and then finds the release already published (#29).
- manifest: `[project]` declares the compiler range `mach = "^5.3"`, so mach 5.3 and later stop warning on every build (#27). mach 5.2.x rejects the key, so building now needs mach 5.3 or later.
- license: copyright is attributed to Briar Systems LLC (#25). The MIT terms are unchanged.
- ci: a pushed `v*` tag is released by the family's shared release workflow (`briar-systems/.github` `mach-release.yml`). It checks the tag against the manifest version and the changelog, runs every CI leg, then publishes the GitHub release with that version's changelog section as notes (#23). A manual dispatch rehearses the same path without a tag.

## [0.4.1] - 2026-09-16

Builds against std 4.0.0. Requires mach 5.2.0 or later.

### Changed
- **Builds against std 4.0.0, which requires mach 5.2.0 or later** (#19). `[dep.std]` moves from `tag/v3.2.0` to `tag/v4.0.0`. None of the names std 4.0.0 removed are used here, and no std type in this library's API changed, so no source changes were needed.

## [0.4.0] - 2026-09-16

Builds with Mach 5.0 and std 3.2.0, and every failure is a closed error tag. This is a breaking release: error results, optional references and glTF enumerations all change type.

### Added
- manifest: `linux-arm64` and `darwin-aarch64` targets, so the native aarch64 hosts build and test for themselves instead of falling back to linux-x86_64.

### Changed
- ci: CI runs the family pipeline (`briar-systems/.github` `mach-lib.yml`) on the pinned, checksum-verified mach seed: debug and release build and test, `mach fmt --check` and an all-targets release build on x86_64-linux for pull requests into dev, plus native aarch64-linux, windows and darwin legs for pull requests into main. A `gate` job is the one required check.
- **Builds against std 3.2.0** (#15). `[dep.std]` moves from `tag/v2.1.0` to `tag/v3.2.0`. No source changes were needed since this library does no io.
- **Builds with Mach 5.0.** The dependency is `[dep.std]`, pinned by the committed `dep/std` gitlink, and `mach.lock` is gone. Every profile states its full field set and the linux-x86_64 target, debug profile and library artifact are the defaults.
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
