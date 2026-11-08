# Position Error Minimized Marching Cubes

Also called: UMC, Unified Marching Cubes

## Overview
Position Error Minimized Marching Cubes (PEM-MC) is an isosurface extraction algorithm that reduces geometric error by snapping grid vertices to the exact isosurface location when they lie sufficiently close to the threshold. It builds on [Marching Cubes on Structured Grids](6051-marching-cubes-on-structured-grids.md) by introducing a 2-bit sign encoding and a post-processing vertex snapping step. Unlike [Bitwise Marching Cubes](7374-bitwise-marching-cubes.md) which prioritizes branchless logic, or [Flying Edges](7520-flying-edges.md) which optimizes for streaming memory access, PEM-MC prioritizes vertex position accuracy for structured grids.

## How it works
### Inputs and Outputs
**Inputs:**
- Scalar field $f$ defined on a structured grid of dimensions $(dim_x+1) \times (dim_y+1) \times (dim_z+1)$.
- Isosurface threshold value $I$ (ISOLEVEL).
- Boolean flag `pem` enabling Position Error Minimization.
- Snapping threshold `snap_threshold` (float).
- Sampler function `sampler_fn(fx, fy, fz, w, osn)` returning the scalar value at world coordinates.

**Outputs:**
- A set of triangles representing the isosurface.
- Vertex positions adjusted via snapping logic.

### Data Structures
The method uses the following structures, consistent with the parent article but extended for PEM mode:

1.  **`grid_signs`**: Array of `uint16_t`, size $(dim_x/2) \times (dim_y/2) \times (dim_z/2)$.
    -   Stores signs for 8 grid vertices per 16-bit word.
    -   **Standard Mode**: 1 bit per vertex.
    -   **PEM Mode**: 2 bits per vertex. The bits are packed based on the vertex parity $(x\&1, y\&1, z\&1)$ within a $2\times2\times2$ block.
    -   Sign encoding in PEM mode:
        -   `0`: $s < I$
        -   `1`: $s = I$
        -   `2`: $s > I$

2.  **`grid_verts`**: Array of struct `UMC_Isovertex`, size $(dim_x+1) \times (dim_y+1) \times (dim_z+1)$.
    -   Fields:
        -   `value` (float): The scalar value at the vertex.
        -   `index` (int32): Output index. Set to `-2` to mark a vertex for snapping.
        -   `position` (vec3): The 3D coordinates of the vertex.

3.  **`edges`**: Array of struct `UMC_Edge`, size $(dim_x+1) \times (dim_y+1) \times (dim_z+1) \times 3$.
    -   Fields:
        -   `grid_v0` (int32): Index into `grid_verts` for the start vertex.
        -   `grid_v1` (int32): Index into `grid_verts` for the end vertex.

### Algorithm Steps

1.  **Grid Labeling**:
    Iterate over all grid vertices $(x, y, z)$ from $0$ to $dim$.
    -   Compute floating-point coordinates $(fx, fy, fz)$.
    -   Sample the scalar field: $s = \text{sampler\_fn}(fx, fy, fz, w, osn)$.
    -   If `pem` is false:
        -   Set a 1-bit sign in `grid_signs`: $0$ if $s < I$, $1$ if $s \ge I$.
    -   If `pem` is true:
        -   Set a 2-bit sign in `grid_signs`:
            -   $0$ if $s < I$
            -   $1$ if $s = I$
            -   $2$ if $s > I$

2.  **Edge Labeling and Crossing Detection**:
    Iterate over all grid vertices and check the three outgoing edges (X, Y, Z directions).
    -   For each edge, call `_UMC_Chunk_calc_edge_crossing`.
    -   If a crossing is detected OR the PEM bit 4 is set (indicating a vertex is exactly on the isosurface):
        -   Compute the isovector intersection via `_UMC_Chunk_calc_edge_isov`.
        -   Store the intersection details in the `edges` array.

3.  **Vertex Snapping Marking (PEM Only)**:
    After computing edge crossings, if `pem` is true:
    -   If `result_mask & 1` is true, call `_UMC_Chunk_set_isov` for the start vertex.
    -   If `result_mask & 2` is true, call `_UMC_Chunk_set_isov` for the end vertex.
    -   This marks vertices for snapping by setting their `index` in `grid_verts` to `-2`.

