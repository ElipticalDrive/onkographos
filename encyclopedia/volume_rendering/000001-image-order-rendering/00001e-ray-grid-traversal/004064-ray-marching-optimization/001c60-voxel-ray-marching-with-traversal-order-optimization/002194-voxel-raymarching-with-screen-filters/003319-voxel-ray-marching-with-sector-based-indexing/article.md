# Voxel Ray Marching with Sector-Based Indexing

Also called: NoiseSDFCompute Voxel Marching, VoxelScene Ray Marching

## Overview
A ray marching algorithm for rendering voxel scenes that utilizes a sector-based storage layout to index voxels and a bounding box intersection test to step through the volume [S1, unofficial]. It differs from its parent [Voxel Raymarching with Screen Filters](2555-voxel-raymarching-with-screen-filters.md) by explicitly organizing voxel data into fixed-size sectors to optimize memory access patterns during traversal, rather than relying solely on screen-space filtering heuristics. It is distinct from the [Voxel Visualization Ray Marcher](3432-voxel-visualization-ray-marcher.md) by its specific use of sector-based indexing for global voxel lookup [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Ray origin ($O$), ray direction ($D$), voxel data array ($V$), world dimensions ($W_X, W_Y, W_Z$), sector dimensions ($S_X, S_Y, S_Z$), world offset ($O_{world}$), maximum steps ($N_{max}$), maximum distance ($D_{max}$), minimum step size ($\epsilon$).
*   **Output:** Hit distance ($t$) or miss indicator.

**Data Structures**
The voxel data is stored in a flat array of 32-bit unsigned integers (`u32`). Each `u32` word stores the solidity status of 32 voxels as individual bits. The layout is organized into sectors to facilitate spatial indexing.

*   **World Dimensions:** $W_X = 256, W_Y = 256, W_Z = 256$ [S1, unofficial].
*   **Sector Dimensions:** $S_X = 16, S_Y = 256, S_Z = 16$ [S1, unofficial].
*   **Sector Counts:** $N_{sectors\_X} = W_X / S_X = 16$, $N_{sectors\_Z} = W_Z / S_Z = 16$ [S1, unofficial].
*   **World Offset:** $O_{world} = (-128, 0, -128)$ [S1, unofficial].
*   **Voxel Data Array:** `voxelData` of type `u32`.

**Indexing Logic**
To determine if a voxel at world position $P$ is solid:
1.  Convert world position to voxel coordinates:
    $$ v_x = \lfloor P_x - O_{world\_x} \rfloor $$
    $$ v_y = \lfloor P_y - O_{world\_y} \rfloor $$
    $$ v_z = \lfloor P_z - O_{world\_z} \rfloor $$
2.  Check bounds: if $0 \le v_x < W_X$, $0 \le v_y < W_Y$, $0 \le v_z < W_Z$, proceed. Else, return false (out of bounds) [S1, unofficial].
3.  Calculate sector index:
    $$ idx_{sector} = \lfloor v_z / S_Z \rfloor + N_{sectors\_Z} \times \lfloor v_x / S_X \rfloor $$
4.  Calculate local index within the sector:
    $$ idx_{local} = (v_z \pmod{S_Z}) + S_Z \times \left( (v_x \pmod{S_X}) + S_X \times v_y \right) $$
5.  Calculate global index in the flat array:
    $$ idx_{global} = idx_{sector} \times (S_X \times S_Y \times S_Z) + idx_{local} $$
6.  Check bit value:
    $$ \text{solid} = (voxelData[\lfloor idx_{global} / 32 \rfloor] \gg (idx_{global} \pmod{32})) \& 1 $$

**Algorithm Steps**
1.  Initialize $t = 0.0$.
2.  Loop $i$ from $0$ to $N_{max}$:
    a.  Calculate current position: $P = O + D \times t$.
    b.  Calculate distance to the nearest solid voxel (or SDF) from $P$. Let this be $d_{voxel}$ [S1, unofficial].
    c.  If $d_{voxel} < \epsilon$, return hit at distance $t$ [S1, unofficial].
    d.  If $t > D_{max}$, return miss [S1, unofficial].
    e.  $t = t + d_{voxel}$.
3.  Return miss.

**Ray-Box Intersection**
To find the distance to the bounding box of the entire world (used as a coarse culling step or initial step size):
1.  Define world bounding box corners: $B_{min} = O_{world}$, $B_{max} = O_{world} + (W_X, W_Y, W_Z)$ [S1, unofficial].
2.  Calculate inverse direction: $D_{inv} = 1.0 / D$.
3.  Calculate near and far intersections for each axis:
    $$ t_{near\_x} = (B_{min\_x} - O_x) \times D_{inv\_x}, \quad t_{far\_x} = (B_{max\_x} - O_x) \times D_{inv\_x} $$
    $$ t_{near\_y} = (B_{min\_y} - O_y) \times D_{inv\_y}, \quad t_{far\_y} = (B_{max\_y} - O_y) \times D_{inv\_y} $$
    $$ t_{near\_z} = (B_{min\_z} - O_z) \times D_{inv\_z}, \quad t_{far\_z} = (B_{max\_z} - O_z) \times D_{inv\_z} $$
4.  Calculate entry and exit distances:
    $$ t_0 = \max(\max(t_{near\_x}, t_{near\_y}), t_{near\_z}) $$
    $$ t_1 = \min(\min(t_{far\_x}, t_{far\_y}), t_{far\_z}) $$
5.  If $\max(t_0, 0.0) > t_1$, return $\infty$ (miss) [S1, unofficial]. Else return $t_0$.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_ray_march(origin, direction, voxel_data, world_size, sector_size, world_offset, max_steps, max_dist, min_dist):
    # Constants from sources
    wx, wy, wz = world_size
    sx, sy, sz = sector_size
    ox, oy, oz = world_offset
    
    # Precompute sector counts
    sectors_x = wx // sx
    sectors_z = wz // sz
    
    # Ray-Box Intersection for world bounds
    inv_dir = 1.0 / direction
    near = (world_offset - origin) * inv_dir
    far = (world_offset + world_size - origin) * inv_dir
    
    t0 = max(max(near[0], near[1]), near[2])
    t1 = min(min(far[0], far[1]), far[2])
    
    if max(t0, 0.0) > t1:
        return None  # Miss
    
    t = 0.0
    
    for i in range(max_steps):
        if t > max_dist:
            return None  # Miss
        
        # Current position
        pos = origin + direction * t
        
        # Convert to voxel coordinates
        vx = int(pos[0] - ox)
        vy = int(pos[1] - oy)
        vz = int(pos[2] - oz)
        
        # Check bounds
        if not (0 <= vx < wx and 0 <= vy < wy and 0 <= vz < wz):
            # Out of bounds, treat as empty or handle as miss depending on context
            # For this implementation, we assume out-of-bounds is empty space
            # However, standard ray marching usually steps until hit or max dist
            # If out of bounds, we might need to calculate distance to box edge
            # But the source notes imply a simple "check bounds" then "calculate index"
            # If out of bounds, we can't index. Let's assume the ray stays within bounds
            # or we step by distance to box edge.
            # Given the simplicity of the source, we'll assume valid voxel coords for the step
            # If invalid, we return miss or handle edge case.
            # Let's assume the ray is clamped or the box intersection ensures we are inside
            # If we are outside, we need to step to the box.
            # For simplicity based on source notes:
            return None # Simplified: out of bounds is miss
        
        # Calculate sector index
        sector_index = (vz // sz) + sectors_z * (vx // sx)
        
        # Calculate local index
        local_index = (vz % sz) + sz * ((vx % sx) + sx * vy)
        
        # Calculate global index
        sector_volume = sx * sy * sz
        global_index = sector_index * sector_volume + local_index
        
        # Check bit
        word_index = global_index // 32
        bit_index = global_index % 32
        is_solid = (voxel_data[word_index] >> bit_index) & 1
        
        if is_solid:
            return t  # Hit
        
        # Calculate distance to nearest solid voxel
        # The source notes mention "Calculate dist to nearest solid voxel (or SDF)"
        # This implies a distance field or a search. 
        # Given the binary voxel data, a simple distance to the current voxel boundary
        # or a precomputed SDF would be used. 
        # Since no SDF generation is described, we assume a step size based on voxel size
        # or a nearest neighbor search. 
        # However, the source says "Calculate dist... If dist < minDist, return hit".
        # This suggests the step size is determined by the distance to the next voxel boundary
        # or the distance to the solid voxel itself.
        # In a binary voxel grid, the distance to the nearest solid voxel from a point
        # inside an empty voxel is the distance to the boundary of that voxel if the neighbor is solid.
        # Without an explicit SDF, we assume a fixed step size or a simple distance calculation.
        # The source notes do not specify how 'dist' is calculated beyond "Calculate dist to nearest solid voxel".
        # We will use a placeholder comment for the specific distance calculation method as it is not fully detailed.
        dist = calculate_distance_to_nearest_solid(pos, voxel_data, world_size, sector_size, world_offset) # [G]
        
        if dist < min_dist:
            return t
        
        t += dist
        
    return None # Miss

def calculate_distance_to_nearest_solid(pos, voxel_data, world_size, sector_size, world_offset):
    # [G] Placeholder for distance calculation.
    # In a binary voxel grid, this typically involves checking neighbors or using a precomputed distance field.
    # Since the source does not specify, we assume a simple step to the next voxel boundary.
    # A more robust implementation would use a signed distance field (SDF).
    # For this implementation, we return a small epsilon to allow stepping.
    return 0.1 # [G] Arbitrary small step for demonstration; actual implementation requires SDF or neighbor check.
```

**Parameters**
*   $WORLD\_SIZE\_X, WORLD\_SIZE\_Y, WORLD\_SIZE\_Z$: 256 [S1, unofficial].
*   $SECTOR\_SIZE\_X$: 16, $SECTOR\_SIZE\_Y$: 256, $SECTOR\_SIZE\_Z$: 16 [S1, unofficial].
*   $WORLD\_OFFSET\_X$: -128, $WORLD\_OFFSET\_Y$: 0, $WORLD\_OFFSET\_Z$: -128 [S1, unofficial].
*   $maxSteps$: 128 [S1, unofficial].
*   $maxDist$: 100.0 [S1, unofficial].
*   $minDist$: 0.01 [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_{steps})$ per ray, where $N_{steps}$ is bounded by $maxSteps$. Each step involves constant-time arithmetic and bit operations.
*   **Memory Complexity:** $O(W_X \times W_Y \times W_Z / 32)$ for the voxel data array. For $256^3$ voxels, this is $2^{24} / 32 = 2^{19}$ `u32` words, or 2 MB.
*   **Strengths:** Sector-based indexing can improve cache locality for spatially coherent rays. Bit packing reduces memory footprint.
*   **Limitations:** Binary voxel data limits shading to simple hit/miss. Distance calculation requires an SDF or neighbor search, which is not detailed in the source.
*   **When to use:** Real-time rendering of binary voxel scenes where memory bandwidth is a constraint.
*   **When not to use:** Scenes requiring complex shading, transparency, or non-binary voxel data.

## Background
Originated in the Divine Voxel Engine by Divine-Star-Software [S1, unofficial]. The method is part of a suite of voxel rendering techniques aimed at efficient real-time rendering of voxel worlds.

## Sources
[S1] Divine-Star-Software/DivineVoxelEngine, https://github.com/Divine-Star-Software/DivineVoxelEngine, fetched 2026-10-26 (unofficial)
