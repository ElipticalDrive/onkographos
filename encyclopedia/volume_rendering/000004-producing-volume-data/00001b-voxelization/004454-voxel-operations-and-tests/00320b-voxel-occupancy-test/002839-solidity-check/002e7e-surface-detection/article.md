# Surface Detection


## Overview
Surface detection determines whether a voxel or particle lies on the boundary of a solid or fluid volume. In voxel-based representations, it identifies voxels that have at least one non-solid neighbor in the 6-connected neighborhood [S1, unofficial]. In particle-based fluid simulations, it identifies boundary particles by estimating occlusion on a spherical screen cover, distinguishing interior particles (shadowed by neighbors) from boundary particles (with unshadowed screen areas) [S2]. It differs from general solidity checks by specifically evaluating the *neighborhood* state rather than the local occupancy, serving as a prerequisite for surface tension, boundary rendering, or collision detection.

## How it works

### Voxel-Based Surface Detection
This method operates on a structured voxel grid, typically used in voxel engines or sparse voxel octrees [S1, unofficial].

**Inputs and Outputs**
- **Input**: A voxel map structure `libvxl_map` containing width, height, depth, and chunk data; coordinates $(x, y, z)$.
- **Output**: Boolean `true` if the voxel is on the surface, `false` otherwise.

