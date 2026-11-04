# World to UVW Normalization

Also called: Coordinate Normalization, DCX Mesh Normalization, Voxel Coordinate Normalization, World-to-Grid Normalization

## Overview
World to UVW Normalization maps arbitrary 3D world coordinates into the normalized $[0, 1]$ range of a voxel grid, enabling consistent indexing and texture sampling regardless of the scene's physical scale or origin [S3, unofficial]. It differs from [Coordinate Space Conversion](1785-coordinate-space-conversion.md) by explicitly handling discrete voxel grid boundaries and dimensionality, and from [Texture Coordinate Conversion](3581-texture-coordinate-conversion.md) by operating at the grid indexing level rather than applying shader-specific offsets like Y-flips or centering adjustments [G].

## How it works
**Inputs:**
- World position $P = (x, y, z)$ [S3, unofficial].
- Volume metadata: origin $O = (x_{min}, y_{min}, z_{min})$, dimensions $D = (d_x, d_y, d_z)$, and voxel sizes $V = (v_x, v_y, v_z)$ [S3, unofficial].

**Outputs:**
- Normalized coordinates $U = (u, v, w) \in [0, 1]^3$ [S3, unofficial].

**Data Structures:**
- `Volume`: Contains fields `origin` (float3), `dim` (int3), and `voxel_size` (float3) [S3, unofficial].
- `Mesh` (for mesh-based variants): Contains `bounding_box` with `centroid` and `extents` [S2, unofficial].

**Steps:**
1. **Compute Inverse Voxel Sizes:** Calculate the inverse of each voxel dimension to convert world units to voxel units. If any voxel size is zero, return a zero vector or handle as an error [S3, unofficial].
   $$ inv\_voxel = \left( \frac{1}{v_x}, \frac{1}{v_y}, \frac{1}{v_z} \right) $$
2. **Compute Relative Indices:** Subtract the volume origin from the world position and multiply by the inverse voxel sizes to get the index in voxel space [S3, unofficial].
   $$ idx = (P - O) \odot inv\_voxel $$
   where $\odot$ is element-wise multiplication [G].
3. **Compute Dimension Minus One:** Convert the grid dimensions to floats and subtract one [S3, unofficial].
   $$ dimm1 = (d_x - 1, d_y - 1, d_z - 1) $$
4. **Normalize and Clamp:** Divide the relative indices by the dimension minus one and clamp the result to $[0, 1]$ to handle out-of-bounds queries [S3, unofficial].
   $$ u = \text{clamp}\left( \frac{idx_x}{dimm1_x}, 0.0, 1.0 \right) $$
   $$ v = \text{clamp}\left( \frac{idx_y}{dimm1_y}, 0.0, 1.0 \right) $$
   $$ w = \text{clamp}\left( \frac{idx_z}{dimm1_z}, 0.0, 1.0 \right) $$

**Alternative Mesh-Based Normalization (DCX):**
For mesh data not inherently tied to a pre-defined voxel grid, normalization involves [S2, unofficial]:
1. Concatenate all geometries into a single mesh [S2, unofficial].
2. Compute the centroid of the bounding box: $C = \text{centroid}$ [S2, unofficial].
3. Translate the mesh by $-C$ [S2, unofficial].
4. Compute the scale factor: $S = \frac{2.0 \times 0.45}{\max(\text{extents})}$, where $0.45$ is a margin multiplier [S2, unofficial].
5. Scale the mesh by $S$ [S2, unofficial].
6. Apply the Smokeview coordinate transformation formulas for interactive viewing [S1, unofficial]:
   $$ \hat{x} = \frac{x - x_{min}}{x_{max} - x_{min}} $$
   $$ \hat{y} = \frac{y - y_{min}}{y_{max} - y_{min}} $$
   $$ \hat{z} = \frac{z - z_{min}}{z_{max} - z_{min}} $$
   where $x_{min}, x_{max}$ etc. are the global bounds of all meshes [S1, unofficial]. (Note: The source text contains a typo `xy_max` in the denominator, corrected here to `x_max` for implementation correctness [G].)

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vector3:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

