# 3D Texture Storage

Also called: 3D Texture Voxel Storage, 3D texture voxel lookup, Texture-based Voxel Storage

## Overview
3D Texture Storage maps a volumetric voxel grid into a hardware-accelerated 3D texture resource, leveraging native 3D coordinate indexing and texture caching for efficient sampling during ray traversal [S1, unofficial][S2, unofficial]. Unlike linear byte buffers which require manual address calculation, this method allows the GPU to handle spatial locality automatically [S1, unofficial]. It is distinct from [Voxel Texture Encoding](8970-voxel-texture-encoding.md), which embeds 3D data into 2D textures to enable specific pipeline techniques like ping-pong recursion [G].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid of dimensions $W \times H \times D$ (width, height, depth) containing voxel data (e.g., occupancy, color, density) [S1, unofficial][S2, unofficial].
*   **Output:** A 3D texture resource (e.g., `Texture3D<uint>`) bound to a shader register, allowing direct lookup of voxel values at 3D coordinates [S1, unofficial].

**Data Structures:**
The voxel grid is stored in a 3D texture resource. In HLSL/DirectX 12 syntax, this is declared as:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```hlsl
Texture3D<uint> voxels : register(t0, space2);
```
The layout follows standard row-major ordering where spatially adjacent voxels are adjacent in memory [S1, unofficial]. For a voxel at integer coordinates $(x, y, z)$, the linear memory index $v$ is:
$$ v = x + y \cdot W + z \cdot W^2 $$
This layout ensures that the 8 voxels forming a tri-linear interpolation cell are contiguous in memory, minimizing cache misses [G].

**Steps:**
1.  **Calculate Ray Position:** Determine the 3D world or texture coordinate $(x, y, z)$ where the ray intersects the volume [S2, unofficial].
2.  **Apply Offset:** Add a small epsilon offset to the ray position along the ray direction [S2, unofficial].
3.  **Sample Texture:** Use the hardware texture sampling function to retrieve the voxel value at the adjusted coordinate [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def load_voxel_3d_texture(voxel_texture, p_x, p_y, p_z):
    """
    Loads a voxel value from a 3D texture resource using integer coordinates.
    Matches the HLSL pattern: voxels.Load(int4(p, 0)).r
    """
    # Construct 3D coordinate vector for hardware lookup
    coord = (p_x, p_y, p_z)
    # Hardware texture load returns a uint4; extract the first component
    voxel_value = voxel_texture.Load(coord)[0]
    return voxel_value

def sample_voxel_with_offset(voxel_texture, ray_origin, ray_direction, epsilon=1e-4):
    """
    Samples a voxel with an offset to avoid interpolation noise.
    """
    # Apply offset to push sampling point slightly forward into the voxel
    adjusted_origin = ray_origin + ray_direction * epsilon
    # Hardware texture sample performs tri-linear interpolation
    value = voxel_texture.Sample(adjusted_origin)
    return value
```

**Parameters:**
*   **Epsilon Offset:** A small value (e.g., $10^{-4}$) added to the ray position to resolve boundary ambiguity [S2, unofficial].
*   **Grid Extent:** The dimensions of the voxel grid, used for linear indexing if manual calculation is required [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Interpolation Noise:** GPU texture sampling interpolates coordinates between pixels, causing random rounding that produces noise when determining binary voxel occupancy (filled vs. empty) [S2, unofficial]. This is mitigated by adding a small offset to the ray position [S2, unofficial].
*   **Cache Locality:** While 3D textures improve cache locality for spatially adjacent accesses, ray tracing may still access voxels far apart in memory if the traversal order is not optimized [S1, unofficial].

## Subtopics

### [Voxel Texture Encoding](8970-voxel-texture-encoding.md)
Stores 3D data in 2D textures by embedding spatial coordinates in channels, enabling pipeline techniques like ping-pong recursion. Chose this when 2D texture units are preferred or when specific GPU pipeline optimizations are required. [G]

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel lookup, as texture sampling is a hardware-accelerated constant-time operation [S1, unofficial][S2, unofficial].
*   **Memory Complexity:** $O(W \cdot H \cdot D)$, where $W, H, D$ are the grid dimensions [G].
*   **Performance:** Compared to `ByteAddressBuffer`, 3D texture storage reduces GPU walltime by 14.2% and increases FPS by 10.3% on a MacBook Air M2, due to better cache utilization for spatially adjacent data access [S1, unofficial].
*   **Strengths:** Hardware-accelerated 3D coordinate indexing, improved cache locality, simpler shader code (no manual byte shifting) [S1, unofficial][S2, unofficial].
*   **Limitations:** Requires hardware support for 3D textures, potential interpolation noise if not handled with offsets [S2, unofficial].
*   **When to Use:** When ray traversal accesses spatially adjacent voxels frequently, and hardware texture caching can be leveraged [S1, unofficial].
*   **When Not to Use:** When manual control over memory layout is required (e.g., custom compression, Morton codes) [G].

## Background
3D texture storage for voxel data emerged as a natural extension of texture mapping techniques into volumetric rendering [G]. Early implementations used linear byte buffers, requiring manual address calculation and suffering from poor cache locality [S1, unofficial]. The shift to 3D textures was driven by the need to leverage hardware-accelerated 3D indexing and caching [S2, unofficial].

## Sources
[S1] Voxel Raytracer Devlog // My thought repository, https://nelari.us/post/voxel-ray-tracing/, fetched 2026-10-24 (unofficial)
[S2] VOXEL Rendering And Traversal Algorithms | Devlog #7, https://www.youtube.com/watch?v=NzVOPyWvBcw, fetched 2026-10-25 (unofficial)
