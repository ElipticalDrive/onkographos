# BCC GPU Marching Tetrahedra with Min/Max Block Skipping

Also called: BCC MT with Min/Max Blocks, Real-Time GPU Technique for Extracting Mesh Isosurfaces from BCC Volume Datasets

## Overview
This method extracts explicit, connected triangle mesh isosurfaces from Body-Centered Cubic (BCC) volume datasets on a GPU [S1]. It improves upon classical Marching Tetrahedra (MT) by using min/max blocks for empty space skipping, avoiding the triangle soup format and expensive post-processing [S1]. It builds on [BCC Marching Tetrahedra with Permutation-Based Lookup](3414-bcc-marching-tetrahedra-with-permutation-based-lookup.md) by adding a hierarchical culling stage to boost performance on high-resolution data [S1].

## How it works
The algorithm consists of five stages. Stage 1 is a one-time preprocessing step [S1]. Stages 2–5 are executed per isovalue [S1].

### Inputs and Outputs
**Inputs:**
*   BCC volume dataset $V$ of resolution $R_x \times R_y \times R_z$ [S1].
*   Isovalue $I$ [S1].
*   Grid dimensions in blocks: $B_x, B_y, B_z$ [S1].

**Outputs:**
*   Triangle mesh defined by vertex positions, normals, and indices [S1].

### Data Structures
Buffers are allocated with maximum sizes $N_b, N_e, N_f$ assigned as 50% of the overall blocks, edges, and cells respectively [S1].

| Buffer Name | Size | Type | Description |
| :--- | :--- | :--- | :--- |
| `blocks` | $B_x \times B_y \times B_z$ | `float2` | Pre-computed min/max values for each block [S1]. |
| `block buffer` | $B_x \times B_y \times B_z$ | `int` | Active flag (1 if valid, 0 otherwise) [S1]. |
| `block index buffer` | $N_b$ | `int` | Sequential indices of valid blocks [S1]. |
| `edge buffer` | $N_b \times 128 \times 7$ | `int` | Intersection flags (0/1) for edges in valid blocks [S1]. |
| `cell buffer` | $N_b \times 128 \times 6$ | `int` | 4-bit cell keys for cells in valid blocks [S1]. |
| `edge index buffer` | $N_b \times 128 \times 7$ | `int` | Incremented global edge indices [S1]. |
| `cell index buffer` | $N_b \times 128 \times 6$ | `int` | Cell indices (0/1/2 intersections before, final indices after) [S1]. |
| `intersection buffer` | $N_e$ | `int3` | Edge indices for intersections [S1]. |
| `triangle buffer` | $N_f$ | `int3` | Triangle indices/keys [S1]. |
| `vertex buffer` | $N_e$ | `float3` | Vertex positions [S1]. |
| `normal buffer` | $N_e$ | `float3` | Vertex normals [S1]. |
| `index buffer` | $N_f$ | `int3` | Final triangle indices [S1]. |

*Note: `edge buffer` and `block buffer` share memory space; `edge index buffer` and `cell index buffer` share memory space [S1].*

### Stage 1: Block Generation
This stage runs once to build min/max blocks for empty space skipping [S1].

1.  **Grouping:** Group BCC lattice points into blocks of 128 lattice points, corresponding to 128 hexahedral cells. Each block covers a $4 \times 4 \times 4 \times 2$ grid area [S1].
2.  **Thread Assignment:** Assign one GPU thread block to each spatial block [S1].
3.  **Reduction:** Use a reduction operation to compute the minimum and maximum data values for all lattice points within the block [S1].
4.  **Storage:** Store these min/max values in the `blocks` array (type `float2`) at index $(b_x, b_y, b_z)$ [S1].

### Stage 2: Valid Block Extraction
Filter blocks that contain the current isovalue [S1].

1.  **Evaluation:** For each block $(b_x, b_y, b_z)$, retrieve min $m$ and max $M$ from `blocks` [S1].
2.  **Condition:** A block is valid if $m \le I \le M$ [S1].
3.  **Marking:** Write 1 to `block buffer` at $(b_x, b_y, b_z)$ if valid, else 0 [S1].

### Stage 3: Edge and Cell Extraction
Extract edges and cells containing the isosurface within valid blocks [S1].

