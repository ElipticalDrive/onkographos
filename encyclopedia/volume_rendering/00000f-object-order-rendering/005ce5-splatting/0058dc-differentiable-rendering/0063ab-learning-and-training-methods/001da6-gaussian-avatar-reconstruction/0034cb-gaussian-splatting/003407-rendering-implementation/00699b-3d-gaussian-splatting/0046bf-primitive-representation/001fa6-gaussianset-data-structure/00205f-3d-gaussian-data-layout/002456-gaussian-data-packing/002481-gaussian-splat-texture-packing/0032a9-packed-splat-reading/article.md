# Packed Splat Reading

Also called: ReadPackedSplat, readPackedArray

## Overview
Packed Splat Reading is a method to read a single 3D Gaussian Splat from a packed texture representation, decoding its geometric and color properties into a `Gsplat` struct [S1, unofficial]. It distinguishes itself from extended variants by utilizing a single `usampler2DArray` texture and a standard unpacking routine for the packed data [S1, unofficial].

## How it works
**Inputs:**
- `packedSplats`: A struct containing:
  - `textureArray`: `usampler2DArray` holding the packed splat data.
  - `numSplats`: `int` total number of splats.
  - `rgbMinMaxLnScaleMinMax`: `vec4` parameters used during decoding.
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
- `GSPLAT_FLAG_ACTIVE`: Defined as `1 << 0` [S1, unofficial].

**Steps:**
1. Initialize `gsplat.flags` to `0` [S1, unofficial].
2. Validate the input `index`. If `index < 0` or `index >= numSplats`, the method returns the initialized `gsplat` with `flags = 0` and no data read [S1, unofficial].
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
   - If `zeroSize` is true, `gsplat.flags = 0`.
   - Otherwise, `gsplat.flags = GSPLAT_FLAG_ACTIVE` [S1, unofficial].
9. Set `gsplat.index` to the input `index` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def read_packed_splat(packed_splats, index):
    # Initialize output struct
    gsplat = Gsplat()
    gsplat.flags = 0

    # Boundary check
    if index < 0 or index >= packed_splats.numSplats:
        return gsplat

    # Calculate texture coordinate for index
    # coord = splatTexCoord(index)  # Not given in the sources [S1, unofficial]

    # Fetch packed data from texture at coordinate and mip level 0
    # packed_data = texelFetch(packed_splats.textureArray, coord, 0)  # Standard GLSL texture fetch [G]

    # Decode packed data into gsplat fields
    # unpackSplatEncoding(packed_data, gsplat.center, gsplat.scales, gsplat.quaternion, gsplat.rgba, packed_splats.rgbMinMaxLnScaleMinMax)  # Not given in the sources [S1, unofficial]

    # LOD opacity adjustment
    if packed_splats.lodOpacity:
        gsplat.rgba.a = 2.0 * gsplat.rgba.a

    # Determine active flag based on scale
    # zero_size = all_equal(gsplat.scales, (0.0, 0.0, 0.0))  # Not given in the sources [S1, unofficial]
    # if zero_size:
    #     gsplat.flags = 0
    # else:
    #     gsplat.flags = 1 << 0  # GSPLAT_FLAG_ACTIVE

    # Set index
    gsplat.index = index

    return gsplat
```

**Edge Cases and Pitfalls:**
- If `index` is out of bounds, `flags` are set to `0` and no data is read [S1, unofficial].
- If `packedSplats` inputs are missing or invalid, `flags` are set to `0` and no data is read [S1, unofficial].

## Subtopics

### Extended Splat Reading
A variant that reads a single 3D Gaussian Splat from two extended texture arrays, decoding properties into a `Gsplat` struct [S1, unofficial]. It differs from the standard method by using `textureArray1` and `textureArray2` and the `unpackSplatExt` helper function [S1, unofficial]. Input struct `ExtSplats` contains `usampler2DArray textureArray1`, `usampler2DArray textureArray2`, and `int numSplats` [S1, unofficial]. Pitfalls include returning `flags = 0` if `index` is out of bounds or inputs are missing [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] sparkjsdev/spark, https://github.com/sparkjsdev/spark, fetched 2026-10-27 (unofficial)
