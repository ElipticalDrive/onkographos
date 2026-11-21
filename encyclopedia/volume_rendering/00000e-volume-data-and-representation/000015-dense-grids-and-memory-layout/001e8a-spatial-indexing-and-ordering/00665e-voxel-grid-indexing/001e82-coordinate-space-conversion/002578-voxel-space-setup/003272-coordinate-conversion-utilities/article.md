# Coordinate Conversion Utilities

Also called: Voxel to World Transform, World-to-Voxel Coordinate Conversion, Voxel Coordinate Conversion, LocalToVoxel, VoxelToLocal

## Overview
Coordinate Conversion Utilities provide the bidirectional mapping between discrete voxel grid indices and continuous world-space coordinates [S1, S2, S3, unofficial]. They solve the problem of spatial indexing by translating between the logical structure of a voxel grid (integer indices) and the physical or logical coordinate system used by the application (floating-point positions) [S1, S2, unofficial]. What sets this method apart from its siblings is that it operates on the fundamental transformation logic required for any voxel-based system, whereas variants like [Sliced Volume Coordinate Mapping](8287-sliced-volume-coordinate-mapping.md) or [Metric Coordinate Conversion](12960-metric-coordinate-conversion.md) add specific constraints (2D texture flattening, metric scaling) on top of this base conversion [S2, S3, S6, unofficial].

## How it works
The method consists of two primary directions: World-to-Voxel (sampling) and Voxel-to-World (rendering/positioning). Implementations often extend this to handle chunked or hierarchical grids, requiring conversion between global voxel coordinates, local chunk coordinates, and world space [S3, unofficial].

### Inputs and Outputs
*   **World-to-Voxel Input:** A 3D world-space position vector $\mathbf{p}_{world} = (x, y, z)$ [S2, S3, unofficial].
*   **World-to-Voxel Output:** A 3D voxel coordinate vector $\mathbf{v}_{voxel} = (i, j, k)$, which may be global indices or local indices within a specific chunk [S2, S3, unofficial].
*   **Voxel-to-World Input:** A 3D voxel coordinate vector $\mathbf{v}_{voxel}$ (global or local) [S1, S3, unofficial].
*   **Voxel-to-World Output:** A 3D world-space position vector $\mathbf{p}_{world}$, typically representing the center of the voxel [S1, S3, unofficial].

### Data Structures
*   **Transformation Parameters:**
    *   `world_scale`: A scalar or vector defining the size of one voxel in world units [S1, unofficial].
    *   `world_position`: The world-space origin or offset of the voxel grid [S1, unofficial].
    *   `center`: The center offset of the voxel grid, often used to align the grid center with the world origin [S1, unofficial].
    *   `chunk_size`: The dimensions of a chunk in voxels (for chunked grids) [S3, unofficial].
    *   `chunk_coord`: The integer coordinates of the chunk containing the voxel [S3, unofficial].
*   **Coordinate System Enum (Optional):** A `uint32_t` enum representing axis directions for coordinate system alignment, used in specific formats like SPZ [S5, unofficial].

### Formulas

#### 1. Standard Voxel-to-World (Grid-Centered)
Used when the voxel grid is centered at a specific point in world space [S1, unofficial].
$$ \mathbf{p}_{world}[i] = \left( \frac{\mathbf{v}_{voxel}[i] - \mathbf{center}[i]}{\mathbf{world\_scale}} \right) + \mathbf{world\_position}[i] $$
Where:
*   $i \in \{0, 1, 2\}$ corresponds to $x, y, z$.
*   $\mathbf{center}[0] = \text{VOXELS\_X} / 2$, $\mathbf{center}[1] = \text{VOXELS\_Y} / 2$, $\mathbf{center}[2] = 0$ [S1, unofficial].
*   $\mathbf{world\_position}$ is the world-space location of the grid's reference point [S1, unofficial].