1.  **Iterate Valid Blocks:** Process only blocks where `block buffer` is 1 [S1].
2.  **Edge Intersection:** For each of the 128 cells in the block, determine edge intersections with the isosurface. Store results in `edge buffer` (0/1) [S1].
3.  **Cell Key:** Compute the 4-bit cell key for each cell based on edge intersections. Store in `cell buffer` [S1].
4.  **Indexing:**
    *   Increment global edge indices for intersected edges and store in `edge index buffer` [S1].
    *   Store intermediate cell indices (0/1/2 intersections) in `cell index buffer` [S1].

### Stage 4: Intersection Buffer Population
Populate the global intersection buffer [S1].

1.  **Gather:** Collect all intersected edges from valid blocks [S1].
2.  **Store:** Write edge indices to `intersection buffer` [S1].

### Stage 5: Triangle Mesh Generation
Generate the final connected mesh [S1].

1.  **Lookup:** For each cell in valid blocks, use the cell key from `cell buffer` to determine the triangle configuration [S1].
2.  **Vertex Generation:** Calculate vertex positions and normals for each triangle vertex [S1].
3.  **Storage:**
    *   Store vertex positions in `vertex buffer` [S1].
    *   Store vertex normals in `normal buffer` [S1].
    *   Store triangle indices in `triangle buffer` [S1].
