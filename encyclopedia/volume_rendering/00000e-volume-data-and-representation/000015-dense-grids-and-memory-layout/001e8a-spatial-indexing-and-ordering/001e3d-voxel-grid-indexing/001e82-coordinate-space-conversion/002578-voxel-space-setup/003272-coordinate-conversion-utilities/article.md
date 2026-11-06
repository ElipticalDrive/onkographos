# Coordinate Conversion Utilities

Also called: FVoxelCoordinates, LocalToVoxel, Voxel Chunk Coordinate Conversion, Voxel Coordinate Conversion, VoxelToLocal, voxelVector

## Overview
Coordinate Conversion Utilities provide the bidirectional mapping between world-space positions, chunk indices, and local voxel indices within a hierarchical voxel grid. They solve the problem of addressing data stored in fixed-size spatial chunks by converting continuous world coordinates into discrete grid indices and vice versa. Unlike [Spherical Voxel Grid Setup](3883-spherical-voxel-grid-setup.md), which handles non-Cartesian topologies, these utilities assume a regular Cartesian grid structure. They build on [Voxel Space Setup](3520-voxel-space-setup.md) by utilizing the bounding box and transformation matrices defined there to perform the actual index arithmetic.

## How it works
The method consists of two primary conversion paths: World-to-Voxel and Voxel-to-World, mediated by a Chunk coordinate system. The core data structures are integer vectors representing coordinates and floating-point vectors for world positions.

### Data Structures
*   `WorldPos`: A 3D vector $(x, y, z)$ in world units.
*   `ChunkCoord`: A 3D integer vector $(cx, cy, cz)$ representing the chunk index in the global grid.
*   `LocalVoxel`: A 3D integer vector $(lx, ly, lz)$ representing the voxel index within a single chunk.
*   `GlobalVoxel`: A 3D integer vector $(gx, gy, gz)$ representing the absolute voxel index in the entire world.
*   `ChunkSize`: An integer $N$, the dimension of a cubic chunk (e.g., 16, 32, 64).
*   `VoxelSize`: A float $s$, the physical size of one voxel unit.

### World to Voxel Conversion
This path converts a continuous world position to a discrete local voxel index within a specific chunk.

1.  **World to Global Voxel**: Convert world position to global voxel coordinates.
    $$
    G_x = \lfloor \frac{P_x - O_x}{s} \rfloor, \quad G_y = \lfloor \frac{P_y - O_y}{s} \rfloor, \quad G_z = \lfloor \frac{P_z - O_z}{s} \rfloor
    $$
    Where $P$ is the world position, $O$ is the world-space origin of the voxel grid (minimum corner), and $s$ is the voxel size. The floor function ensures consistent indexing for negative coordinates relative to the origin.
2.  **Global Voxel to Chunk**: Determine which chunk contains the global voxel.
    $$
    C_x = \lfloor \frac{G_x}{N} \rfloor, \quad C_y = \lfloor \frac{G_y}{N} \rfloor, \quad C_z = \lfloor \frac{G_z}{N} \rfloor
    $$
3.  **Global Voxel to Local Voxel**: Calculate the offset within the chunk. This requires handling negative modulo results correctly.
    $$
    L_x = G_x \pmod N
    $$
    If $L_x < 0$, then $L_x = L_x + N$. The same logic applies to $L_y$ and $L_z$.
4.  **Result**: The local voxel coordinates are $(L_x, L_y, L_z)$.

### Voxel to World Conversion
This path converts a local voxel index back to a world position, typically representing the center of the voxel.

1.  **Local to Global Voxel**: Calculate the global voxel index from the chunk coordinate and local position.
    $$
    G_x = C_x \times N + L_x, \quad G_y = C_y \times N + L_y, \quad G_z = C_z \times N + L_z
    $$
2.  **Global Voxel to World Position**: Convert the global voxel index to a world position. To get the voxel center, add 0.5 to the index before scaling.
    $$
    P_x = O_x + (G_x + 0.5) \times s, \quad P_y = O_y + (G_y + 0.5) \times s, \quad P_z = O_z + (G_z + 0.5) \times s
    $$

### Pseudocode
The following implementation combines these steps. It assumes `floor` behaves as mathematical floor (rounding towards negative infinity).

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def world_to_local_voxel(world_pos, grid_origin, voxel_size, chunk_size):
    """
    Converts a world-space position to local voxel coordinates within a chunk.
    
    Args:
        world_pos: tuple (x, y, z) in world units
        grid_origin: tuple (ox, oy, oz) world-space minimum corner of the grid
        voxel_size: float size of one voxel
        chunk_size: int dimension of a chunk (e.g., 32)
        
    Returns:
        tuple (lx, ly, lz) local voxel indices
    """
    # Step 1: World to Global Voxel
    gx = int((world_pos[0] - grid_origin[0]) / voxel_size)
    gy = int((world_pos[1] - grid_origin[1]) / voxel_size)
    gz = int((world_pos[2] - grid_origin[2]) / voxel_size)
    
    # Note: In many languages, integer division truncates towards zero.
    # If the language does not floor negative numbers correctly, use math.floor.
    # Example for Python: gx = math.floor(...)
    
    # Step 2: Global Voxel to Local Voxel (with modulo correction)
    lx = gx % chunk_size
    ly = gy % chunk_size
    lz = gz % chunk_size
    
    if lx < 0:
        lx += chunk_size
    if ly < 0:
        ly += chunk_size
    if lz < 0:
        lz += chunk_size
        
    return (lx, ly, lz)

