# Tiled 2D Texture Storage for 3D Fields
Also called: 2D Texture Tiling for 3D Fields, Tiled 2D Texture Storage

## Overview
Tiled 2D Texture Storage for 3D Fields is a technique to store a 3D scalar field in a 2D texture by tiling the 3D volume into 2D slices, allowing efficient GPU access via texture lookups [S1, unofficial]. It solves the problem of accessing 3D volume data on GPUs that lack efficient 3D texture support or when memory bandwidth for 3D textures is a bottleneck, by mapping the 3D coordinates to 2D UV space [S1, unofficial]. Unlike [2D Texture-Based Volume Rendering with Z-Slice Packing](4269-2d-texture-based-volume-rendering-with-z-slice-packing.md) which arranges Z-slices in a fixed grid layout within a 2D texture to enable standard bilinear filtering for Z-interpolation, this method specifically tiles the volume into a larger 2D texture where the Z-coordinate determines the tile [S1, unofficial]. It is distinct from [Point Cloud Volumetric Rendering via 2D Atlas](4879-point-cloud-volumetric-rendering-via-2d-atlas.md) as it operates on dense grids rather than point clouds [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A 3D scalar field of dimensions $W \times H \times D$ (width, height, depth).
*   Output: A 2D texture of dimensions $W_{tex} \times H_{tex}$ containing the tiled volume data.

**Data Structures:**
The 2D texture is organized as a grid of tiles. Each tile corresponds to a slice of the 3D volume along the Z-axis.
*   `tilesX`: The number of tiles along the X-axis of the 2D texture. This parameter determines the tiling layout [S1, unofficial].
*   `gridSize`: The spatial resolution of each tile, typically equal to the volume's width $W$ and height $H$ [S1, unofficial].

**Parameters:**
*   `tilesX`: The number of tiles along the X-axis of the 2D texture. It determines the tiling layout [S1, unofficial].
*   `tilesY`: The number of tiles along the Y-axis of the 2D texture. Calculated as $\lceil D / \text{tilesX} \rceil$ [G].
*   `gridSize`: The width and height of each tile, equal to the volume dimensions $W$ and $H$ [S1, unofficial].

**Steps:**
1.  Calculate the number of tiles required along the X and Y axes of the 2D texture. Let `tilesX` be a chosen parameter [S1, unofficial]. Calculate `tilesY` as $\lceil D / \text{tilesX} \rceil$ [G].
2.  Allocate a 2D texture of size $(W \times \text{tilesX}) \times (H \times \text{tilesY})$ [G].
3.  For each voxel at 3D coordinates $(x, y, z)$ where $0 \le x < W$, $0 \le y < H$, $0 \le z < D$:
    a. Calculate the tile coordinates in the 2D texture:
       $$ \text{tileX} = z \pmod{\text{tilesX}} $$
       $$ \text{tileY} = \lfloor z / \text{tilesX} \rfloor $$
       [S1, unofficial]
    b. Calculate the 2D texture coordinates $(u, v)$:
       $$ u = \text{tileX} \times W + x $$
       $$ v = \text{tileY} \times H + y $$
       [S1, unofficial]
    c. Store the scalar value of the voxel at $(u, v)$ in the 2D texture.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def store_volume_in_tiled_2d_texture(volume, tilesX):
    """
    Stores a 3D volume into a 2D texture using tiling.
    
    Args:
        volume: A 3D array of shape (W, H, D).
        tilesX: The number of tiles along the X-axis of the 2D texture.
        
    Returns:
        A 2D array of shape (W * tilesX, H * tilesY) representing the texture.
    """
    W, H, D = volume.shape
    tilesY = (D + tilesX - 1) // tilesX  # Ceiling division [G]
    
    # Initialize 2D texture with zeros
    texture = np.zeros((W * tilesX, H * tilesY), dtype=volume.dtype)
    
    for z in range(D):
        tileX = z % tilesX
        tileY = z // tilesX
        
        for y in range(H):
            for x in range(W):
                u = tileX * W + x
                v = tileY * H + y
                texture[u, v] = volume[x, y, z]
                
    return texture
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Ensure that the 2D texture dimensions are correctly calculated to accommodate all tiles. If $D$ is not a multiple of `tilesX`, the last row of tiles will have empty space [G].
*   **Precision:** When sampling, ensure that the texture coordinates are correctly mapped to avoid sampling from adjacent tiles [G].
*   **Degenerate Input:** If `tilesX` is 0 or greater than $D$, the method is invalid [G].

## Subtopics

### [2D Texture-Based Volume Rendering with Z-Slice Packing](4269-2d-texture-based-volume-rendering-with-z-slice-packing.md)
2D Texture-Based Volume Rendering with Z-Slice Packing is a technique that stores a 3D scalar field as a 2D texture composed of a grid of Z-slices, allowing 3D sampling via 2D texture lookups and trilinear interpolation. It solves the problem of accessing 3D volume data on GPUs that lack efficient 3D texture support or when memory bandwidth for 3D textures is a bottleneck, by mapping the 3D coordinates to 2D UV space. Unlike [Tiled 2D Texture Storage for 3D Fields](2740-tiled-2d-texture-storage-for-3d-fields.md) which tiles the volume into a larger 2D texture, this method specifically arranges Z-slices in a fixed grid layout within a 2D texture to enable standard bilinear filtering for Z-interpolation. It is distinct from [Point Cloud Volumetric Rendering via 2D Atlas](4879-point-cloud-volumetric-rendering-via-2d-atlas.md) as it operates on dense grids rather than point clouds.

### [Point Cloud Volumetric Rendering via 2D Atlas](4879-point-cloud-volumetric-rendering-via-2d-atlas.md)
A real-time GPU-accelerated technique for rendering 3D volumetric data (specifically point clouds or scalar fields) by storing the volume in a 2D texture atlas and rendering it as a point cloud where each point represents a voxel/sample. The depth of each sample is encoded in the atlas layout to allow correct sorting and rendering order without traditional depth buffering for the volume itself. It achieves 60Hz on GTX1080 for a 4k atlas, while Intel 630 graphics render a 2k atlas at ~20fps.

## Operating characteristics
**Time and Memory Complexity:**
*   Time Complexity: $O(W \times H \times D)$ for storing the volume [G].
*   Memory Complexity: $O(W \times H \times D)$ for the 2D texture [G].

**Measured Performance:**
Not given in the sources.

**Scaling:**
Not given in the sources.

**Strengths and Limitations:**
*   Strengths: Allows efficient GPU access to 3D volume data using 2D texture lookups [S1, unofficial].
*   Limitations: Requires careful mapping of 3D coordinates to 2D texture coordinates [G].

**When to Use:**
Use this method when 3D texture support is limited or when memory bandwidth for 3D textures is a bottleneck [S1, unofficial].

## Background
**Origin:**
The technique is described in the GitHub repository [3merillon/WebGL2-GPU-Marching-Cubes](https://github.com/3merillon/WebGL2-GPU-Marching-Cubes) [S1, unofficial].

**History and Development:**
Not given in the sources.

**Where it is Used:**
Used in WebGL2 GPU Marching Cubes implementations [S1, unofficial].

## Sources
[S1] 3merillon/WebGL2-GPU-Marching-Cubes, https://github.com/3merillon/WebGL2-GPU-Marching-Cubes, fetched 2026-10-28 (unofficial)