**Data Structures**
The map is organized into chunks to handle large volumes efficiently [S1, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```c
struct libvxl_map {
    size_t width, height, depth;
    struct libvxl_chunk* chunks;
    size_t* geometry; // Bitmask array for occupancy/solidity
    size_t streamed;
};

struct libvxl_chunk {
    struct libvxl_block* blocks;
    size_t length, index;
};

struct libvxl_block {
    uint32_t position;
    uint32_t color;
};
```
The `geometry` array acts as a bitmask where each bit represents the solidity of a voxel. The function `libvxl_map_issolid` checks these bits [S1, unofficial].

**Algorithm Steps**
1.  **Null Check**: If the map pointer is null, return `false` [S1, unofficial].
2.  **Boundary Check**: Ensure $(x, y, z)$ is within map bounds $[0, width) \times [0, height) \times [0, depth)$. If out of bounds, the voxel is considered on the surface (or the behavior is undefined; standard practice treats out-of-bounds as empty/non-solid) [G].
3.  **Neighbor Evaluation**: Check the 6-connected neighbors: $(x, y+1, z)$, $(x, y-1, z)$, $(x+1, y, z)$, $(x-1, y, z)$, $(x, y, z+1)$, $(x, y, z-1)$ [S1, unofficial].
4.  **Solidity Query**: For each neighbor, call `libvxl_map_issolid`.
5.  **Result**: If *any* neighbor returns `false` (is not solid), return `true` (is on surface). If all 6 neighbors are solid, return `false` [S1, unofficial].

**Formula**
$$ \text{onsurface}(x,y,z) = \neg \text{solid}(x,y+1,z) \lor \neg \text{solid}(x,y-1,z) \lor \neg \text{solid}(x+1,y,z) \lor \neg \text{solid}(x-1,y,z) \lor \neg \text{solid}(x,y,z+1) \lor \neg \text{solid}(x,y,z-1) $$
where $\text{solid}(x,y,z)$ is true if the voxel at $(x,y,z)$ is occupied and not air [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def libvxl_map_onsurface(map, x, y, z):
    if map is None:
        return False
    
    # Check bounds. Out of bounds is treated as non-solid.
    if not (0 <= x < map.width and 0 <= y < map.height and 0 <= z < map.depth):
        return True

    # Define 6-connected neighbors
    neighbors = [
        (x, y + 1, z),
        (x, y - 1, z),
        (x + 1, y, z),
        (x - 1, y, z),
        (x, y, z + 1),
        (x, y, z - 1)
    ]

    for nx, ny, nz in neighbors:
        # Check if neighbor is out of bounds (treated as non-solid)
        if not (0 <= nx < map.width and 0 <= ny < map.height and 0 <= nz < map.depth):
            return True
        
        # Check if neighbor is solid
        if not libvxl_map_issolid(map, nx, ny, nz):
            return True

    return False
```

### Voxel Air Setting (Variant)
This variant handles the dynamic update of surface status when a voxel is set to air [S1, unofficial].

**Steps**
1.  **Bounds Check**: Verify $(x, y, z)$ is within bounds [S1, unofficial].
2.  **Record Previous State**: For each of the 6 neighbors, record `surface_prev[i]`. A neighbor is considered "previously on surface" if it is not solid OR if it is solid but already on surface [S1, unofficial].
3.  **Set Air**: Call internal function `libvxl_map_setair_internal` to set the voxel to air and reset the geometry bit to 0 [S1, unofficial].
4.  **Update Neighbors**: For each neighbor $i$:
    -   If `!surface_prev[i]` (was not on surface) AND the neighbor is now on surface (due to the air voxel), call `libvxl_map_set_internal` with `DEFAULT_COLOR` to update its state [S1, unofficial].

**Pitfall**
`surface_prev` defaults to `true` if the neighbor is not solid. This prevents false restoration of air blocks for neighbors that were already empty/non-solid [S1, unofficial].

### Particle-Based Surface Tension Detection
This method detects fluid surface particles in Smoothed Particle Hydrodynamics (SPH) or Position-Based Dynamics (PBD) by estimating occlusion on a spherical screen [S2].

**Inputs and Outputs**
- **Input**: Particle position $p_i$, neighbor positions $p_j$, particle radius $R$, distance threshold $d_0$.
- **Output**: Boolean indicating if particle $i$ is on the surface.

**Data Structures**
-   **Screen**: Parameterized as an $18 \times 36$ environment map. Each column corresponds to 18 bits of an integer. 36 integers are used to mask the screen [S2].

**Algorithm Steps**
1.  **Encapsulation**: Encapsulate particle $i$ with a spherical cover of radius $R$ [S2].
2.  **Projection**: For each neighboring particle $j$, calculate the vector $\Delta p = p_j - p_i$ [S2].
3.  **Angle Calculation**: Calculate projection angles $\phi$ and $\theta$ on the spherical screen [S2]:
    -   $\phi = \tan^{-1}\left(\frac{\Delta p_y}{\sqrt{\Delta p_x^2 + \Delta p_z^2}}\right)$
    -   $\theta = \tan^{-1}\left(\frac{\Delta p_z}{\Delta p_x}\right)$
    -   $\Delta \theta = \tan^{-1}\left(\frac{R}{\sqrt{\|\Delta p\|^2 - R^2}}\right)$
    -   $\Delta \phi = \Delta \theta$ [S2]
4.  **Masking**: The occluding area on the screen is $[\theta - \Delta \theta, \theta + \Delta \theta] \times [\phi - \Delta \phi, \phi + \Delta \phi]$. Mask the corresponding bits in the 36 integers representing the environment map [S2].
5.  **Counting**: Count the mask ratio (shadowed area) [S2].
6.  **Decision**: If the total projection area from neighbors is below a given threshold, the particle is considered on the fluid surface [S2].

**Formula**
$$ \Delta p_y = p_{j,y} - p_{i,y}, \quad \Delta p_x = p_{j,x} - p_{i,x}, \quad \Delta p_z = p_{j,z} - p_{i,z} $$
$$ \|\Delta p\|^2 = \Delta p_x^2 + \Delta p_y^2 + \Delta p_z^2 $$
$$ \phi = \tan^{-1}\left(\frac{\Delta p_y}{\sqrt{\Delta p_x^2 + \Delta p_z^2}}\right) $$
$$ \theta = \tan^{-1}\left(\frac{\Delta p_z}{\Delta p_x}\right) $$
$$ \Delta \theta = \tan^{-1}\left(\frac{R}{\sqrt{\|\Delta p\|^2 - R^2}}\right) $$
$$ \Delta \phi = \Delta \theta $$ [S2]

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def detect_surface_particle(p_i, neighbors, R, d0):
    # Initialize 36 integers for the 18x36 environment map
    screen_mask = [0] * 36
    
    for p_j in neighbors:
        dx = p_j.x - p_i.x
        dy = p_j.y - p_i.y
        dz = p_j.z - p_i.z
        
        dist_sq = dx*dx + dy*dy + dz*dz
        
        # Avoid division by zero or invalid geometry
        if dist_sq <= R*R:
            continue
            
        # Calculate angles
        phi = atan2(dy, sqrt(dx*dx + dz*dz))
        theta = atan2(dz, dx)
        
        # Calculate angular radius of the neighbor's projection
        delta_theta = atan2(R, sqrt(dist_sq - R*R))
        delta_phi = delta_theta
        
        # Map angles to screen coordinates
        # Assuming phi in [-pi/2, pi/2] maps to 0..17
        # Assuming theta in [-pi, pi] maps to 0..35
        row = int((phi + pi/2) / pi * 18)
        col = int((theta + pi) / (2*pi) * 36)
        
        # Clamp row/col to 0..17 and 0..35
        row = max(0, min(17, row))
        col = max(0, min(35, col))
        
        # Mask the bit in the integer for this column
        # Each integer holds 18 bits (rows)
        bit_pos = row
        screen_mask[col] |= (1 << bit_pos)
        
    # Count mask ratio (shadowed area)
    total_bits = 18 * 36
    masked_bits = sum(bin(m).count('1') for m in screen_mask)
    ratio = masked_bits / total_bits
    
    # Threshold check (value not specified in source, assume generic threshold)
    # If ratio is low, particle is on surface
    return ratio < 0.5 # Placeholder threshold
```

## Subtopics

### Voxel Air Setting
Sets a voxel to air, updates the geometry bit, and restores surface status to previously covered neighbors. It differs from standard surface detection by actively modifying the map state and propagating surface updates to neighbors. Choose this when dynamically modifying voxel geometry and maintaining consistent surface flags is required. [S1, unofficial]

### Surface Tension Detection
Detects fluid surface particles by estimating occlusion on a spherical screen cover. It differs from voxel-based detection by operating on unstructured particles and using geometric projection rather than grid adjacency. Choose this for particle-based fluid simulations (PBD/SPH) where surface tension forces need to be applied only to boundary particles. [S2]

## Operating characteristics
**Time Complexity**:
-   Voxel-based: $O(1)$ per voxel, as it checks exactly 6 neighbors [S1, unofficial].
-   Particle-based: $O(N)$ per particle, where $N$ is the number of neighbors, due to the loop over neighbors and spherical projection calculations [S2].

**Memory Complexity**:
-   Voxel-based: Depends on the map structure. The `libvxl_map` stores chunks and a geometry bitmask [S1, unofficial].
-   Particle-based: Requires storing neighbor lists and temporary screen masks (36 integers) per particle [S2].

**Performance**:
-   Voxel-based detection is highly parallelizable and efficient on GPUs, as each voxel's surface status is independent [S1, unofficial].
-   Particle-based detection can also be parallelized on the GPU, as the occlusion calculation for each particle is independent of others [S2].

**Strengths**:
-   Voxel-based: Simple, robust, and fast for grid-based data.
-   Particle-based: Accurate for fluid dynamics, captures surface curvature via spherical projection.

**Limitations**:
-   Voxel-based: Discrete nature may lead to jagged surfaces; requires careful handling of boundary conditions.
-   Particle-based: Computationally more expensive due to trigonometric functions and neighbor searches; sensitive to particle distribution.

## Background
The voxel-based surface detection method originates from the `libvxl` library, an unofficial C library for voxel manipulation [S1, unofficial]. The particle-based surface tension detection is described in the "Gaussian Splashing" paper, which extends Position-Based Dynamics (PBD) with surface tension for fluid simulation [S2]. The Gaussian Splashing paper cites [70] for the original surface tension model, which was implemented on the CPU but is noted to be parallelizable on the GPU [S2].

## Sources
[S1] xtreme8000/libvxl, https://github.com/xtreme8000/libvxl, fetched 2026-10-28 (unofficial)
[S2] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
