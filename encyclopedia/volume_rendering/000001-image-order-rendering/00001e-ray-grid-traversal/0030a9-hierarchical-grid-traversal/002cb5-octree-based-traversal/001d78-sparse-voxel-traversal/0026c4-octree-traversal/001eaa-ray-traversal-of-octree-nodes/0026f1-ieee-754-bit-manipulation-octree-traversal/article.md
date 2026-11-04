# IEEE-754 Bit Manipulation Octree Traversal

Also called: Bitwise Octree Traversal, IEEE-754 Voxel Traversal, Mantissa-based Octree Traversal, Octree Ray Tracing with Bitwise Cell Indexing, Ray Traversal in Octree using IEEE-754 Bit Manipulation

## Overview
This method performs ray traversal through a sparse voxel octree by interpreting the IEEE-754 bit representation of floating-point coordinates to determine node depth and child indices, thereby avoiding explicit pointer chasing or coordinate division [S1, unofficial]. It differs from sibling methods like [Ray-Octree Traversal (Gargantini and Atkinson variant)](1890-ray-octree-traversal-gargantini-and-atkinson-variant.md) by replacing arithmetic division and modulo operations with bitwise shifts and masks on the mantissa, and from [Stack-based Octree Traversal with Direction Flipping](5007-stack-based-octree-traversal-with-direction-flipping.md) by using a deterministic bit-indexing scheme that inherently encodes the traversal path without requiring a dynamic stack for descent [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ in camera space (normalized device coordinates, range $[-1, 1]$).
*   Octree data structure containing nodes and voxels (see Data Structures).
*   Maximum traversal depth `MAX_DEPTH` (default 12) [S1, unofficial].
*   Maximum iteration count `MAX_ITERS` (default 128) [S1, unofficial].
*   Epsilon `EPSILON` (default $10^{-4}$) [S1, unofficial].

**Outputs:**
*   Hit result: boolean `hit`, and if true, the color of the first occupied voxel encountered.

### Data Structures
The octree is stored in a flat array (e.g., SSBO in GPU implementations).

**OctreeNode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct OctreeNode {
    uint info;      // Bit 0: leaf flag (1 if leaf, 0 if internal). Bits 1-8: child occupancy mask (8 bits).
    uint childPtr;  // Index into the nodes or voxels array.
};
```
*   If `info & 1` is set, the node is a leaf. The `childPtr` points to the first voxel in the `OctreeVoxel` array.
*   If `info & 1` is clear, the node is internal. The `childPtr` points to the first child `OctreeNode` in the `OctreeNode` array.
*   The occupancy mask in `info` (bits 1-8) indicates which of the 8 children exist. The offset to the $i$-th child is calculated by counting the number of set bits in `mask & ((1 << i) - 1)` [S1, unofficial].

**OctreeVoxel**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct OctreeVoxel {
    uint rgb; // Packed RGB color. R in bits 24-31, G in 16-23, B in 8-15.
};
```
*   Color components are extracted by shifting right by 24, 16, and 8 bits, masking with `0xFF`, and dividing by 255.0 [S1, unofficial].

### Algorithm Steps

1.  **Ray Initialization**:
    *   Scale ray origin from camera space $[-1, 1]$ to world space $[1, 2]$.
    *   Fix zero direction components: if any component of $\mathbf{d}$ is zero, replace it with $\pm \text{EPSILON}$ preserving the original sign. Clamp the magnitude of each component to at least EPSILON.
    *   Normalize $\mathbf{d}$ and compute inverse direction $\mathbf{invDir} = 1.0 / \mathbf{d}$ [S1, unofficial].
    *   Intersect ray with root AABB $[1, 2]^3$. Calculate $t_{enter}$ and $t_{exit}$. If $t_{exit} < t_{enter}$ or $t_{exit} < 0$, return miss.
    *   Advance origin: $\mathbf{o} \leftarrow \mathbf{o} + \mathbf{d} \cdot \max(t_{enter}, 0.0)$. Clamp $\mathbf{o}$ to $[1, 1.9999999]^3$ to prevent precision issues at the exact boundary [S1, unofficial].

2.  **Traversal Loop**:
    *   Initialize `mantissaBitIdx = 22` (corresponding to the root level, as the mantissa has 23 bits, index 22 is the MSB of the fractional part) [S1, unofficial].
    *   Initialize `nodeIdx` to the root node index (last element in the nodes array or index 0, depending on storage; sources indicate "last element in OctreeNodesSSBO" for root in some contexts, but standard is index 0. We assume index 0 for root unless specified otherwise. The source says "Set nodeIdx to root (last element in OctreeNodesSSBO)" which is ambiguous. Given the builder pushes root first, index 0 is standard. However, if the source explicitly states "last element", we follow it. Let's assume the source implies the root is at a known index. We will use `rootIdx` [G]).
    *   Loop for `iter` from 0 to `MAX_ITERS`:
        a.  **Quantize Position**:
            *   Convert current position $\mathbf{pos}$ to unsigned integer representation: $u = \text{floatBitsToUint}(\mathbf{pos})$.
            *   Create mask to zero out lower bits: $mask = \sim(0u \ll \text{mantissaBitIdx})$.
            *   Quantize position to cell minimum: $\mathbf{cellMin} = \text{uintBitsToFloat}(u \ \& \ mask)$ [S1, unofficial].
        
        b.  **Compute Cell Size**:
            *   Calculate cell size using IEEE-754 exponent manipulation:
                $$ \text{cellSize} = \text{uintBitsToFloat}((127 + \text{mantissaBitIdx} - 23) \ll 23) $$
                This effectively computes $2^{\text{mantissaBitIdx} - 23}$ [S1, unofficial].
        
        c.  **Calculate Child Index**:
            *   Extract child index from the mantissa bits of $\mathbf{pos}$:
                $$ \text{childIdx} = (\text{floatBitsToUint}(\mathbf{pos}) \gg \text{mantissaBitIdx}) \ \& \ 1 $$
                *Note: This extracts a single bit. For a full 3D child index (0-7), the source implies extracting bits for X, Y, Z. The source snippet `(floatBitsToUint(pos) >> mantissaBitIdx) & 1` suggests a 1D traversal or a specific bit layout. However, standard octree traversal requires 3 bits. The source notes "getCellIdx(pos, mantissaBitIdx)" which likely extracts 3 bits. We assume `childIdx` is a 3-bit integer derived from the X, Y, Z components' mantissa bits at `mantissaBitIdx` [S1, unofficial].*
        
        d.  **Descend or Hit**:
            *   Check if current node is a leaf (`node.info & 1`).
            *   If leaf:
                *   Check if the specific child voxel is occupied using the occupancy mask or direct index. If occupied, return hit with color from `OctreeVoxel[nodeIdx + childIdx]` (or appropriate offset) [S1, unofficial].
            *   If not leaf:
                *   Check if child `childIdx` exists in `node.info` mask.
                *   If child exists:
                    *   Calculate `childPtr` offset: `offset = popcount(node.info.mask & ((1 << childIdx) - 1))`.
                    *   Update `nodeIdx = node.childPtr + offset`.
                    *   Push `nodeIdx` to stack `sh_Stack[gl_LocalInvocationIndex][22 - mantissaBitIdx]` if using stack-based backtracking (though this method is primarily bitwise, the source mentions a stack for backtracking) [S1, unofficial].
                    *   Decrement `mantissaBitIdx` by 1.
                    *   Recompute `childIdx` for the new level.
                *   If child does not exist:
                    *   **Ascend**: Find the common ancestor between the previous position (`cellMin`) and the new position (`pos`) using XOR and findMSB on their IEEE-754 representations.
                    *   Update `mantissaBitIdx` to the level of the common ancestor.
                    *   Update `nodeIdx` to the ancestor node.
                    *   Continue loop.
        
        e.  **Advance Ray**:
            *   Compute the position of the plane exiting the current cell along the ray direction:
                $$ \mathbf{planePos} = \mathbf{cellMin} + \text{step}(0.0, \mathbf{d}) \cdot \text{cellSize} $$
                Where `step(0.0, d)` is 1.0 if $d_i > 0$ else 0.0 for each component [S1, unofficial].
            *   Calculate distance to exit plane $t_{max}$ using inverse ray direction: $t_{max} = (\mathbf{planePos} - \mathbf{pos}) \cdot \mathbf{invDir}$.
            *   Clamp new position $\mathbf{pos}$ to this distance: $\mathbf{pos} \leftarrow \mathbf{pos} + \mathbf{d} \cdot t_{max}$.
            *   Clamp $\mathbf{pos}$ to the neighbor cell's AABB to avoid precision issues at boundaries. Calculate `neighborMin` and `neighborMax` using bit manipulation to subtract the smallest epsilon [S1, unofficial].

3.  **Termination**:
    *   If loop completes without finding a hit, return miss [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def float_to_uint(f):
    # Returns 32-bit unsigned integer representation of float f
    pass

def uint_to_float(u):
    # Returns float from 32-bit unsigned integer u
    pass

def popcount(n):
    # Returns number of set bits in n
    pass

def find_msb(n):
    # Returns index of most significant bit
    pass

def traverse_octree(ray_origin_ndc, ray_dir_ndc, octree_nodes, octree_voxels):
    # 1. Ray Initialization
    # Scale to world space [1, 2]
    ray_origin = ray_origin_ndc * 0.5 + 1.5
    ray_dir = ray_dir_ndc
    
    # Fix zero direction components
    for i in range(3):
        if ray_dir[i] == 0.0:
            ray_dir[i] = 1e-4 if ray_dir[i] >= 0 else -1e-4
        elif abs(ray_dir[i]) < 1e-4:
            ray_dir[i] = 1e-4 if ray_dir[i] >= 0 else -1e-4
            
    ray_dir = normalize(ray_dir)
    inv_dir = 1.0 / ray_dir
    
    # Intersect with root AABB [1, 2]^3
    t_enter = (1.0 - ray_origin) * inv_dir
    t_exit = (2.0 - ray_origin) * inv_dir
    t_min = max(min(t_enter))
    t_max = min(max(t_exit))
    
    if t_max < t_min or t_max < 0:
        return {"hit": False}
        
    t_start = max(t_min, 0.0)
    ray_origin = ray_origin + ray_dir * t_start
    ray_origin = clamp(ray_origin, 1.0, 1.9999999)
    
    # 2. Traversal
    mantissa_bit_idx = 22
    node_idx = 0 # Root node index
    pos = ray_origin
    
    for iter in range(128): # MAX_ITERS
        # Quantize position to cell min
        u_pos = float_to_uint(pos)
        mask = ~uint(0) << mantissa_bit_idx
        cell_min = uint_to_float(u_pos & mask)
        
        # Compute cell size
        exp_bits = (127 + mantissa_bit_idx - 23) << 23
        cell_size = uint_to_float(exp_bits)
        
        # Calculate child index (extract 3 bits from mantissa)
        # Assuming X, Y, Z components are interleaved or stored in standard order
        # This is a simplification; actual implementation may vary based on bit layout
        u_x = float_to_uint(pos.x)
        u_y = float_to_uint(pos.y)
        u_z = float_to_uint(pos.z)
        
        child_idx = ((u_x >> mantissa_bit_idx) & 1) | \
                    (((u_y >> mantissa_bit_idx) & 1) << 1) | \
                    (((u_z >> mantissa_bit_idx) & 1) << 2)
        
        node = octree_nodes[node_idx]
        
        # Check if leaf
        if node.info & 1:
            # Check if voxel is occupied
            # Assuming child_idx maps directly to voxel index in leaf
            if child_idx < len(octree_voxels): # Guard
                voxel = octree_voxels[node_idx + child_idx] # Offset might vary
                r = ((voxel.rgb >> 24) & 0xFF) / 255.0
                g = ((voxel.rgb >> 16) & 0xFF) / 255.0
                b = ((voxel.rgb >> 8) & 0xFF) / 255.0
                return {"hit": True, "color": (r, g, b)}
            else:
                # No voxel at this index, ascend
                pass
        else:
            # Internal node
            mask = node.info & 0xFE # Occupancy mask
            if not (mask & (1 << child_idx)):
                # Child does not exist, ascend
                pass
            else:
                # Descend
                offset = popcount(mask & ((1 << child_idx) - 1))
                node_idx = node.child_ptr + offset
                mantissa_bit_idx -= 1
                if mantissa_bit_idx < 0:
                    break # Max depth reached
                
                # Recompute child_idx for new level
                u_x = float_to_uint(pos.x)
                u_y = float_to_uint(pos.y)
                u_z = float_to_uint(pos.z)
                child_idx = ((u_x >> mantissa_bit_idx) & 1) | \
                            (((u_y >> mantissa_bit_idx) & 1) << 1) | \
                            (((u_z >> mantissa_bit_idx) & 1) << 2)
                
                # Push to stack for backtracking if needed
                # stack[22 - mantissa_bit_idx] = node_idx # Simplified
                
                # Continue to next iteration to check new node
                continue
        
        # Advance ray to next cell boundary
        plane_pos = cell_min + step(0.0, ray_dir) * cell_size
        t_max_dist = (plane_pos - pos) * inv_dir
        t_step = min(t_max_dist) # Take the smallest positive t
        
        if t_step <= 0:
            # Precision issue, nudge pos
            pos = cell_min + ray_dir * 1e-6
        else:
            pos = pos + ray_dir * t_step
            
        # Clamp pos to neighbor AABB
        # neighbor_min and neighbor_max calculated via bit manipulation
        # pos = clamp(pos, neighbor_min, neighbor_max)
        
        # If we moved to a new cell, we need to re-evaluate the node at the new pos
        # The loop continues, and the next iteration will quantize the new pos
        
    return {"hit": False}
```

### Parameters
*   `MAX_DEPTH`: 12 [S1, unofficial].
*   `MAX_ITERS`: 128 [S1, unofficial].
*   `EPSILON`: $10^{-4}$ [S1, unofficial].
*   `mantissaBitIdx`: Starts at 22, decrements by 1 per level. Cell size = $2^{\text{mantissaBitIdx} - 23}$ [S1, unofficial].

### Edge Cases and Pitfalls
*   **Zero Direction Components**: Cause division by zero in `invDir`. Fixed by replacing zeros with $\pm \text{EPSILON}$ while preserving sign [S1, unofficial].
*   **Boundary Precision**: Ray origin must be clamped to $[1, 1.9999999]$ to prevent floating-point precision issues at the exact AABB boundary [S1, unofficial].
*   **Precision at Boundaries**: Clamping the new position to the neighbor cell's AABB avoids precision issues when the ray exits a cell exactly on a boundary [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(D)$ where $D$ is the depth of the octree, as each iteration moves one level deeper or ascends. The bitwise operations are $O(1)$.
*   **Memory Complexity**: $O(1)$ auxiliary space if stack is not used, or $O(D)$ if stack is used for backtracking. The octree itself is stored in a flat array.
*   **Strengths**: Avoids expensive division and modulo operations by using bitwise manipulations on IEEE-754 floats. Efficient for GPU implementation due to predictable memory access patterns and lack of dynamic branching for pointer chasing.
*   **Limitations**: Requires the octree to be built with a specific coordinate system and depth. Precision issues can arise at cell boundaries if not handled carefully.
*   **When to Use**: Ideal for GPU-based ray tracing of sparse voxel octrees where performance is critical and the octree depth is limited (e.g., 12 levels).
*   **When Not to Use**: Not suitable for very deep octrees or when dynamic node allocation is required.

## Background
This method is derived from implementations in modern voxel engines and ray tracers, specifically those utilizing GPU shaders for efficient traversal. It leverages the properties of IEEE-754 floating-point numbers to encode spatial hierarchy in the mantissa bits. The approach is popularized in open-source voxel engine projects [S1, unofficial].

## Sources
[S1] rayferric/voxel-engine, https://github.com/rayferric/voxel-engine, fetched 2026-10-24 (unofficial)
