# Position Map Voxelization

Also called: Voxelization via Position Map and 3D Image Store

## Overview
Position Map Voxelization is a post-processing step that converts a 2D position map—containing world-space coordinates of previously identified surface points—into a 3D volume texture by marking active voxels [S1, unofficial]. It distinguishes itself from sibling methods by explicitly filling the primary voxel and its 8 immediate neighbors to ensure connectivity and fill gaps, rather than relying solely on the precise rasterization of the surface geometry [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 2D image texture (`positionMap`) storing `vec4` world-space positions, where the alpha channel indicates validity (active surface point) [S1, unofficial].
*   **Output:** A 3D image texture (`volume`) storing 8-bit values (typically `r8` format) indicating active voxels [S1, unofficial].

**Data Structures:**
*   `positionMap`: `layout(binding=1, rgba32f) uniform image2D`. Stores `vec4` where `xyz` is world position and `a` is a validity flag [S1, unofficial].
*   `volume`: `layout(binding=0, r8) uniform image3D`. Stores 8-bit values. Although the shader writes `vec4(1)`, the format truncates this to `1.0` (integer 1) [S1, unofficial].

**Parameters:**
*   `stepSize`: The distance in world units to step to reach neighbor voxels. Used to offset the position before indexing [S1, unofficial].
*   `voxelDim`: The resolution of the volume along each axis (assumed uniform for the formula below) [S1, unofficial].
*   `xBounds`, `yBounds`, `zBounds`: The bounding box of the volume in world space, defined as $[min, max]$ for each axis [S1, unofficial].

**Steps:**
1.  Bind the voxelization shader, the `positionMap` texture (binding point 1), and the `volume` texture (binding point 0) [S1, unofficial].
2.  Bind a full-screen quad VAO and load identity matrices for projection, view, and model matrices [S1, unofficial].
3.  Disable depth testing, depth mask, culling, and color mask to allow direct writes to the volume texture without overwriting concerns [S1, unofficial].
4.  Render the full-screen quad. For each fragment:
    a.  Read the world-space position from the position map: `worldPos = imageLoad(positionMap, ivec2(gl_FragCoord.xy))` [S1, unofficial].
    b.  Check if the voxel is active by testing the alpha channel: `if (worldPos.a > 0)` [S1, unofficial].
    c.  If active, convert `worldPos.xyz` to voxel indices $(x, y, z)$ using the formula below and store a white color `vec4(1)` in the volume texture at that index [S1, unofficial].
    d.  For each of the 8 neighbors defined by the sign combinations of $(\pm 1, \pm 1, \pm 1)$:
        i.  Calculate the neighbor's world position: $pos_{neighbor} = worldPos.xyz + stepSize \times \text{normalize}(vec3(dx, dy, dz))$, where $(dx, dy, dz)$ is the current sign combination [S1, unofficial].
        ii. Convert $pos_{neighbor}$ to voxel indices and store `vec4(1)` in the volume texture at that index [S1, unofficial].
5.  Generate mipmaps for the 3D volume texture [S1, unofficial].
6.  Unbind textures and shader, and reset OpenGL state (enable depth test, culling, depth mask, color mask) [S1, unofficial].

**Formula:**
The conversion from world position $pos = (pos.x, pos.y, pos.z)$ to voxel indices $(x, y, z)$ for a volume of dimension `voxelDim` and bounds $[xBounds, yBounds, zBounds]$ is:

$$
x = \text{voxelDim} \times \frac{pos.x - xBounds.x}{xBounds.y - xBounds.x}
$$

$$
y = \text{voxelDim} \times \frac{pos.y - yBounds.x}{yBounds.y - yBounds.x}
$$

$$
z = \text{voxelDim} \times \frac{pos.z - zBounds.x}{zBounds.y - zBounds.x}
$$

The function returns `ivec3(x, y, z)`. Standard integer truncation or floor is implied by the return type `ivec3` in typical GLSL implementations [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader Logic for Position Map Voxelization

# Uniforms/Inputs
# positionMap: image2D (binding 1, rgba32f)
# volume: image3D (binding 0, r8)
# voxelDim: int
# xBounds: vec2 (min, max)
# yBounds: vec2 (min, max)
# zBounds: vec2 (min, max)
# stepSize: float

def calculateVoxelIndex(pos: vec3) -> ivec3:
    x = voxelDim * (pos.x - xBounds.x) / (xBounds.y - xBounds.x)
    y = voxelDim * (pos.y - yBounds.x) / (yBounds.y - yBounds.x)
    z = voxelDim * (pos.z - zBounds.x) / (zBounds.y - zBounds.x)
    return ivec3(int(x), int(y), int(z)) # Truncation implied

def main():
    # 1. Read world position from position map
    worldPos = imageLoad(positionMap, ivec2(gl_FragCoord.xy))
    
    # 2. Check validity
    if worldPos.a > 0:
        col = vec4(1.0) # White color, truncated to 1.0 in r8 format
        
        # 3. Mark primary voxel
        primaryIndex = calculateVoxelIndex(worldPos.xyz)
        imageStore(volume, primaryIndex, col)
        
        # 4. Mark 8 neighbors
        # Sign combinations for dx, dy, dz: (-1,-1,-1), (-1,-1,1), ..., (1,1,1)
        signs = [
            (-1, -1, -1), (-1, -1, 1), (-1, 1, -1), (-1, 1, 1),
            (1, -1, -1), (1, -1, 1), (1, 1, -1), (1, 1, 1)
        ]
        
        for dx, dy, dz in signs:
            neighborPos = worldPos.xyz + stepSize * normalize(vec3(dx, dy, dz))
            neighborIndex = calculateVoxelIndex(neighborPos)
            imageStore(volume, neighborIndex, col)
```

**Edge Cases and Pitfalls:**
*   **Precision:** The position map uses `rgba32f` to preserve world-space accuracy [S1, unofficial]. Insufficient precision in the position map will lead to incorrect voxel indexing.
*   **Boundary Conditions:** The formula assumes the volume bounds are correctly set. If a world position falls outside the bounds, the resulting voxel index may be negative or exceed `voxelDim`, leading to out-of-bounds memory access in `imageStore`. The sources do not specify explicit clamping logic [S1, unofficial].
*   **Neighbor Offset:** The use of `stepSize * normalize(...)` for neighbors assumes a uniform voxel grid. If the grid is non-uniform, this approach will incorrectly offset the neighbor positions [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** $O(N)$, where $N$ is the number of pixels in the position map (screen resolution). Each active pixel triggers 9 writes to the volume texture (1 primary + 8 neighbors) [S1, unofficial].
*   **Memory:** The position map requires $W \times H \times 16$ bytes (for `rgba32f`). The volume texture requires $X \times Y \times Z \times 1$ byte (for `r8`) [S1, unofficial].

**Measured Performance:**
*   Not given in the sources.

**Scaling:**
*   Scales linearly with screen resolution for the position map read. The volume texture writes are bounded by the volume resolution.

**Strengths and Limitations:**
*   **Strengths:** Ensures connectivity by filling neighbors, reducing gaps in the voxelized surface [S1, unofficial]. Leverages GPU parallelism for fast voxelization [S1, unofficial].
*   **Limitations:** The fixed `stepSize` for neighbors may not adapt to varying surface curvature or density. Out-of-bounds indices are not handled in the provided sources [S1, unofficial].

**When to use it and when not:**
*   Use when connectivity and gap-filling are critical for subsequent volume rendering steps (e.g., ray marching) [S1, unofficial].
*   Do not use if precise surface representation without neighbor filling is required, or if the volume bounds are not well-defined [S1, unofficial].

## Background
**Origin:**
The method is described in the context of a cloud renderer implementation, specifically as a "second voxelizer" pass that updates the position map and generates mipmaps for the volume texture [S1, unofficial].

**History and Development:**
The technique is part of a broader family of rasterization-based voxelization methods that use image-space transfer mechanisms [S1, unofficial].

**Where it is used:**
Used in real-time cloud rendering applications where efficient voxelization of surface geometry is required for subsequent volume rendering techniques like cone tracing [S1, unofficial].

## Sources
[S1] jaafersheriff/Cloud-Renderer, https://github.com/jaafersheriff/Cloud-Renderer, fetched 2026-10-27 (unofficial)
