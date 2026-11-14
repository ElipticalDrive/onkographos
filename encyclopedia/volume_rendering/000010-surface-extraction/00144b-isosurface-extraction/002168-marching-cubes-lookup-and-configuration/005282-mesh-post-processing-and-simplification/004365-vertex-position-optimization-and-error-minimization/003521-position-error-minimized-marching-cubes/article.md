# Position Error Minimized Marching Cubes

Also called: UMC, Unified Marching Cubes

## Overview
Position Error Minimized Marching Cubes (PEM-MC), also known as Unified Marching Cubes (UMC), is an isosurface extraction algorithm that reduces geometric error by snapping grid vertices to the exact isosurface location when they lie sufficiently close to the threshold [S1, unofficial]. It builds on [Marching Cubes on Structured Grids](6051-marching-cubes-on-structured-grids.md) by introducing a 2-bit sign encoding and a post-processing vertex snapping step [S1, unofficial]. Unlike [Bitwise Marching Cubes](7374-bitwise-marching-cubes.md) which prioritizes branchless logic, or [Flying Edges](7520-flying-edges.md) which optimizes for streaming memory access, PEM-MC prioritizes vertex position accuracy for structured grids [G].

## How it works
### Inputs and Outputs
**Inputs:**
- Scalar field $f$ defined on a structured grid of dimensions $(dim_x+1) \times (dim_y+1) \times (dim_z+1)$ [S1, unofficial].
- Isosurface threshold value $I$ (ISOLEVEL) [S1, unofficial].
- Boolean flag `pem` enabling Position Error Minimization [S1, unofficial].
- Snapping threshold `snap_threshold` (float) [S1, unofficial].
- Sampler function `sampler_fn(fx, fy, fz, w, osn)` returning the scalar value at world coordinates [S1, unofficial].

**Outputs:**
- A set of triangles representing the isosurface [G].
- Vertex positions adjusted via snapping logic [S1, unofficial].

### Data Structures
The method uses the following structures, consistent with the parent article but extended for PEM mode:

1.  **`grid_signs`**: Array of `uint16_t`, size $(dim_x/2) \times (dim_y/2) \times (dim_z/2)$ [S1, unofficial].
    -   Stores signs for 8 grid vertices per 16-bit word [S1, unofficial].
    -   **Standard Mode**: 1 bit per vertex [S1, unofficial].
    -   **PEM Mode**: 2 bits per vertex. The bits are packed based on the vertex parity $(x\&1, y\&1, z\&1)$ within a $2\times2\times2$ block [S1, unofficial].
    -   Sign encoding in PEM mode:
        -   `0`: $s < I$
        -   `1`: $s = I$
        -   `2`: $s > I$

2.  **`grid_verts`**: Array of struct `UMC_Isovertex`, size $(dim_x+1) \times (dim_y+1) \times (dim_z+1)$ [S1, unofficial].
    -   Fields:
        -   `value` (float): The scalar value at the vertex [S1, unofficial].
        -   `index` (int32): Output index. Set to `-2` to mark a vertex for snapping [S1, unofficial].
        -   `position` (vec3): The 3D coordinates of the vertex [S1, unofficial].

3.  **`edges`**: Array of struct `UMC_Edge`, size $(dim_x+1) \times (dim_y+1) \times (dim_z+1) \times 3$ [S1, unofficial].
    -   Fields:
        -   `grid_v0` (int32): Index into `grid_verts` for the start vertex [S1, unofficial].
        -   `grid_v1` (int32): Index into `grid_verts` for the end vertex [S1, unofficial].

### Algorithm Steps

1.  **Grid Labeling**:
    Iterate over all grid vertices $(x, y, z)$ from $0$ to $dim$ [S1, unofficial].
    -   Compute floating-point coordinates $(fx, fy, fz)$ [S1, unofficial].
    -   Sample the scalar field: $s = \text{sampler\_fn}(fx, fy, fz, w, osn)$ [S1, unofficial].
    -   If `pem` is false:
        -   Set a 1-bit sign in `grid_signs` [S1, unofficial].
    -   If `pem` is true:
        -   Set a 2-bit sign in `grid_signs`:
            -   $0$ if $s < I$
            -   $1$ if $s = I$
            -   $2$ if $s > I$

2.  **Edge Labeling and Crossing Detection**:
    Iterate over all grid vertices and check the three outgoing edges (X, Y, Z directions) [S1, unofficial].
    -   For each edge, call `_UMC_Chunk_calc_edge_crossing` [S1, unofficial].
    -   If a crossing is detected OR the PEM bit 4 is set (indicating a vertex is exactly on the isosurface):
        -   Compute the isovector intersection via `_UMC_Chunk_calc_edge_isov` [S1, unofficial].
        -   Store the intersection details in the `edges` array [S1, unofficial].

3.  **Vertex Snapping Marking (PEM Only)**:
    After computing edge crossings, if `pem` is true:
    -   If `result_mask & 1` is true, call `_UMC_Chunk_set_isov` for the start vertex [S1, unofficial].
    -   If `result_mask & 2` is true, call `_UMC_Chunk_set_isov` for the end vertex [S1, unofficial].
    -   This marks vertices for snapping by setting their `index` in `grid_verts` to `-2` [S1, unofficial].