#### 2. Standard Voxel-to-World (Origin-Aligned)
Used when the voxel grid origin $(0,0,0)$ aligns with `world_position` [S3, G, unofficial].
$$ \mathbf{p}_{world} = \mathbf{chunk\_origin} + \mathbf{v}_{local} \times \mathbf{voxel\_size} + \frac{\mathbf{voxel\_size}}{2} $$
Where $\mathbf{chunk\_origin}$ is the world-space minimum corner of the chunk containing the voxel [S3, unofficial].

#### 3. Chunked Global/Local Conversion
For chunked voxel worlds, global voxel coordinates $\mathbf{v}_{global}$ are decomposed into chunk coordinates $\mathbf{c}$ and local voxel coordinates $\mathbf{v}_{local}$ [S3, unofficial].

**Global to Local:**
$$ \mathbf{v}_{local}[i] = \mathbf{v}_{global}[i] \pmod{\text{chunk\_size}} $$
If $\mathbf{v}_{local}[i] < 0$, then $\mathbf{v}_{local}[i] \leftarrow \mathbf{v}_{local}[i] + \text{chunk\_size}$ [S3, unofficial].

**Local to Global:**
$$ \mathbf{v}_{global}[i] = \mathbf{c}[i] \times \text{chunk\_size} + \mathbf{v}_{local}[i] $$
Where $\mathbf{c}[i] = \lfloor \mathbf{v}_{global}[i] / \text{chunk\_size} \rfloor$ [S3, unofficial].

#### 4. Coordinate System Bit Encoding (SPZ)
Used to determine axis alignment and rotation requirements between two coordinate systems represented as bitmasks [S5, unofficial].
*   **Axes Match:**
    $$ \text{axesMatch}(a, b) = \{ \text{match}_0, \text{match}_1, \text{match}_2 \} $$
    Where $\text{match}_k = ((a_{num} \gg k) \& 1) == ((b_{num} \gg k) \& 1)$ for $k \in \{0, 1, 2\}$.
    If either system is `UNSPECIFIED` (<0), returns $\{ \text{true}, \text{true}, \text{true} \}$ [S5, unofficial].
