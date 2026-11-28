# UV-mapped Instanced Point Cloud/Voxel Rendering

Also called: Blender Instanced Point Cloud Rendering, UV Atlas Instancing for Voxels

## Overview
This method renders large-scale point clouds or voxel grids by mapping each occupied voxel or point to a unique location in a 2D color texture atlas via UV coordinates, then using face-based object instancing to duplicate a single particle mesh (pseudo-face) at each position [S1, unofficial]. It differs from standard instancing by decoupling per-instance color data from the geometry buffer, storing it in a texture atlas to optimize memory bandwidth and allow for high instance counts on hardware with limited per-instance attribute support [S1, unofficial].

## How it works
**Inputs:**
- Array of $N$ 3D positions $P = \{p_0, \dots, p_{N-1}\}$ where $p_i \in \mathbb{R}^3$.
- Array of $N$ colors $C = \{c_0, \dots, c_{N-1}\}$ where $c_i \in [0, 1]^4$ (RGBA).
- Texture width exponent $E$ (default 13, i.e., $W=8192$) [S1, unofficial].
- Particle geometry type (point cloud or voxel cube) [S1, unofficial].

**Outputs:**
- A single instancer mesh object containing $N$ triangular faces.
- A 2D color texture atlas image.
- An instanced particle mesh parented to the instancer.

**Data Structures:**
1. **Instancer Mesh Vertices:** Array of shape $(3N, 3)$. For each point $i$, the 3D position $p_i$ is repeated three times consecutively.
   $$ V_{instancer} = [p_0, p_0, p_0, p_1, p_1, p_1, \dots, p_{N-1}, p_{N-1}, p_{N-1}] $$
2. **Instancer Mesh Loops:** Array of $3N$ integers, sequential from $0$ to $3N-1$.
3. **Instancer Mesh Polygons:** List of $N$ polygons. Each polygon has 3 loops. The loop start indices are $[0, 3, 6, \dots, 3(N-1)]$.
4. **UV Coordinates:** Array of shape $(3N, 2)$. For each point $i$, the UV coordinates are computed for all 3 vertices of the corresponding face.
   $$ uv\_idx = i $$
   $$ uv\_x = uv\_idx \ \& \ (2^E - 1) $$
   $$ uv\_y = uv\_idx \gg E $$
   The UV array contains $(uv\_x, uv\_y)$ repeated 3 times per point.
5. **Color Texture Atlas:** 2D image with width $W = 2^E$ and height $H = \lceil N / W \rceil$. Pixel data is a flat RGBA array of size $H \times W \times 4$.

**Steps:**
1. **Create Instancer Vertices:** Generate an array of $3N$ vertices. For each $i \in [0, N-1]$, set vertices $3i, 3i+1, 3i+2$ to $p_i$.
2. **Create Instancer Loops:** Generate $3N$ loop indices $0 \dots 3N-1$.
3. **Create Instancer Polygons:** Create $N$ triangles. Triangle $i$ uses loop indices $3i, 3i+1, 3i+2$.
4. **Generate UVs:** Compute linear index $i$ for each point. Calculate $uv\_x = i \pmod{2^E}$ and $uv\_y = \lfloor i / 2^E \rfloor$. Assign $(uv\_x, uv\_y)$ to the 3 vertices of triangle $i$.
5. **Create Color Texture Atlas:**
   - Calculate rows $R = \lceil N / 2^E \rceil$ and columns $C = \min(N, 2^E)$.
   - Resize/flatten the color array $C$ to shape $(R \times C, 4)$. If $N < R \times C$, pad with zeros or transparent black.
   - Create a 2D image of size $(C, R)$ and set pixel data from the flattened array.
   - **Pitfall:** Pack the image immediately after setting pixels to prevent data loss upon file save/load [S1, unofficial].
6. **Setup Instancing:**
   - Create the particle mesh (e.g., a small quad for points or a cube for voxels) [S1, unofficial].
   - Parent the particle mesh to the instancer object.
   - Set instancer type to FACES.
   - Hide the instancer object from rendering.
   - Assign the color texture atlas to the particle mesh's material color channel.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def create_uv_mapped_instancer(positions, colors, exp=13):
    """
    Create instancer mesh and texture atlas for UV-mapped instancing.
    
    Args:
        positions: (N, 3) array of 3D coordinates.
        colors: (N, 4) array of RGBA colors in [0, 1].
        exp: Exponent for texture width (default 13 for 8192).
        
    Returns:
        instancer_vertices: (3N, 3) array
        instancer_loops: (3N,) array
        instancer_polygons: list of (3,) loop index tuples
        uv_coords: (3N, 2) array
        texture_image: 2D numpy array (H, W, 4)
    """
    N = len(positions)
    W = 1 << exp  # 2^exp
    
    # 1. Instancer Vertices: Repeat each position 3 times
    instancer_vertices = np.repeat(positions, 3, axis=0)
    
    # 2. Instancer Loops: Sequential indices
    instancer_loops = np.arange(3 * N)
    
    # 3. Instancer Polygons: Triangles
    instancer_polygons = []
    for i in range(N):
        start = 3 * i
        instancer_polygons.append((start, start + 1, start + 2))
        
    # 4. UV Coordinates
    # Linear index for each point
    linear_indices = np.arange(N)
    uv_x = linear_indices & (W - 1)
    uv_y = linear_indices >> exp
    
    # Stack UVs and repeat for 3 vertices per point
    uv_points = np.column_stack((uv_x, uv_y))
    uv_coords = np.repeat(uv_points, 3, axis=0)
    
    # 5. Color Texture Atlas
    H = int(np.ceil(N / W))
    # Flatten colors to (H*W, 4), padding with zeros if necessary
    total_pixels = H * W
    padded_colors = np.zeros((total_pixels, 4), dtype=np.float32)
    padded_colors[:N] = colors
    
    # Reshape to (H, W, 4)
    texture_image = padded_colors.reshape(H, W, 4)
    
    return instancer_vertices, instancer_loops, instancer_polygons, uv_coords, texture_image
```

**Parameters:**
- `MAX_TEXTURE_WIDTH_EXP`: 13 (Texture width 8192). Chosen for AMD RDNA3 compatibility [S1, unofficial].
- `particle_radius`: 0.02 (Default for point clouds) [S1, unofficial].
- `edge_length`: 0.16 (Default for voxel cubes) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Texture Width Limit:** Exceeding 8192 pixels in width causes rendering failures on AMD RDNA3 GPUs [S1, unofficial].
- **Memory Persistence:** Pixel data is lost if the texture image is not packed into the file after creation [S1, unofficial].
- **Degenerate Input:** If $N=0$, no geometry is generated.

## Operating characteristics
- **Time Complexity:** $O(N)$ for mesh and texture generation.
- **Memory Complexity:** $O(N)$ for vertex/UV data and $O(2^E \cdot \lceil N/2^E \rceil)$ for the texture atlas.
- **Performance:** Efficient for very large $N$ due to reduced per-instance attribute overhead and optimized texture sampling.
- **Limitations:** Limited by texture size (8192x8192 max practical width). Not suitable for dynamic per-vertex color changes without texture updates.

## Background
Originated in the Blender community for rendering large-scale point clouds and voxel data (e.g., KITTI dataset) using Blender's Cycles engine [S1, unofficial]. Developed by Christoph Rist to handle large datasets efficiently on consumer hardware [S1, unofficial].

## Sources
[S1] risteon/blender_kitti, https://github.com/risteon/blender_kitti, fetched 2026-10-27 (unofficial)