4.  **Vertex Snapping**:
    Iterate over output indices in `grid_verts` [S1, unofficial].
    -   Extract grid coordinates $(x, y, z)$ [S1, unofficial].
    -   **Boundary Check**: If $x=0$, $x=dim_x$, $y=0$, $y=dim_y$, $z=0$, or $z=dim_z$, skip snapping for this vertex [S1, unofficial].
    -   If not on the boundary and `index` is `-2`:
        -   Adjust the vertex position towards the exact isosurface intersection point [S1, unofficial].
        -   The adjustment is controlled by `snap_threshold` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def position_error_minimized_marching_cubes(grid_dims, isovalue, snap_threshold, sampler_fn):
    """
    grid_dims: tuple (dim_x, dim_y, dim_z)
    isovalue: float I
    snap_threshold: float
    sampler_fn: function returning scalar at (fx, fy, fz, w, osn)
    """
    dim_x, dim_y, dim_z = grid_dims
    pem = True  # Enable PEM mode per evidence

    # Data structures
    # grid_signs: (dim_x/2) * (dim_y/2) * (dim_z/2) uint16_t array
    signs_w = dim_x // 2
    signs_h = dim_y // 2
    signs_d = dim_z // 2
    grid_signs = [0] * (signs_w * signs_h * signs_d)

    # grid_verts: (dim_x+1) * (dim_y+1) * (dim_z+1)
    grid_verts = []
    for z in range(dim_z + 1):
        for y in range(dim_y + 1):
            for x in range(dim_x + 1):
                grid_verts.append({
                    'value': 0.0,
                    'index': -1,
                    'position': (float(x), float(y), float(z))
                })

    # Step 1: Grid Labeling
    for z in range(dim_z + 1):
        for y in range(dim_y + 1):
            for x in range(dim_x + 1):
                idx = z * (dim_x + 1) * (dim_y + 1) + y * (dim_x + 1) + x
                fx, fy, fz = grid_verts[idx]['position']
                s = sampler_fn(fx, fy, fz, 0, 0)  # w, osn per evidence signature
                grid_verts[idx]['value'] = s

                # Pack sign into grid_signs
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
                grid_signs[block_z * signs_w * signs_h + block_y * signs_w + block_x] |= (sign_val << (2 * bit_offset))

    # Step 2 & 3: Edge Labeling, Crossing Detection, and Snapping Marking
    # (Topology lookup and edge intersection calculation follow the parent algorithm [G])
    # Simulating the PEM-specific marking logic from evidence:
    for z in range(dim_z + 1):
        for y in range(dim_y + 1):
            for x in range(dim_x + 1):
                idx = z * (dim_x + 1) * (dim_y + 1) + y * (dim_x + 1) + x
                s = grid_verts[idx]['value']
                # Evidence: if s == ISOLEVEL, mark for snapping via result_mask logic
                if s == isovalue:
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
                    # [G] Snapping logic: project vertex towards isosurface intersection
                    # using snap_threshold to limit displacement.
                    # In the reference implementation, _UMC_Chunk_snap_verts handles this.
                    # The source does not provide the exact projection formula.
                    pos = grid_verts[idx]['position']
                    # grid_verts[idx]['position'] = snap_to_isosurface(pos, isovalue, snap_threshold)
                    grid_verts[idx]['index'] = idx  # Reset to valid output index

    # [G] Generate triangles using standard Marching Cubes lookup on grid_signs and edges array
    return grid_verts
```

### Parameters
-   `pem`: Boolean. Enables 2-bit sign encoding and vertex snapping [S1, unofficial].
-   `ISOLEVEL`: Float. The threshold value for scalar field comparison [S1, unofficial].
-   `snap_threshold`: Float. Controls the maximum distance for vertex adjustment during snapping [S1, unofficial].

### Edge Cases and Pitfalls
-   **Boundary Vertices**: Vertices on the boundary of the grid ($x=0, x=dim_x, y=0, y=dim_y, z=0, z=dim_z$) are skipped during snapping to prevent out-of-bounds access or incorrect geometric adjustments [S1, unofficial].
-   **Sign Encoding**: In PEM mode, the 2-bit encoding allows for three states, which is crucial for identifying vertices exactly on the isosurface [S1, unofficial].

## Operating characteristics
-   **Time Complexity**: $O(dim^3)$ for labeling and edge processing [S1, unofficial].
-   **Memory Complexity**: $O(dim^3)$ for grid vertices, edges, and signs [S1, unofficial].
-   **Strengths**: Provides higher geometric accuracy by snapping vertices to the isosurface, reducing position error compared to standard linear interpolation [S1, unofficial].
-   **Limitations**: Slightly higher memory usage due to 2-bit sign encoding and additional snapping logic. Boundary conditions require special handling [S1, unofficial].

## Background
Position Error Minimized Marching Cubes is part of the "Unified Marching Cubes" (UMC) approach, aiming to combine the efficiency of bitwise operations with the accuracy of position error minimization [S1, unofficial]. It originates from the Lin20/PushingVoxelsForward repository [S1, unofficial].

## Sources
[S1] Lin20/PushingVoxelsForward, https://github.com/Lin20/PushingVoxelsForward, fetched 2026-10-27 (unofficial)