*   **Need Rotation:**
    $$ \text{needRotation}(a, b) = ((a_{num} \gg 3) \& 1) \neq ((b_{num} \gg 3) \& 1) $$
    If either is `UNSPECIFIED`, returns `false` [S5, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def world_to_voxel(world_pos, world_scale, world_position, center=None):
    """
    Converts a world-space position to voxel coordinates.
    If center is provided, uses grid-centered conversion [S1, unofficial].
    Otherwise, assumes origin-aligned conversion [S3, unofficial].
    """
    if center is not None:
        # Formula from [S1, unofficial]
        voxel_x = (world_pos[0] - world_position[0]) * world_scale + center[0]
        voxel_y = (world_pos[1] - world_position[1]) * world_scale + center[1]
        voxel_z = (world_pos[2] - world_position[2]) * world_scale + center[2]
        return (voxel_x, voxel_y, voxel_z)
    else:
        # Formula from [S3, unofficial] (simplified for single chunk, no chunk coord)
        # Assumes world_position is the chunk origin
        voxel_x = (world_pos[0] - world_position[0]) / world_scale
        voxel_y = (world_pos[1] - world_position[1]) / world_scale
        voxel_z = (world_pos[2] - world_position[2]) / world_scale
        return (voxel_x, voxel_y, voxel_z)

def voxel_to_world(voxel_pos, world_scale, world_position, center=None):
    """
    Converts voxel coordinates to world-space position.
    """
    if center is not None:
        # Formula from [S1, unofficial]
        world_x = ((voxel_pos[0] - center[0]) / world_scale) + world_position[0]
        world_y = ((voxel_pos[1] - center[1]) / world_scale) + world_position[1]
        world_z = ((voxel_pos[2] - center[2]) / world_scale) + world_position[2]
        return (world_x, world_y, world_z)
    else:
        # Formula from [S3, unofficial] (Voxel Center Precomputation style)
        # Adds half voxel size to get center
        world_x = world_position[0] + (voxel_pos[0] + 0.5) * world_scale
        world_y = world_position[1] + (voxel_pos[1] + 0.5) * world_scale
        world_z = world_position[2] + (voxel_pos[2] + 0.5) * world_scale
        return (world_x, world_y, world_z)

def global_to_local_voxel(global_voxel, chunk_size):
    """
    Converts global voxel coordinates to local chunk coordinates [S3, unofficial].
    """
    local_x = global_voxel[0] % chunk_size
    local_y = global_voxel[1] % chunk_size
    local_z = global_voxel[2] % chunk_size
    
    if local_x < 0:
        local_x += chunk_size
    if local_y < 0:
        local_y += chunk_size
    if local_z < 0:
        local_z += chunk_size
        
    return (local_x, local_y, local_z)

def local_to_global_voxel(local_voxel, chunk_coord, chunk_size):
    """
    Converts local chunk voxel coordinates to global voxel coordinates [S3, unofficial].
    """
    global_x = chunk_coord[0] * chunk_size + local_voxel[0]
    global_y = chunk_coord[1] * chunk_size + local_voxel[1]
    global_z = chunk_coord[2] * chunk_size + local_voxel[2]
    return (global_x, global_y, global_z)

def axes_match(coord_sys_a, coord_sys_b):
    """
    Determines if axes align between two coordinate systems [S5, unofficial].
    """
    UNSPECIFIED = -1
    if coord_sys_a < UNSPECIFIED or coord_sys_b < UNSPECIFIED:
        return (True, True, True)
    
    a_num = coord_sys_a - 1
    b_num = coord_sys_b - 1
    
    match_x = (a_num & 1) == (b_num & 1)
    match_y = ((a_num >> 1) & 1) == ((b_num >> 1) & 1)
    match_z = ((a_num >> 2) & 1) == ((b_num >> 2) & 1)
    
    return (match_x, match_y, match_z)

def need_rotation(coord_sys_a, coord_sys_b):
    """
    Checks if a pi/2 rotation is required between coordinate systems [S5, unofficial].
    """
    UNSPECIFIED = -1
    if coord_sys_a < UNSPECIFIED or coord_sys_b < UNSPECIFIED:
        return False
    
    a_num = coord_sys_a - 1
    b_num = coord_sys_b - 1
    
    return ((a_num >> 3) & 1) != ((b_num >> 3) & 1)
```

### Parameters and Typical Values
*   `world_scale`: Typically 1.0 for unit voxels, or the physical size of a voxel (e.g., 0.01 for 1cm voxels) [S1, S6, unofficial].
*   `chunk_size`: Commonly $16 \times 16 \times 16$ or $32 \times 32 \times 32$ in chunked voxel engines [S3, unofficial].
*   `ground`: In heightmap conversion, typically `'xy'` or `'xz'` [S6, unofficial].

### Edge Cases and Pitfalls
*   **Negative Coordinates:** Modulo operations with negative numbers vary by language. The standard approach is to add the chunk size if the result is negative to ensure positive local indices [S3, unofficial].
*   **Matrix Indexing vs. Isometric Space:** When converting heightmaps, matrices are indexed top-down, while isometric space often increases bottom-up. A vertical flip (`flipy=TRUE`) is often required [S6, unofficial].
*   **Solid Operation Cost:** Making a heightmap "solid" (filling gaps) is computationally expensive in memory and CPU. It should be disabled (`solid=FALSE`) for large or complex heightmaps if performance is critical [S6, unofficial].
*   **Visibility Removal:** Removing non-visible voxels (`check_visibility=TRUE`) can cause issues if the rendering viewpoint changes, as hidden voxels are permanently removed from the coordinate list [S6, unofficial].

## Subtopics

### [Sliced Volume Coordinate Mapping](8287-sliced-volume-coordinate-mapping.md)
Converts 3D voxel coordinates to 2D texture coordinates for tiled 2D slice storage, enabling GPU sampling of 3D data without 3D texture support.

### [Metric Coordinate Conversion](12960-metric-coordinate-conversion.md)
Decomposes world positions into integer meter and fractional meter components for large-scale spatial indexing and high-precision local sampling.

### [Coordinate System Conversion](12990-coordinate-system-conversion.md)
Maps 3D spatial positions to 2D panel coordinates and angular sweep parameters for swept-volume displays.

### [Coordinate Alignment](14251-coordinate-alignment.md)
Shifts voxel coordinates to align statistical properties (min, mean, median, max) with a target location without altering topology.

### Adaptive Coordinate Conversion
A technique used by Lim et al. to visualize 3D ultrasound datasets, where points within the native pyramidal ultrasound coordinate system were adaptively converted on the fly to a Cartesian grid [S4].

### Coordinate System Bit Encoding
A method for determining axis alignment and rotation requirements between coordinate systems using bitmasks. The `CoordinateSystem` enum is a `uint32_t` with 16 valid values (1-16) representing axis directions. Bit 0: Left(0)/Right(1). Bit 1: Down(0)/Up(1). Bit 2: Back(0)/Front(1). Value 0 is UNSPECIFIED. Examples: LDB=1, RDB=2, LUB=3, RUB=4, LDF=5, RDF=6, LUF=7, RUF=8, LFD=9, RFD=10, LFU=11, RFU=12, LBD=13, RBD=14, LBU=15, RBU=16 [S5, unofficial].

### Heightmap Coordinate Calculation
Calculates voxel coordinates from a matrix where values indicate height above the ground. Parameters include `mat` (integer matrix), `fill` (color matrix, optional), `scale` (default 1), `flipx` (default FALSE), `flipy` (default TRUE), `ground` (default 'xy'), `solid` (default TRUE, expensive), and `check_visibility` (default FALSE). Returns a data.frame of voxel coordinates [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for standard coordinate conversions [S1, S3, unofficial]. Chunked conversions involve modulo and division, still $O(1)$ per coordinate [S3, unofficial].
*   **Memory Complexity:** $O(1)$ for the conversion logic itself. Chunked systems require storing chunk coordinates and metadata, scaling with the number of chunks [S3, unofficial].
*   **Strengths:** Simple, fast, and deterministic. Essential for any voxel-based rendering or simulation [S1, S3, unofficial].
*   **Limitations:** Standard conversions assume a uniform grid. Non-uniform or adaptive grids require more complex mapping logic [S4].
*   **When to Use:** Whenever voxel indices need to be mapped to world positions or vice versa [S1, S2, unofficial].
*   **When Not to Use:** For non-Cartesian coordinate systems (e.g., spherical) without modification [S3883-spherical-voxel-grid-setup.md].

## Background
Coordinate conversion is a fundamental operation in voxel-based graphics and simulation. Early implementations focused on simple grid-to-world transformations [S1, unofficial]. With the rise of chunked voxel engines (e.g., Minecraft-like worlds), the need for global/local coordinate conversion became prominent to manage memory and world size [S3, unofficial]. In medical imaging, adaptive conversion was developed to handle non-Cartesian data sources like ultrasound [S4]. In file formats like SPZ, coordinate system encoding was introduced to handle axis alignment and rotation between different software vendors [S5, unofficial]. Heightmap conversion utilities emerged in the R ecosystem for visualizing isometric voxel data [S6, unofficial].

## Sources
[S1] Multivox: Volumetric Display, https://github.com/AncientJames/multivox, fetched 2026-10-26 (unofficial)
[S2] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
[S3] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
[S4] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S5] nianticlabs/spz, https://github.com/nianticlabs/spz, fetched 2026-10-27 (unofficial)
[S6] isocubes: Voxel Data Visualization with Isometric Cubes, https://doi.org/10.32614/cran.package.isocubes, fetched 2026-10-26 (unofficial)
