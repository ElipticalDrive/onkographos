# Packed Splat Reading

Also called: ReadPackedSplat, readPackedArray

## Overview
Packed Splat Reading is a method to decode a single 3D Gaussian Splat from a packed texture representation, extracting its geometric and color properties into a `Gsplat` struct [S1, unofficial]. It serves as the primary accessor for the standard Gaussian Splatting texture layout, distinguishing itself from extended variants by utilizing a single texture array and a specific unpacking routine for standard spherical harmonics or color data [S1, unofficial].

## How it works
**Inputs:**
- `packedSplats`: A struct containing:
  - `textureArray`: `usampler2DArray` holding the packed splat data.
  - `numSplats`: `int` total number of splats.
  - `rgbMinMaxLnScaleMinMax`: `vec4` parameters used during decoding (specific usage defined in `unpackSplatEncoding`).
  - `lodOpacity`: `bool` flag to adjust alpha for level-of-detail rendering.
- `index`: `int` the index of the splat to read.

**Outputs:**
- `gsplat`: A `Gsplat` struct with the following layout:
  - `center`: `vec3`
  - `flags`: `uint`
  - `scales`: `vec3`
  - `index`: `int`
  - `quaternion`: `vec4`
  - `rgba`: `vec4`

**Constants:**
- `GSPLAT_FLAG_ACTIVE`: Defined as `1u << 0u` [S1, unofficial].

**Steps:**
1. Initialize `gsplat.flags` to `0u` [S1, unofficial].
2. Validate the input `index`. If `index < 0` or `index >= numSplats`, the method returns the initialized `gsplat` with `flags = 0u` and no data read [S1, unofficial].
3. Calculate the texture coordinate using `splatTexCoord(index)` [S1, unofficial].
4. Fetch the packed data from `textureArray` at the calculated coordinate and mip level 0:
   $$ \text{packedData} = \text{texelFetch}(\text{textureArray}, \text{splatTexCoord(index)}, 0) $$
   where `packedData` is a `uvec4` [S1, unofficial].
5. Decode `packedData` into the `gsplat` fields using the helper function `unpackSplatEncoding`:
   $$ \text{unpackSplatEncoding}(\text{packedData}, \text{gsplat.center}, \text{gsplat.scales}, \text{gsplat.quaternion}, \text{gsplat.rgba}, \text{rgbMinMaxLnScaleMinMax}) $$
   [S1, unofficial].
6. If `lodOpacity` is true, double the alpha component:
   $$ \text{gsplat.rgba.a} = 2.0 \times \text{gsplat.rgba.a} $$
   [S1, unofficial].
7. Check for zero size by comparing `gsplat.scales` to `vec3(0.0, 0.0, 0.0)`:
   $$ \text{zeroSize} = \text{all}(\text{equal}(\text{gsplat.scales}, \text{vec3}(0.0, 0.0, 0.0))) $$
   [S1, unofficial].
8. Set `gsplat.flags`:
   - If `zeroSize` is true, `gsplat.flags = 0u`.
   - Otherwise, `gsplat.flags = GSPLAT_FLAG_ACTIVE` [S1, unofficial].
9. Set `gsplat.index` to the input `index` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def read_packed_splat(packed_splats, index):
    # Initialize output struct
    gsplat = Gsplat()
    gsplat.flags = 0u
    gsplat.index = index

    # Boundary check
    if index < 0 or index >= packed_splats.numSplats:
        return gsplat

    # Fetch packed data
    coord = splatTexCoord(index)
    packed_data = texelFetch(packed_splats.textureArray, coord, 0)

    # Decode fields
    unpack_splat_encoding(
        packed_data,
        gsplat.center,
        gsplat.scales,
        gsplat.quaternion,
        gsplat.rgba,
        packed_splats.rgbMinMaxLnScaleMinMax
    )

    # LOD opacity adjustment
    if packed_splats.lodOpacity:
        gsplat.rgba.a = 2.0 * gsplat.rgba.a

    # Determine active flag based on scale
    zero_size = all_equal(gsplat.scales, vec3(0.0, 0.0, 0.0))
    if zero_size:
        gsplat.flags = 0u
    else:
        gsplat.flags = 1u << 0u  # GSPLAT_FLAG_ACTIVE

    return gsplat
```

**Edge Cases and Pitfalls:**
- If `index` is out of bounds, `flags` are set to `0u` and no data is read [S1, unofficial].
- If `packedSplats` inputs are missing or invalid, `flags` are set to `0u` and no data is read [S1, unofficial].

## Subtopics

### Extended Splat Reading
A variant that reads a single 3D Gaussian Splat from two extended texture arrays, decoding properties into a `Gsplat` struct [S1, unofficial]. It differs from the standard method by using `textureArray1` and `textureArray2` and the `unpackSplatExt` helper function [S1, unofficial]. It is chosen when the splat data requires more bits than the standard packing allows, typically for higher-order spherical harmonics or extended color representations [S1, unofficial]. Input struct `ExtSplats` contains `usampler2DArray textureArray1`, `usampler2DArray textureArray2`, and `int numSplats` [S1, unofficial]. Pitfalls include returning `flags = 0u` if `index` is out of bounds or inputs are missing [S1, unofficial].

## Operating characteristics
Time complexity is $O(1)$ per splat read, dominated by texture fetch and unpacking operations [S1, unofficial]. Memory complexity is determined by the packed texture size, which is optimized for cache locality in GPU rendering pipelines [S1, unofficial]. Strengths include efficient random access to splat data in fragment shaders [S1, unofficial]. Limitations include the fixed bit-depth of the packing scheme, which may not support all extended attributes without the extended variant [S1, unofficial]. Use when rendering standard 3D Gaussian Splatting scenes; do not use when extended spherical harmonics or custom attributes require the extended layout [S1, unofficial].

## Background
Originating from the implementation details of the Spark 3D viewer and related Gaussian Splatting libraries, this method provides the runtime accessor for texture-packed splat data [S1, unofficial]. It is used in real-time rendering engines to decode splat attributes for rasterization or ray marching [S1, unofficial].

## Sources
[S1] sparkjsdev/spark, https://github.com/sparkjsdev/spark, fetched 2026-10-27 (unofficial)
