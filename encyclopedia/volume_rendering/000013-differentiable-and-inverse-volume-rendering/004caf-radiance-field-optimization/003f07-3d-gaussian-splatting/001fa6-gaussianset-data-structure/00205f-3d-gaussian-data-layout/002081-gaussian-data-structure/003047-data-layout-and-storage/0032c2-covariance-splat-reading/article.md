# Covariance Splat Reading

Also called: CovSplat Data Layout and Reading, CovSplat struct, ReadCovSplat, gsplat texture storage, readCovArrays

## Overview
Covariance Splat Reading decodes a single 3D Gaussian Splat from two `usampler2DArray` textures into a `CovSplat` structure [S1, unofficial]. It retrieves position, color, opacity, and covariance matrix components from GPU texture storage during rendering [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** `index` (integer), `numSplats` (integer), `texture1` (`usampler2DArray`), `texture2` (`usampler2DArray`), `splatTexCoord(index)` (returns `ivec3`) [S1, unofficial].
- **Output:** `covsplat` (`CovSplat` struct) [S1, unofficial].

**Data Structures:**
The `CovSplat` struct contains `flags` (`uint`), `index` (`int`), `rgba` (`vec4`), `center` (`vec3`), `xxyyzz` (`vec3`), and `xyxzyz` (`vec3`) [S1, unofficial]. The `xxyyzz` and `xyxzyz` fields store the diagonal and off-diagonal elements of the 3×3 covariance matrix, derived from a scale-quaternion representation [S1, unofficial]. Covariance data is stored in two `usampler2DArray` textures and fetched as two `uvec4` values (`packed1`, `packed2`) [S1, unofficial].

**Steps:**
1. Initialize `covsplat.flags` to `0u` [S1, unofficial].
2. Validate `index`: if `index < 0` or `index >= numSplats`, set `covsplat.flags` to `0u` and return [S1, unofficial].
3. Compute `coord = splatTexCoord(index)` [S1, unofficial].
4. Fetch `packed1 = texelFetch(texture1, coord, 0)` [S1, unofficial].
5. Fetch `packed2 = texelFetch(texture2, coord, 0)` [S1, unofficial].
6. Decode properties by calling `unpackSplatExtCov(packed1, packed2, covsplat.center, covsplat.rgba, covsplat.xxyyzz, covsplat.xyxzyz)` [S1, unofficial].
7. Set `covsplat.flags` to `GSPLAT_FLAG_ACTIVE` if `covsplat.xxyyzz` and `covsplat.xyxzyz` are not all zero; otherwise set to `0u` [S1, unofficial].
8. Set `covsplat.index = index` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
GSPLAT_FLAG_ACTIVE = 1  # Value not given in the sources [S1, unofficial]

def read_cov_splat(index, num_splats, texture1, texture2, splat_tex_coord):
    covsplat = {
        'flags': 0,
        'index': index,
        'rgba': [0.0, 0.0, 0.0, 0.0],
        'center': [0.0, 0.0, 0.0],
        'xxyyzz': [0.0, 0.0, 0.0],
        'xyxzyz': [0.0, 0.0, 0.0]
    }
    if index < 0 or index >= num_splats:
        covsplat['flags'] = 0
        return covsplat
    coord = splat_tex_coord(index)
    packed1 = texture1.fetch(coord, 0)
    packed2 = texture2.fetch(coord, 0)
    unpack_splat_ext_cov(packed1, packed2, covsplat['center'], covsplat['rgba'], covsplat['xxyyzz'], covsplat['xyxzyz'])
    if (covsplat['xxyyzz'][0] != 0 or covsplat['xxyyzz'][1] != 0 or covsplat['xxyyzz'][2] != 0 or
        covsplat['xyxzyz'][0] != 0 or covsplat['xyxzyz'][1] != 0 or covsplat['xyxzyz'][2] != 0):
        covsplat['flags'] = GSPLAT_FLAG_ACTIVE
    else:
        covsplat['flags'] = 0
    return covsplat

def unpack_splat_ext_cov(packed1, packed2, center, rgba, xxyyzz, xyxzyz):
    # The exact bit-packing scheme and decoding logic for packed1/packed2 into center, rgba, xxyyzz, xyxzyz
    # is not provided in the sources [S1, unofficial].
    pass
```

**Parameters:**
- `GSPLAT_FLAG_ACTIVE`: Constant indicating an active splat; value not given in the sources [S1, unofficial].
- `unpackSplatExtCov`: Helper function that decodes `packed1` and `packed2` into the `CovSplat` fields; the internal bit-packing logic is not provided in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
- If the index is out of bounds or invalid, the `CovSplat` flags are set to `0u`, marking it as inactive [S1, unofficial].
- If `covSplats` or `index` inputs are missing or null in the JS wrapper, the output `covsplat.flags` is explicitly set to `0u` [S1, unofficial].
- The source excerpt ends abruptly after fetching `packed2`, implying the decoding step is missing from the provided text [S1, unofficial].

## Operating characteristics
- Time and memory complexity: Not given in the sources.
- Measured performance, scaling, strengths, limitations, and usage conditions: Not given in the sources.

## Background
The method is part of the `sparkjsdev/spark` project, which provides tools for compressing and viewing Gaussian Splats [S1, unofficial].

## Sources
[S1] sparkjsdev/spark, https://github.com/sparkjsdev/spark, fetched 2026-10-27 (unofficial)
