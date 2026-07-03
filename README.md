# mach-gltf

A [glTF 2.0](https://registry.khronos.org/glTF/) loader written in pure Mach:
no C, no Assimp, no FFI. It reads both container forms — `.gltf` (a JSON
document with external buffers) and `.glb` (the binary container) — and
produces engine-agnostic, in-memory scene data. Project id is `gltf`, so
consumers reach everything as `gltf.*`.

```mach
use gltf;

# a .glb file opens with a 12-byte header carrying the container magic and
# version; validate it before trusting the chunks that follow.
fun accepts(h: *gltf.Header) bool {
    ret gltf.is_valid(h);
}
```

Consuming projects vendor the loader as a normal Mach dependency:

```toml
[deps.mach-gltf]
git = "https://github.com/briar-systems/mach-gltf"
ref = "branch/main"
```

## Scope

Loading a glTF 2.0 asset into memory, covering the full core feature set:

- Both container forms: `.gltf` (JSON + external `.bin`/data-URI buffers) and
  `.glb` (12-byte header, JSON chunk, optional BIN chunk).
- Meshes: primitives, accessors, buffer views, index and vertex attributes.
- Materials: PBR metallic-roughness, plus the base texture/normal/occlusion/
  emissive set and their samplers.
- Scene hierarchy: nodes, local transforms (TRS or matrix), and the scene
  graph.
- Skins: joints, inverse-bind matrices, skeleton roots.
- Animations: keyframe samplers and channels (translation, rotation, scale,
  and weights) with their interpolation modes.
- Morph targets: per-primitive target attributes and node/mesh weights.

The output is plain in-memory data. The animation **runtime** — pose sampling,
blending, and skinning — is engine code that consumes this data; it is
deliberately out of scope here so the loader stays engine-agnostic.

## Non-goals

- Other formats. FBX, OBJ, COLLADA, USD, and the rest are not in scope; the
  ecosystem answer is to export to glTF.
- Writing or exporting. This is a loader, not a serializer.
- Rasterizing or decoding textures. glTF references image bytes (PNG/JPEG or a
  KTX2/`data:` URI); turning those into pixels is
  [mach-image](https://github.com/briar-systems/mach-image)'s job, a sibling
  library. mach-gltf hands back the image references and lets the caller decode.

## Dependencies

JSON parsing is a prerequisite: the `.gltf` document and the `.glb` JSON chunk
are both JSON, and it comes from the standard library
([`std.data.json`](https://github.com/briar-systems/mach-std)) rather than an
in-tree parser. `std.data.json` now parses the full RFC 8259 number grammar, so
glTF's floating-point values — node transforms, accessor bounds, and material
factors — load directly; the lockfile pins a `mach-std` commit carrying that
support.

## Status

The structural loader is in place: the `.glb` container, the typed document
model for every core top-level array, JSON parsing into that model (integer and
float fields), and binary accessor readers over buffer bytes. What is not yet
implemented: reading `.gltf` (external `.bin` and `data:` URI buffers) — only
the `.glb` JSON chunk and BIN buffer are wired today — and the animation runtime
(deliberately out of scope, see above).

## Architecture

```
src/
  bytes.mach     little-endian scalar read/write primitives over byte buffers
  glb.mach       the .glb binary container: header, chunk iteration, validation
  doc.mach       the typed glTF 2.0 document model and accessor-layout helpers
  parse.mach     JSON to document model, plus the load_glb convenience entry
  accessor.mach  typed, bounds-checked reads of accessor data out of buffer bytes
  gltf.mach      library surface: re-exports every public symbol under `gltf.*`
```

The surface (`gltf.mach`) is what `[project].module = "gltf.mach"` binds, so a
bare `use gltf;` reaches the whole API. It also carries `use std.runtime;` so a
library `mach test` links a runnable binary. New modules (a `.gltf` reader)
forward through this surface as they land.

## Multiplatform

The loader is pure parsing — no system libraries, no display, no OS calls — so
it builds for every target the Mach toolchain can emit. Run `mach info targets`
for the authoritative matrix of the compiler in use; the current prime set is
x86-64 and aarch64 across Linux, macOS, and Windows (Windows is x86-64 only, as
COFF carries no aarch64 relocations), with riscv64-linux as a cross-compile
target.

## Tests

`test` blocks live beside the code they cover and are display-free. They pin the
container tags and header layout against the glTF 2.0 specification, build
synthetic `.glb` fixtures as bytes in test code — integer and float JSON with a
BIN chunk of known `float32` data — and assert the parsed document structure,
exact accessor reads, and rejection of malformed containers and out-of-bounds
accessors. Everything runs under `mach test .` with no external fixtures or
system dependencies.
