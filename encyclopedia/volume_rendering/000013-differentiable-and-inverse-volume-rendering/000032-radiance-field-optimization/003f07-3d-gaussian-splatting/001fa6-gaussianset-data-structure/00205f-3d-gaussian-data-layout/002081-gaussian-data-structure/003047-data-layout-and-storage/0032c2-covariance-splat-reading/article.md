# Covariance Splat Reading

Also called: CovSplat Data Layout and Reading, CovSplat struct, ReadCovSplat, gsplat texture storage, readCovArrays

## Overview
Covariance Splat Reading is a method for decoding a single 3D Gaussian Splat from compressed texture storage into a structured representation suitable for rendering. It solves the problem of efficiently retrieving Gaussian properties (position, color, opacity, and covariance) from a GPU-friendly texture array format. It differs from raw buffer access by utilizing two `usampler2DArray` textures to store packed integer data, requiring specific unpacking logic to reconstruct the continuous covariance matrix components from their discrete storage format [S1, unofficial].

## How it works
**Inputs:**
*   `index`: Integer index of the splat to read.
*   `numSplats`: Total number of splats in the scene.
*   `texture1`: First `usampler2DArray` containing packed Gaussian data.
*   `texture2`: Second `usampler2DArray` containing packed Gaussian data.
*   `splatTexCoord(index)`: Helper function returning the `ivec3` coordinate for a given splat index in the texture arrays.

**Outputs:**
*   `covsplat`: A `CovSplat` struct instance populated with the splat's properties.

**Data Structures:**
The `CovSplat` struct layout is defined as follows [S1, unofficial]:
*   `center`: `vec3` (x, y, z position).
*   `flags`: `uint` (status flags, e.g., active/inactive).
*   `rgba`: `vec4` (color and opacity).
*   `xxyyzz`: `vec3` (diagonal elements of the covariance matrix).
*   `xyxzyz`: `vec3` (off-diagonal elements of the covariance matrix).
*   `index`: `int` (original index).

**Steps:**
1.  Initialize `covsplat.flags` to `0u`.
2.  Validate `index`: If `index < 0` or `index >= numSplats`, set `covsplat.flags` to `0u` and return [S1, unofficial].
3.  Compute texture coordinate: `coord = splatTexCoord(index)`.
4.  Fetch packed data:
    *   `packed1 = texelFetch(texture1, coord, 0)`
    *   `packed2 = texelFetch(texture2, coord, 0)`
5.  Unpack data: Call `unpackSplatExtCov(packed1, packed2, covsplat.center, covsplat.rgba, covsplat.xxyyzz, covsplat.xyxzyz)`. Note: The exact bit-packing and decoding logic for `unpackSplatExtCov` is not provided in the source excerpts [S1, unofficial].
6.  Set active flag: If `covsplat.xxyyzz` and `covsplat.xyxzyz` are not all zero, set `covsplat.flags = GSPLAT_FLAG_ACTIVE`; otherwise, set `covsplat.flags = 0u` [S1, unofficial].
7.  Set `covsplat.index = index`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def read_covariance_splat(index, num_splats, texture1, texture2, splat_tex_coord):
    """
    Reads a single Gaussian splat from texture arrays into a CovSplat struct.
    
    Args:
        index (int): Index of the splat.
        num_splats (int): Total number of splats.
        texture1 (Texture2DArray): First texture array.
        texture2 (Texture2DArray): Second texture array.
        splat_tex_coord (function): Returns ivec3 coordinate for index.
        
    Returns:
        dict: CovSplat struct with fields: center, flags, rgba, xxyyzz, xyxzyz, index.
    """
    covsplat = {
        'center': [0.0, 0.0, 0.0],
        'flags': 0,
        'rgba': [0.0, 0.0, 0.0, 0.0],
        'xxyyzz': [0.0, 0.0, 0.0],
        'xyxzyz': [0.0, 0.0, 0.0],
        'index': index
    }
    
    # Step 2: Validate index
    if index < 0 or index >= num_splats:
        covsplat['flags'] = 0
        return covsplat
        
    # Step 3: Compute coordinate
    coord = splat_tex_coord(index)
    
    # Step 4: Fetch packed data
    # Note: texelFetch returns uvec4 (uint4)
    packed1 = texture1.fetch(coord, 0)
    packed2 = texture2.fetch(coord, 0)
    
    # Step 5: Unpack data
    # The specific unpacking logic is not provided in the source.
    # This step reconstructs center, rgba, xxyyzz, and xyxzyz from packed1 and packed2.
    unpack_splat_ext_cov(packed1, packed2, covsplat['center'], covsplat['rgba'], 
                         covsplat['xxyyzz'], covsplat['xyxzyz'])
    
    # Step 6: Set active flag
    # Check if covariance components are non-zero
    if (covsplat['xxyyzz'][0] != 0 or covsplat['xxyyzz'][1] != 0 or covsplat['xxyyzz'][2] != 0 or
        covsplat['xyxzyz'][0] != 0 or covsplat['xyxzyz'][1] != 0 or covsplat['xyxzyz'][2] != 0):
        covsplat['flags'] = GSPLAT_FLAG_ACTIVE
    else:
        covsplat['flags'] = 0
        
    # Step 7: Set index
    covsplat['index'] = index
    
    return covsplat
```

**Parameters:**
*   `GSPLAT_FLAG_ACTIVE`: Constant indicating an active splat. Value not specified in sources [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Out of Bounds:** If `index` is invalid, the function returns an inactive splat with `flags=0` [S1, unofficial].
*   **Null Inputs:** If `covSplats` or `index` are null/missing in the JS wrapper, `covsplat.flags` is explicitly set to `0u` [S1, unofficial].
*   **Missing Unpack Logic:** The source code excerpt ends abruptly after fetching `packed2`, implying the decoding step (`unpackSplatExtCov`) is missing or external [S1, unofficial]. Implementers must derive or locate the specific bit-packing format used for `packed1` and `packed2`.

## Operating characteristics
*   **Time Complexity:** O(1) per splat read, assuming texture fetch and unpacking are constant time operations.
*   **Memory Complexity:** O(1) auxiliary memory for the `CovSplat` struct.
*   **Strengths:** Efficient GPU-side decoding from compressed texture storage; avoids CPU-GPU synchronization for individual splat reads.
*   **Limitations:** Requires custom unpacking logic not provided in the source; dependent on specific texture coordinate mapping (`splatTexCoord`).
*   **When to Use:** When rendering 3D Gaussian Splats from texture-based storage formats (e.g., SPZ or custom compressed formats) on the GPU.
*   **When Not to Use:** When using raw buffer layouts (SoA) directly accessible in memory without texture indirection.

## Background
The method is part of the `sparkjsdev/spark` project, an unofficial implementation of 3D Gaussian Splatting tools [S1, unofficial]. It is designed to work with compressed splat formats (e.g., SPZ) and integrates with WebAssembly (WASM) and Rust-based rendering pipelines. The `CovSplat` struct is a specific data layout optimized for reading covariance representations from texture arrays, differing from the standard SoA buffer layout by using packed integer textures for storage [S1, unofficial].

## Sources
[S1] sparkjsdev/spark, https://github.com/sparkjsdev/spark, fetched 2026-10-27 (unofficial)
