# D-Sdf

D bindings for [sxyu/sdf](https://github.com/sxyu/sdf) — a fast mesh-to-signed-distance-field library written in C++.

Exposes a thin `extern "C"` shim (`include/sdf_c.h`) so D code can build and query
SDFs without touching C++ or Eigen directly. The library links statically — no shared
sdf or Eigen objects appear in the final binary.

## Layout

```
include/sdf_c.h         C shim API (opaque handle + 4 entry points)
csrc/sdf_c.cpp          C++ implementation of the shim
source/sdf/c.d          D module — extern(C) 1:1 with sdf_c.h
examples/sdf_cube.d     Smoke-test: unit cube SDF query + assertions
CMakeLists.txt          Builds sdf_c static library
dub.json                dub package definition
extern/sdf              git submodule → sxyu/sdf (pinned SHA below)
extern/eigen            git submodule → Eigen 3.4.0 (header-only)
```

## Pinned upstream SHAs

| Submodule      | URL                                     | SHA                                        |
|----------------|-----------------------------------------|--------------------------------------------|
| `extern/sdf`   | https://github.com/sxyu/sdf             | `1923465258c101bccd39f8115ffe19e9a8eea70c` |
| `extern/eigen` | https://gitlab.com/libeigen/eigen.git   | `3147391d946bb4b6c68edd901f2add6ac1f31f8c` |

`sxyu/sdf` has no release tags — the SHA above is the latest `master` commit at
the time this binding was created. `extern/eigen` is the Eigen 3.4.0 release tag.

**Eigen wiring:** sxyu/sdf does not vendor Eigen; it expects `<Eigen/Core>` etc. on
the include path. D-Sdf supplies Eigen via `extern/eigen` (the Eigen repo root is the
include path — headers live at `extern/eigen/Eigen/Core`, etc.). `SDF_USE_SYSTEM_EIGEN`
is intentionally not used; our CMakeLists.txt sets `EIGEN_INCLUDE_DIR` directly.

## First build

Prerequisites: `git`, `cmake >= 3.16`, `g++ / clang++` (C++14), `dmd` or `ldc2`.

```bash
git clone <this-repo> D-Sdf
cd D-Sdf
# Build the library configuration (compiles and links sdf_c static archive):
dub build

# Build and run the sdf_cube smoke-test:
dub run --config=sdf_cube
```

`dub` will automatically:
1. Run `git submodule update --init --recursive` to populate `extern/sdf` and `extern/eigen`.
2. Configure and build the `sdf_c` static library via CMake.
3. Compile and link the D source against `build/libsdf_c.a`.

## API summary

```d
import sdf.c;

// Build
sdfc_t* h = sdfc_build(verts.ptr, numVerts, tris.ptr, numTris, /*robust=*/1);
scope(exit) sdfc_free(h);

// Distance (negative = inside, positive = outside)
float dist;
sdfc_distance(h, point.ptr, 1, &dist);

// Containment (1 = inside, 0 = outside)
int inside;
sdfc_contains(h, point.ptr, 1, &inside);
```

## Sign convention

`sdfc_distance` returns **negative** values inside the mesh and **positive** values
outside. This is the standard signed-distance-function convention and is the opposite
of sxyu/sdf's native `operator()` convention (positive inside). The shim negates
the output transparently.

## License

The D shim (this repository) is released under the BSD 2-Clause License — see `LICENSE`.

[sxyu/sdf](https://github.com/sxyu/sdf) is Copyright Alex Yu 2020, BSD 2-Clause License.

[Eigen](https://eigen.tuxfamily.org) is MPL-2.0 licensed (header-only; no compiled
Eigen code is linked into the final binary).