def world_to_uvw(p_world: Vector3, vol: Volume) -> Vector3:
    # Step 1: Compute inverse voxel sizes
    # Handle zero voxel size to avoid division by zero [S3, unofficial]
    inv_vx = 1.0 / vol.voxel_size.x if vol.voxel_size.x != 0 else 0.0
    inv_vy = 1.0 / vol.voxel_size.y if vol.voxel_size.y != 0 else 0.0
    inv_vz = 1.0 / vol.voxel_size.z if vol.voxel_size.z != 0 else 0.0
    
    # Step 2: Compute relative indices
    idx_x = (p_world.x - vol.origin.x) * inv_vx
    idx_y = (p_world.y - vol.origin.y) * inv_vy
    idx_z = (p_world.z - vol.origin.z) * inv_vz
    
    # Step 3: Compute dimension minus one
    dimm1_x = float(vol.dim.x - 1)
    dimm1_y = float(vol.dim.y - 1)
    dimm1_z = float(vol.dim.z - 1)
    
    # Handle degenerate dimensions (size 1) to prevent division by zero [G]
    if dimm1_x == 0: dimm1_x = 1.0
    if dimm1_y == 0: dimm1_y = 1.0
    if dimm1_z == 0: dimm1_z = 1.0
    
    # Step 4: Normalize and clamp
    u = max(0.0, min(1.0, idx_x / dimm1_x))
    v = max(0.0, min(1.0, idx_y / dimm1_y))
    w = max(0.0, min(1.0, idx_z / dimm1_z))
    
    return Vector3(u, v, w)
```

**Parameters:**
- `vol.origin`: The world coordinate of the first voxel (index 0,0,0) [S3, unofficial].
- `vol.dim`: The number of voxels in each dimension [S3, unofficial].
- `vol.voxel_size`: The physical size of each voxel [S3, unofficial].
- `0.45`: Margin factor for mesh normalization to ensure the object fits within the normalized cube with some padding [S2, unofficial].

**Edge Cases:**
- **Zero Voxel Size:** Triggers a debug print and returns a zero vector [S3, unofficial].
- **Dimension 1:** If a dimension is 1, `dimm1` becomes 0, leading to division by zero. The pseudocode handles this by defaulting to 1.0 [G].
- **Out of Bounds:** The clamping step ensures that coordinates outside the grid map to the nearest boundary (0 or 1) [S3, unofficial].

## Subtopics

### [Texture Coordinate Conversion](3581-texture-coordinate-conversion.md)
Transforms normalized coordinates into texture-specific coordinates. It differs from the parent method by applying shader-space adjustments like a 0.5 offset and a Y-axis flip. Choose it when sampling volume textures in shaders.

### [Coordinate Space Conversion](1785-coordinate-space-conversion.md)
General conversion between coordinate systems without specific voxel grid constraints. It differs by handling continuous spaces rather than discrete grid boundaries. Choose it for general scene transformations [G].

### [Voxel Grid Initialization](2168-voxel-grid-initialization.md)
Sets up the grid structure and metadata required for normalization. It differs by allocating memory and setting origin/dimensions before indexing. Choose it at scene load time [G].

### [Bitstream Voxel Position Encoding](2371-bitstream-voxel-position-encoding.md)
Encodes voxel positions for compact storage. It differs by packing indices into bitstreams rather than returning floats. Choose it for memory-constrained transmission [G].

### [3D Texture Representation (CPU/GPU Indexing)](2393-3d-texture-representation-cpu-gpu-indexing.md)
Maps normalized coordinates to 3D texture memory. It differs by handling hardware texture fetch instructions. Choose it for GPU-accelerated rendering [G].

### [Uniform Grid Adjacent Cell Indexing](2992-uniform-grid-adjacent-cell-indexing.md)
Finds neighboring cells based on normalized coordinates. It differs by computing offsets for neighbors rather than absolute positions. Choose it for gradient or stencil operations [G].

### [Multi-Material Cell Map Indexing & Access](4886-multi-material-cell-map-indexing-access.md)
Handles multi-material grids using normalized indices. It differs by resolving material IDs at grid boundaries. Choose it for heterogeneous volume data [G].

### [Spatial Hashing](5002-spatial-hashing.md)
Alternative indexing method for sparse grids. It differs by mapping coordinates to hash buckets instead of dense arrays. Choose it for highly sparse or dynamic scenes [G].

## Operating characteristics
- **Time Complexity:** $O(1)$ per coordinate query [G].
- **Memory Complexity:** $O(1)$ auxiliary memory [G].
- **Strengths:** Fast, deterministic mapping; handles arbitrary world scales; robust to out-of-bounds queries via clamping [G].
- **Limitations:** Assumes uniform voxel sizes within a single volume [S3, unofficial]; does not handle non-uniform grids without modification [G].
- **When to Use:** When sampling dense voxel grids or transferring world coordinates to texture samplers [G].
- **When Not to Use:** For non-uniform grids or sparse data where spatial hashing is more appropriate [G].

## Background
The method originates from the need to visualize Fire Dynamics Simulation (FDS) data in Smokeview, where world coordinates must be mapped to a normalized grid for interactive viewing [S1, unofficial]. It is also used in mesh normalization pipelines to standardize scene scales [S2, unofficial], and in CUDA-accelerated volume renderers for efficient voxel indexing [S3, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S2] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
[S3] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