4.  **Vertex Snapping**:
    Iterate over output indices in `grid_verts`.
    -   Extract grid coordinates $(x, y, z)$.
    -   **Boundary Check**: If $x=0$, $x=dim_x$, $y=0$, $y=dim_y$, $z=0$, or $z=dim_z$, skip snapping for this vertex [S1, unofficial].
    -   If not on the boundary and `index` is `-2`:
        -   Adjust the vertex position towards the exact isosurface intersection point.
        -   The adjustment is controlled by `snap_threshold`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def position_error_minimized_marching_cubes(grid_dims, isovalue, snap_threshold, sampler_fn):
    """
    grid_dims: tuple (dim_x, dim_y, dim_z)
    isovalue: float I
    snap_threshold: float
    sampler_fn: function returning scalar at (fx, fy, fz)
    """
    dim_x, dim_y, dim_z = grid_dims
    
    # Initialize data structures
    # grid_signs size: (dim_x/2) * (dim_y/2) * (dim_z/2)
    grid_signs = [0] * ((dim_x // 2) * (dim_y // 2) * (dim_z // 2))
    
    # grid_verts size: (dim_x+1) * (dim_y+1) * (dim_z+1)
    grid_verts = []
    for z in range(dim_z + 1):
        for y in range(dim_y + 1):
            for x in range(dim_x + 1):
                grid_verts.append({
                    'value': 0.0,
                    'index': -1,
                    'position': (x, y, z), # Placeholder, actual world coords computed later
                    'is_snapped': False
                })
                
    # edges array is implicit in the parent algorithm's chunk processing
    # We focus on the PEM-specific steps
    
    # Step 1: Grid Labeling
    for z in range(dim_z + 1):
        for y in range(dim_y + 1):
            for x in range(dim_x + 1):
                # Compute world coordinates (assuming unit spacing for simplicity)
                fx, fy, fz = float(x), float(y), float(z)
                s = sampler_fn(fx, fy, fz)
                
                # Store value
                idx = z * (dim_x + 1) * (dim_y + 1) + y * (dim_x + 1) + x
                grid_verts[idx]['value'] = s
                
                # Compute sign index for grid_signs
                # Pack 8 vertices into one uint16_t
                block_x = x // 2
                block_y = y // 2
                block_z = z // 2
                bit_offset = (x & 1) | ((y & 1) << 1) | ((z & 1) << 2)
                
                sign_val = 0
                if s < isovalue:
                    sign_val = 0
                elif s == isovalue:
                    sign_val = 1
                else:
                    sign_val = 2
                
                # Update grid_signs (bitwise OR with shifted sign_val)
                # Note: In C, this would be bit manipulation. Here we simulate.
                # grid_signs[block_z * (dim_x//2)*(dim_y//2) + block_y * (dim_x//2) + block_x]
                # |= (sign_val << (2 * bit_offset))
                
    # Step 2 & 3: Edge Labeling and Snapping Marking
    # This part is handled by the parent algorithm's chunk processing.
    # We simulate the marking of vertices for snapping.
    
    for z in range(dim_z + 1):
        for y in range(dim_y + 1):
            for x in range(dim_x + 1):
                idx = z * (dim_x + 1) * (dim_y + 1) + y * (dim_x + 1) + x
                s = grid_verts[idx]['value']
                
                # Check if vertex is exactly on isosurface (PEM specific)
                if s == isovalue:
                    # Mark for snapping
                    grid_verts[idx]['index'] = -2
                    
    # Step 4: Vertex Snapping
    for z in range(dim_z + 1):
        for y in range(dim_y + 1):
            for x in range(dim_x + 1):
                idx = z * (dim_x + 1) * (dim_y + 1) + y * (dim_x + 1) + x
                
                # Boundary check
                if x == 0 or x == dim_x or y == 0 or y == dim_y or z == 0 or z == dim_z:
                    continue
                
                if grid_verts[idx]['index'] == -2:
                    # Snap vertex to isosurface
                    # The exact snapping logic depends on _UMC_Chunk_snap_verts
                    # which uses snap_threshold to adjust position.
                    # [G] Assuming a simple projection towards the gradient direction
                    # or linear interpolation along edges for demonstration.
                    # In practice, this involves finding the nearest point on the 
                    # isosurface within snap_threshold distance.
                    
                    # Placeholder for actual snapping math
                    # grid_verts[idx]['position'] = snap_to_isosurface(grid_verts[idx], isovalue, snap_threshold)
                    grid_verts[idx]['index'] = idx # Reset index to valid output index after snapping

    return grid_verts
```

### Parameters
-   `pem`: Boolean. Enables 2-bit sign encoding and vertex snapping.
-   `ISOLEVEL`: Float. The threshold value for scalar field comparison.
-   `snap_threshold`: Float. Controls the maximum distance for vertex adjustment during snapping.

### Edge Cases and Pitfalls
-   **Boundary Vertices**: Vertices on the boundary of the grid ($x=0, x=dim_x, y=0, y=dim_y, z=0, z=dim_z$) are skipped during snapping to prevent out-of-bounds access or incorrect geometric adjustments [S1, unofficial].
-   **Precision**: Floating-point equality check ($s == I$) may be unreliable due to precision errors. In practice, a tolerance should be used, though the source notes specify exact equality [S1, unofficial].
-   **Sign Encoding**: In PEM mode, the 2-bit encoding allows for three states, which is crucial for identifying vertices exactly on the isosurface.

## Operating characteristics
-   **Time Complexity**: $O(dim^3)$ for labeling and edge processing [S1, unofficial].
-   **Memory Complexity**: $O(dim^3)$ for grid vertices, edges, and signs [S1, unofficial].
-   **Strengths**: Provides higher geometric accuracy by snapping vertices to the isosurface, reducing position error compared to standard linear interpolation.
-   **Limitations**: Slightly higher memory usage due to 2-bit sign encoding and additional snapping logic. Boundary conditions require special handling.

## Background
Position Error Minimized Marching Cubes is part of the "Unified Marching Cubes" (UMC) approach, aiming to combine the efficiency of bitwise operations with the accuracy of position error minimization. It is described in the context of the "Pushing Voxels Forward" project [S1, unofficial].

## Sources
[S1] Lin20/PushingVoxelsForward, https://github.com/Lin20/PushingVoxelsForward, fetched 2026-10-27 (unofficial)
