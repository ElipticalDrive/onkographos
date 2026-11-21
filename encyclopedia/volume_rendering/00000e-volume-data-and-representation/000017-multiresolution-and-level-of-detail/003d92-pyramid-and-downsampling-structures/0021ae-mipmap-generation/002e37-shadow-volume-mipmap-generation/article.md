# Shadow Volume Mipmap Generation
Also called: Bitwise OR Mipmap Accumulation

## Overview
Shadow Volume Mipmap Generation constructs a 3-level mipmap chain for a shadow volume texture using bitwise OR accumulation to preserve shadow presence across downsampled resolutions [S1, unofficial]. It sets itself apart from standard 3D mipmap generation by using logical OR instead of averaging or min/max filtering [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D shadow volume texture (mip0) of dimensions $W_0 \times H_0 \times D_0$ [S1, unofficial].
*   **Output:** A 3D texture object with a mipmap chain (mip0, mip1, mip2) [S1, unofficial].

**Data Structures:**
*   **Shadow Volume Mip Level:** A 1D array of `uint8_t` values representing the 3D volume [S1, unofficial].
*   **Indexing Formula:** For a volume of dimensions $W \times H \times D$, the linear index for coordinates $(x, y, z)$ is:
    $$ \text{index} = x + W \times (y + H \times z) $$
    where $0 \le x < W$, $0 \le y < H$, and $0 \le z < D$ [S1, unofficial].

**Algorithm Steps:**
1.  **Initialize Mip0:** Ensure the base level (mip0) is populated with the raw shadow volume data [S1, unofficial].
2.  **Generate Mip1:**
    *   Calculate dimensions: $W_1 = W_0 / 2$, $H_1 = H_0 / 2$, $D_1 = D_0 / 2$ [S1, unofficial].
    *   Calculate volume size: $V_1 = W_1 \times H_1 \times D_1$ [S1, unofficial].
    *   Allocate memory for `shadow_volume_mip1` of size $V_1$ and initialize all bytes to 0 [S1, unofficial].
    *   Iterate through all voxels in mip0:
        *   For each $x \in [0, W_0)$, $y \in [0, H_0)$, $z \in [0, D_0)$:
            *   Calculate mip0 index: $idx_0 = x + W_0 \times (y + H_0 \times z)$ [S1, unofficial].
            *   Calculate mip1 target index: $idx_1 = (x / 2) + W_1 \times ((y / 2) + H_1 \times (z / 2))$ [S1, unofficial].
            *   If $idx_1 < V_1$, perform bitwise OR accumulation: `shadow_volume_mip1[idx_1] |= shadow_volume_mip0[idx_0]` [S1, unofficial].
3.  **Generate Mip2:**
    *   Calculate dimensions: $W_2 = W_1 / 2$, $H_2 = H_1 / 2$, $D_2 = D_1 / 2$ [S1, unofficial].
    *   Calculate volume size: $V_2 = W_2 \times H_2 \times D_2$ [S1, unofficial].
    *   Allocate memory for `shadow_volume_mip2` of size $V_2$ and initialize all bytes to 0 [S1, unofficial].
    *   Iterate through all voxels in mip1:
        *   For each $x \in [0, W_1)$, $y \in [0, H_1)$, $z \in [0, D_1)$:
            *   Calculate mip1 index: $idx_1 = x + W_1 \times (y + H_1 \times z)$ [S1, unofficial].
            *   Calculate mip2 target index: $idx_2 = (x / 2) + W_2 \times ((y / 2) + H_2 \times (z / 2))$ [S1, unofficial].
            *   If $idx_2 < V_2$, perform bitwise OR accumulation: `shadow_volume_mip2[idx_2] |= shadow_volume_mip1[idx_1]` [S1, unofficial].
4.  **Upload to GPU:**
    *   Bind the volume texture [S1, unofficial].
    *   Set pixel unpack alignment to 1 byte: `glPixelStorei(GL_UNPACK_ALIGNMENT, 1)` [S1, unofficial].
    *   Upload mip0, mip1, and mip2 using `glTexSubImage3D` with format `GL_RED_INTEGER` and type `GL_UNSIGNED_BYTE` [S1, unofficial].
    *   Unbind the texture [S1, unofficial].
    *   Free the temporary CPU memory for mip1 and mip2 arrays [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_shadow_volume_mipmaps(shadow_volume_mip0, width0, height0, depth0):
    """
    Generates a 3-level mipmap chain for a shadow volume texture using bitwise OR accumulation.
    
    Args:
        shadow_volume_mip0: list of uint8, size width0 * height0 * depth0
        width0: int
        height0: int
        depth0: int
        
    Returns:
        list of lists of uint8: [mip0, mip1, mip2]
    """
    # Mip 1 Generation
    width1 = width0 // 2
    height1 = height0 // 2
    depth1 = depth0 // 2
    volume1 = width1 * height1 * depth1
    
    shadow_volume_mip1 = [0] * volume1
    
    for z in range(depth0):
        for y in range(height0):
            for x in range(width0):
                idx0 = x + width0 * (y + height0 * z)
                idx1 = (x // 2) + width1 * ((y // 2) + height1 * (z // 2))
                
                if idx1 < volume1:
                    shadow_volume_mip1[idx1] |= shadow_volume_mip0[idx0]
                    
    # Mip 2 Generation
    width2 = width1 // 2
    height2 = height1 // 2
    depth2 = depth1 // 2
    volume2 = width2 * height2 * depth2
    
    shadow_volume_mip2 = [0] * volume2
    
    for z in range(depth1):
        for y in range(height1):
            for x in range(width1):
                idx1 = x + width1 * (y + height1 * z)
                idx2 = (x // 2) + width2 * ((y // 2) + height2 * (z // 2))
                
                if idx2 < volume2:
                    shadow_volume_mip2[idx2] |= shadow_volume_mip1[idx1]
                    
    return [shadow_volume_mip0, shadow_volume_mip1, shadow_volume_mip2]
```

**Parameters and Edge Cases:**
*   **Data Type:** The implementation uses `uint8_t` (unsigned byte) for storage [S1, unofficial].
*   **Integer Division:** The mapping uses floor division to determine the target voxel index, grouping input voxels into the output voxel [S1, unofficial].
*   **Memory Alignment:** When uploading to OpenGL, `glPixelStorei(GL_UNPACK_ALIGNMENT, 1)` is required because the data is packed tightly (1 byte per voxel) and may not be aligned to 4-byte boundaries [S1, unofficial].
*   **Boundary Handling:** The condition `if index_mip1 < volume_mip1` handles cases where parent dimensions are odd, preventing out-of-bounds writes [S1, unofficial].

## Subtopics

### [1D Min-Max Mipmap Construction](9410-1d-min-max-mipmap-construction.md)
This method constructs a hierarchical data structure on each row of a rectified shadow map to accelerate finding the lit segments of a camera ray by storing the minimum and maximum values of the shadow heightfield. It solves the problem of quickly determining the intersection of a 1D heightfield with a camera ray, enabling volume rendering algorithms to skip fully shadowed regions.

## Operating characteristics
Not given in the sources.

## Background
Shadow Volume Mipmap Generation using bitwise OR accumulation is described in the context of voxel rendering implementations, specifically in the TTFH/Voxel-Render project [S1, unofficial]. It is a practical technique for optimizing shadow ray traversal in voxel-based rendering engines. The method leverages the hierarchical nature of mipmaps to create a coarse representation of shadow volumes, allowing for efficient early-out tests during volume rendering.

## Sources
[S1] TTFH/Voxel-Render, https://github.com/TTFH/Voxel-Render, fetched 2026-10-27 (unofficial)