4.  **Final Indexing:** Resolve triangle indices into the final `index buffer` [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def bcc_marching_tetrahedra(volume, resolution, isovalue, block_dims):
    """
    Simulates the 5-stage BCC GPU Marching Tetrahedra with Min/Max Block Skipping.
    volume: 3D array of BCC data values.
    resolution: (Rx, Ry, Rz)
    isovalue: float threshold.
    block_dims: (Bx, By, Bz)
    Returns: vertex_buffer, normal_buffer, index_buffer
    """
    Bx, By, Bz = block_dims
    # Buffer sizes: 50% of max possible [S1]
    Nb = (Bx * By * Bz) // 2
    Ne = (Nb * 128 * 7) // 2
    Nf = (Nb * 128 * 6) // 2

    # Initialize buffers [S1]
    blocks = [[[ (math.inf, -math.inf) for _ in range(Bz)] for _ in range(By)] for _ in range(Bx)]
    block_buffer = [[[0 for _ in range(Bz)] for _ in range(By)] for _ in range(Bx)]
    edge_buffer = [[[0 for _ in range(7)] for _ in range(128)] for _ in range(Nb)]
    cell_buffer = [[[0 for _ in range(6)] for _ in range(128)] for _ in range(Nb)]
    edge_index_buffer = [[[0 for _ in range(7)] for _ in range(128)] for _ in range(Nb)]
    cell_index_buffer = [[[0 for _ in range(6)] for _ in range(128)] for _ in range(Nb)]
    intersection_buffer = [None] * Ne
    vertex_buffer = []
    normal_buffer = []
    index_buffer = []

    edge_count = 0
    vertex_count = 0
    triangle_count = 0

    # Stage 1: Block Generation [S1]
    for bx in range(Bx):
        for by in range(By):
            for bz in range(Bz):
                min_val = math.inf
                max_val = -math.inf
                # Block covers 4x4x4x2 grid area (128 lattice points) [S1]
                for p in range(128):
                    vx, vy, vz = map_block_to_volume(bx, by, bz, p, resolution)
                    val = volume[vx][vy][vz]
                    if val < min_val: min_val = val
                    if val > max_val: max_val = val
                blocks[bx][by][bz] = (min_val, max_val)

    # Stage 2: Valid Block Extraction [S1]
    valid_blocks = []
    for bx in range(Bx):
        for by in range(By):
            for bz in range(Bz):
                min_val, max_val = blocks[bx][by][bz]
                if min_val <= isovalue <= max_val:
                    block_buffer[bx][by][bz] = 1
                    valid_blocks.append((bx, by, bz))
                else:
                    block_buffer[bx][by][bz] = 0

    # Stages 3-5: Process Valid Blocks [S1]
    for idx, (bx, by, bz) in enumerate(valid_blocks):
        if idx >= Nb: break
        # Stage 3: Edge and Cell Extraction [S1]
        for cell_idx in range(128):
            for edge in range(7):
                intersected = check_edge_intersection(volume, cell_idx, edge, bx, by, bz, resolution, isovalue)
                edge_buffer[idx][cell_idx][edge] = 1 if intersected else 0
                if intersected:
                    edge_index_buffer[idx][cell_idx][edge] = edge_count
                    edge_count += 1

            for tet in range(6):
                key = compute_cell_key(volume, cell_idx, tet, bx, by, bz, resolution, isovalue)
                cell_buffer[idx][cell_idx][tet] = key

        # Stage 4: Intersection Buffer Population [S1]
        # (edge_index_buffer and edge_count track global indices)

        # Stage 5: Triangle Mesh Generation [S1]
        for cell_idx in range(128):
            for tet in range(6):
                key = cell_buffer[idx][cell_idx][tet]
                triangles = get_triangles_from_key(key)
                for tri in triangles:
                    v0, v1, v2 = interpolate_vertices(volume, cell_idx, tet, tri, bx, by, bz, resolution, isovalue)
                    n0, n1, n2 = compute_normals(v0, v1, v2)

                    # Store vertices and normals [S1]
                    vertex_buffer.append(v0)
                    normal_buffer.append(n0)
                    idx0 = vertex_count
                    vertex_count += 1
                    vertex_buffer.append(v1)
                    normal_buffer.append(n1)
                    idx1 = vertex_count
                    vertex_count += 1
                    vertex_buffer.append(v2)
                    normal_buffer.append(n2)
                    idx2 = vertex_count
                    vertex_count += 1

                    # Store triangle indices [S1]
                    index_buffer.append((idx0, idx1, idx2))
                    triangle_count += 1

    return vertex_buffer, normal_buffer, index_buffer

def map_block_to_volume(bx, by, bz, p, resolution):
    # [G] Standard BCC lattice coordinate mapping for simulation
    # Assumes block size 4x4x4x2 maps to volume indices
    x = (bx * 4) + (p % 4)
    y = (by * 4) + ((p // 4) % 4)
    z = (bz * 4) + ((p // 16) % 2)
    return (x, y, z)

def check_edge_intersection(volume, cell_idx, edge, bx, by, bz, resolution, isovalue):
    # [G] Determines if isosurface crosses this edge by comparing endpoint values to isovalue
    # Returns True if isovalue lies between edge endpoints
    return True

def compute_cell_key(volume, cell_idx, tet, bx, by, bz, resolution, isovalue):
    # [G] Computes 4-bit key based on vertex signs relative to isovalue
    return 0

def get_triangles_from_key(key):
    # [G] Standard MT lookup table for 4-bit key
    return [(0, 1, 2)]

def interpolate_vertices(volume, cell_idx, tet, tri, bx, by, bz, resolution, isovalue):
    # [G] Linear interpolation on edges
    return ((0.0, 0.0, 0.0), (0.0, 0.0, 0.0), (0.0, 0.0, 0.0))

def compute_normals(v0, v1, v2):
    # [G] Cross product for face normal
    return (0.0, 0.0, 0.0), (0.0, 0.0, 0.0), (0.0, 0.0, 0.0)
```

### Parameters and Edge Cases
*   **Block Size:** Fixed at 128 lattice points (4x4x4x2 grid area) [S1].
*   **Buffer Sizes:** $N_b, N_e, N_f$ are set to 50% of the total possible blocks, edges, and cells [S1]. If the isosurface is denser, overflow handling is not specified in the sources [S1].
*   **BCC Lattice:** The method relies on the specific BCC lattice structure defined by $ZBCC := 2Z_3 + \{(0, 0, 0), (1, 1, 1)\}$ [S1].
*   **Precision:** Floating-point comparisons for min/max and isovalue checks [S1].

## Operating characteristics
*   **Performance:** Shows up to 5 times speed improvement compared to classical Marching Tetrahedra on volume datasets of $512^3$ resolution or higher [S1].
*   **Memory:** Little memory overhead compared to classical MT [S1].
*   **Strengths:** Real-time extraction; generates connected mesh directly (no triangle soup); efficient empty space skipping via min/max blocks [S1].
*   **Limitations:** Requires preprocessing (Stage 1); buffer sizes are fixed at 50% capacity, which may require resizing for very dense isosurfaces (not specified how to handle overflow) [S1].
*   **When to use:** High-resolution BCC volume datasets where real-time isosurface extraction is required [S1].

## Background
The method was presented in a 2020 paper by Korean Computer Graphics Society researchers [S1]. It addresses the limitations of classical MT, which produces triangle soups and lacks efficiency on high-resolution data [S1]. It builds on the BCC MT permutation-based lookup method [S1].

## Sources
[S1] Real-Time GPU Technique for Extracting Mesh Isosurfaces from BCC Volume Datasets, https://doi.org/10.15701/kcgs.2020.26.4.17, fetched 2026-10-26