def local_voxel_to_world(local_voxel, chunk_coord, grid_origin, voxel_size, chunk_size):
    """
    Converts a local voxel position and chunk coordinate to world position (center).
    
    Args:
        local_voxel: tuple (lx, ly, lz)
        chunk_coord: tuple (cx, cy, cz)
        grid_origin: tuple (ox, oy, oz)
        voxel_size: float
        chunk_size: int
        
    Returns:
        tuple (x, y, z) world position
    """
    # Step 1: Local to Global Voxel
    gx = chunk_coord[0] * chunk_size + local_voxel[0]
    gy = chunk_coord[1] * chunk_size + local_voxel[1]
    gz = chunk_coord[2] * chunk_size + local_voxel[2]
    
    # Step 2: Global Voxel to World Position (Center)
    wx = grid_origin[0] + (gx + 0.5) * voxel_size
    wy = grid_origin[1] + (gy + 0.5) * voxel_size
    wz = grid_origin[2] + (gz + 0.5) * voxel_size
    
    return (wx, wy, wz)
```

### Edge Cases and Pitfalls
*   **Negative Modulo**: The modulo operator `%` in many programming languages (C++, Java, JavaScript) returns a negative result if the dividend is negative. The algorithm must explicitly add `ChunkSize` to the result if it is negative to ensure the local coordinate is always within $[0, ChunkSize-1]$ [S2, unofficial].
*   **Boundary Precision**: Floating-point precision errors near chunk boundaries can cause a point to be assigned to the wrong chunk or voxel. Using `floor` consistently for both chunk and voxel calculation is critical [S2, unofficial].
*   **Origin Alignment**: The `grid_origin` must be consistent with the transformation matrices provided by [Voxel Space Setup](3520-voxel-space-setup.md). Mismatched origins will result in systematic offset errors.

## Subtopics

### Adaptive Coordinate Conversion
A technique used by Lim et al. to visualize 3D ultrasound datasets, where points within the native pyramidal ultrasound coordinate system were adaptively converted on the fly to a Cartesian grid [S3]. This variant handles non-Cartesian source data by performing dynamic conversion during the rendering pipeline rather than pre-transforming the entire dataset.

### Coordinate System Bit Encoding
A method for representing and comparing 3D coordinate system orientations using a single 32-bit integer. The `CoordinateSystem` enum uses 16 valid values (1-16) to represent axis directions. Bit 0 determines Left (0) or Right (1). Bit 1 determines Down (0) or Up (1). Bit 2 determines Back (0) or Front (1). Value 0 is UNSPECIFIED. Examples include LDB=1, RDB=2, LUB=3, RUB=4, LDF=5, RDF=6, LUF=7, RUF=8, LFD=9, RFD=10, LFU=11, RFU=12, LBD=13, RBD=14, LBU=15, RBU=16 [S4, unofficial].
*   `axesMatch(a, b)`: Determines if two coordinate systems align. Converts enums to 0-based indices ($aNum = a-1, bNum = b-1$). If either is UNSPECIFIED (<0), returns `{true, true, true}`. Otherwise, compares bit $i$ of $aNum$ and $bNum$ for $i=0,1,2$. Returns a 3-element bool array indicating match per axis [S4, unofficial].
*   `needRotation(a, b)`: Checks if a $\pm \pi/2$ rotation is required. Compares bit 3 of the 0-based indices. If either is UNSPECIFIED, returns false. Otherwise, returns true if bit 3 differs [S4, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(1)$ for all conversion operations. The calculations involve a constant number of arithmetic operations (addition, multiplication, division, modulo).
*   **Memory Complexity**: $O(1)$ auxiliary space. No additional data structures are required beyond the input coordinates and constants.
*   **Performance**: Extremely fast, suitable for per-voxel evaluation in raycasting shaders or CPU-side chunk management.
*   **Strengths**: Simple implementation, no preprocessing required, handles arbitrary world origins and chunk sizes.
*   **Limitations**: Assumes a regular Cartesian grid. Does not handle non-uniform voxel sizes or non-cubic chunks without modification. Precision issues may arise with very large world coordinates due to floating-point limitations.

## Background
Coordinate conversion utilities are fundamental to voxel-based rendering and simulation. The specific implementation details regarding chunk-based modulo arithmetic and local/global coordinate separation are commonly found in procedural voxel world generators, such as those used in Minecraft-like games [S1, S2, unofficial]. The adaptive conversion for non-Cartesian data (e.g., ultrasound) was introduced in medical visualization research to enable standard raycasting techniques on pyramidal datasets [S3]. The bit-encoding scheme for coordinate systems is used in 3D file format parsers to handle axis orientation mismatches between different software vendors [S4, unofficial].

## Sources
[S1] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
[S2] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
[S3] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S4] nianticlabs/spz, https://github.com/nianticlabs/spz, fetched 2026-10-27 (unofficial)
