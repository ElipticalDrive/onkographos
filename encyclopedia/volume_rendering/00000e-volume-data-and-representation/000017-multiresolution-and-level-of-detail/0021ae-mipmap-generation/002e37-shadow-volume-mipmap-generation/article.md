# Shadow Volume Mipmap Generation
Also called: Bitwise OR Mipmap Accumulation

## Overview
Shadow Volume Mipmap Generation constructs a hierarchical mipmap chain from a 3D shadow volume texture using bitwise OR accumulation to preserve occlusion presence across downsampled resolutions [S1, unofficial]. It enables volume rendering algorithms to quickly identify fully shadowed regions by allowing rays to skip large portions of the domain where no geometry exists, solving the performance bottleneck of naive ray marching through empty space [S1, unofficial]. It sets itself apart from standard 3D mipmap generation by using logical OR instead of averaging or min/max filtering, ensuring that any occlusion in a parent voxel propagates to all child mip levels [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D shadow volume texture (mip0) of dimensions $W_0 \times H_0 \times D_0$, containing binary occlusion data (1 for shadowed, 0 for lit).
*   **Output:** A 3D texture object with a mipmap chain (mip0, mip1, mip2, ...) where each level is half the resolution of the previous level in each dimension.

**Data Structures:**
*   **Shadow Volume Mip Level:** A 1D array of `uint8_t` values representing the 3D volume. The array is indexed linearly.
*   **Indexing Formula:** For a volume of dimensions $W \times H \times D$, the linear index for coordinates $(x, y, z)$ is:
    $$ \text{index} = x + W \times (y + H \times z) $$
    where $0 \le x < W$, $0 \le y < H$, and $0 \le z < D$.

**Algorithm Steps:**
1.  **Initialize Mip0:** Ensure the base level (mip0) is populated with the raw shadow volume data.
2.  **Generate Mip1:**
    *   Calculate dimensions: $W_1 = \lfloor W_0 / 2 \rfloor$, $H_1 = \lfloor H_0 / 2 \rfloor$, $D_1 = \lfloor D_0 / 2 \rfloor$.
    *   Calculate volume size: $V_1 = W_1 \times H_1 \times D_1$.
    *   Allocate memory for `shadow_volume_mip1` of size $V_1$ and initialize all bytes to 0.
    *   Iterate through all voxels in mip0:
        *   For each $x \in [0, W_0)$, $y \in [0, H_0)$, $z \in [0, D_0)$:
            *   Calculate mip0 index: $idx_0 = x + W_0 \times (y + H_0 \times z)$.
            *   Calculate mip1 target coordinates: $x' = \lfloor x / 2 \rfloor$, $y' = \lfloor y / 2 \rfloor$, $z' = \lfloor z / 2 \rfloor$.
            *   Calculate mip1 index: $idx_1 = x' + W_1 \times (y' + H_1 \times z')$.
            *   Perform bitwise OR accumulation: `shadow_volume_mip1[idx_1] |= shadow_volume_mip0[idx_0]`.
3.  **Generate Mip2:**
    *   Calculate dimensions: $W_2 = \lfloor W_1 / 2 \rfloor$, $H_2 = \lfloor H_1 / 2 \rfloor$, $D_2 = \lfloor D_1 / 2 \rfloor$.
    *   Calculate volume size: $V_2 = W_2 \times H_2 \times D_2$.
    *   Allocate memory for `shadow_volume_mip2` of size $V_2$ and initialize all bytes to 0.
    *   Iterate through all voxels in mip1:
        *   For each $x \in [0, W_1)$, $y \in [0, H_1)$, $z \in [0, D_1)$:
            *   Calculate mip1 index: $idx_1 = x + W_1 \times (y + H_1 \times z)$.
            *   Calculate mip2 target coordinates: $x' = \lfloor x / 2 \rfloor$, $y' = \lfloor y / 2 \rfloor$, $z' = \lfloor z / 2 \rfloor$.
            *   Calculate mip2 index: $idx_2 = x' + W_2 \times (y' + H_2 \times z')$.
            *   Perform bitwise OR accumulation: `shadow_volume_mip2[idx_2] |= shadow_volume_mip1[idx_1]`.
4.  **Upload to GPU:**
    *   Bind the 3D texture object.
    *   Set pixel unpack alignment to 1 byte: `glPixelStorei(GL_UNPACK_ALIGNMENT, 1)`.
    *   Upload mip0 using `glTexSubImage3D` with format `GL_RED_INTEGER` and type `GL_UNSIGNED_BYTE`.
    *   Upload mip1 using `glTexSubImage3D` with the same format and type.
    *   Upload mip2 using `glTexSubImage3D` with the same format and type.
    *   Unbind the texture.
    *   Free the temporary CPU memory for mip1 and mip2 arrays.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_shadow_volume_mipmaps(shadow_volume_mip0, width0, height0, depth0):
    """
    Generates a 3-level mipmap chain for a shadow volume texture using bitwise OR accumulation.
    
    Args:
        shadow_volume_mip0: list of uint8_t, size width0 * height0 * depth0
        width0: int
        height0: int
        depth0: int
        
    Returns:
        list of lists of uint8_t: [mip0, mip1, mip2]
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
                x1 = x // 2
                y1 = y // 2
                z1 = z // 2
                idx1 = x1 + width1 * (y1 + height1 * z1)
                
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
                x2 = x // 2
                y2 = y // 2
                z2 = z // 2
                idx2 = x2 + width2 * (y2 + height2 * z2)
                
                if idx2 < volume2:
                    shadow_volume_mip2[idx2] |= shadow_volume_mip1[idx1]
                    
    return [shadow_volume_mip0, shadow_volume_mip1, shadow_volume_mip2]
```

**Parameters and Edge Cases:**
*   **Bitwise OR:** The core operation is logical OR. If any of the 2x2x2 input voxels mapping to a single output voxel is 1 (shadowed), the output voxel is 1. This ensures conservative shadowing; no false positives (lit where shadowed) are introduced, but false negatives (shadowed where lit) are possible if the downsampled region contains mixed occlusion [S1, unofficial].
*   **Integer Division:** The mapping uses floor division (`//` in Python, `/` in C++ for integers) to determine the target voxel index. This effectively groups voxels $(2i, 2j, 2k), (2i+1, 2j, 2k), (2i, 2j+1, 2k), \dots$ into the output voxel $(i, j, k)$.
*   **Memory Alignment:** When uploading to OpenGL, `glPixelStorei(GL_UNPACK_ALIGNMENT, 1)` is required because the data is packed tightly (1 byte per voxel) and may not be aligned to 4-byte boundaries [S1, unofficial].
*   **Data Type:** The source uses `uint8_t` (unsigned byte) for storage, which is sufficient for binary shadow data [S1, unofficial].

## Subtopics

### [1D Min-Max Mipmap Construction](9410-1d-min-max-mipmap-construction.md)
This method constructs a hierarchical data structure from each row of a rectified shadow map to accelerate the identification of lit segments along a camera ray. It solves the problem of efficiently determining which parts of a rectified shadow map are occluded by geometry, thereby allowing volume rendering algorithms to skip fully shadowed regions. It sets itself apart from naive ray marching by using hierarchical min-max bounds to skip large portions of the domain, and from standard binary tree traversal by utilizing a flat array representation with specific iterative advancement rules to avoid recursion overhead in shaders.

## Operating characteristics
*   **Time Complexity:** Generating each mip level takes $O(N)$ time, where $N$ is the number of voxels in the parent level. For a 3-level chain, the total time is proportional to $N_0 + N_1 + N_2 \approx N_0 (1 + 1/8 + 1/64)$, which is dominated by the base level.
*   **Memory Complexity:** The total memory required for the mipmap chain is $N_0 + N_1 + N_2 \approx 1.14 N_0$ voxels. This is a small overhead compared to the base volume.
*   **Performance:** The method is extremely fast on CPU due to the simplicity of the bitwise OR operation and linear memory access. On GPU, this can be implemented as a series of 3D fragment shaders or compute shaders, leveraging texture filtering hardware if configured correctly, though explicit accumulation offers more control [S1, unofficial].
*   **Strengths:** Simple to implement, low memory overhead, preserves shadow presence accurately (conservative), fast generation.
*   **Limitations:** Does not provide information about the *amount* of shadowing, only presence. May overestimate shadowing in regions with sparse geometry. Not suitable for soft shadows or penumbra calculation.
*   **When to Use:** When rendering large volumes with sparse geometry and needing to quickly cull fully shadowed regions.
*   **When Not to Use:** When soft shadows, penumbra, or precise shadow intensity are required.

## Background
Shadow Volume Mipmap Generation using bitwise OR accumulation is described in the context of voxel rendering implementations, specifically in the TTFH/Voxel-Render project [S1, unofficial]. It is a practical technique for optimizing shadow ray traversal in voxel-based rendering engines. The method leverages the hierarchical nature of mipmaps to create a coarse representation of shadow volumes, allowing for efficient early-out tests during volume rendering.

## Sources
[S1] TTFH/Voxel-Render, https://github.com/TTFH/Voxel-Render, fetched 2026-10-27 (unofficial)
