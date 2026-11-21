# ClearView Unified Buffer Construction

Also called: ClearView Buffer Construction

## Overview
ClearView Unified Buffer Construction is a preprocessing technique that converts a hexahedral mesh (positions, connectivity, and scalar values) into four GPU-friendly texture buffers—vertices, edges, faces, and cellScalar—optimized for the ClearView volume rendering pipeline [S1, unofficial]. It differs from standard volume texture creation by explicitly encoding topological connectivity (faces and edges) alongside geometric data to support fragment-based rendering passes, whereas siblings like [Volume Upload via Staging Buffer](15388-volume-upload-via-staging-buffer.md) focus on raw data transfer and [Device Volume Snapshot](2246-device-volume-snapshot.md) focuses on immutable state management.

## How it works
**Inputs:**
- Hexahedral mesh with $N_c$ cells.
- Vertex positions: array of size $8 N_c$ (assuming non-shared vertices for simplicity in this layout) or shared vertex list.
- Cell scalar values: array of size $N_c$ (or per-vertex, aggregated here).
- Scalar range for normalization.

**Outputs:**
Four 2D textures bound to specific formats:
1.  `edgesTex`: `RG32F` (Float32Array).
2.  `facesTex`: `RG32UI` (Uint32Array).
3.  `cellScalarTex`: `R32F` (Float32Array).
4.  `vertexTex`: (Implied by context, though specific layout details for vertices are not fully detailed in the provided notes, it is listed as a deleted resource [S1, unofficial]).

**Data Structures and Layouts:**

1.  **Edges Texture (`edgesTex`)**
    -   Format: `RG32F` (2 floats per texel).
    -   Dimensions: Width $W_e$, Height $H_e$ determined by `texDims(edgeCount)`.
    -   Total Texels: $N_e = N_c \times 12$ (12 edges per hexahedron).
    -   Linear Index: $L = c \times 12 + e$, where $c \in [0, N_c)$ is cell index, $e \in [0, 11)$ is edge index within cell.
    -   Texel Coordinates:
        -   $col = L \pmod{W_e}$
        -   $row = \lfloor L / W_e \rfloor$
        -   Texel Index in array: $ti = (row \times W_e + col) \times 2$
    -   Values:
        -   `edgesTex[ti]` = $\max(scalars[p_{ia}], scalars[p_{ib}])$ (Attribute: max scalar of edge endpoints).
        -   `edgesTex[ti+1]` = $1 - normScalar$ (Attribute: LOD factor).

2.  **Faces Texture (`facesTex`)**
    -   Format: `RG32UI` (4 unsigned ints per texel).
    -   Dimensions: Width $W_f = 2$, Height $H_f = N_f = N_c \times 6$ (6 faces per hexahedron).
    -   Total Texels: $2 \times N_f$.
    -   Face ID: $faceId = c \times 6 + f$, where $f \in [0, 5)$ is face index within cell.
    -   Two texels per face:
        -   Texel 0 (Vertex Indices): Stores 4 vertex indices.
        -   Texel 1 (Edge Indices): Stores 4 edge indices.
    -   Linear Texel Index:
        -   $t0linear = faceId \times 2 + 0$
        -   $t1linear = faceId \times 2 + 1$
    -   Texel Coordinates:
        -   $col = tlinear \pmod{W_f}$
        -   $row = \lfloor tlinear / W_f \rfloor$
        -   Texel Index in array: $ti = (row \times W_f + col) \times 4$
    -   Values:
        -   `facesTex[ti+0..3]` = $c \times 8 + face.corners[0..3]$ (Vertex indices relative to cell start).
        -   `facesTex[ti+4..7]` = $c \times 12 + face.edges[0..3]$ (Edge indices relative to cell start).

3.  **Cell Scalar Texture (`cellScalarTex`)**
    -   Format: `R32F` (1 float per texel).
    -   Dimensions: Width $W_s$, Height $H_s$ determined by `texDims(faceCount)`. Note: The source maps this to face count, implying one scalar value per face for the purpose of the rendering pass, or potentially one per cell mapped to face slots. The source explicitly states "1 texel per face" and uses `faceId`.
    -   Total Texels: $N_f = N_c \times 6$.
    -   Linear Index: $faceId = c \times 6 + f$.
    -   Texel Coordinates:
        -   $col = faceId \pmod{W_s}$
        -   $row = \lfloor faceId / W_s \rfloor$
    -   Value:
        -   `cellScalarTex[row * W_s + col]` = $cellScalarAvg$.

**Algorithm Steps:**

1.  **Compute Scalar Range:**
    $$s_{min} = \min(mesh.scalars)$$
    $$s_{max} = \max(mesh.scalars)$$
    $$s_{range} = \max(s_{max} - s_{min}, 10^{-9})$$

2.  **Compute Edge LOD:**
    For each cell $c$:
    $$cellScalarAvg = \frac{\sum_{i=0}^{7} scalars[corner_i]}{8}$$
    $$normScalar = \frac{cellScalarAvg - s_{min}}{s_{range}}$$
    $$edgeLod = 1 - normScalar$$

3.  **Construct Edges Texture:**
    For each cell $c$ and each of its 12 edges $e$:
    -   Identify endpoints $p_{ia}, p_{ib}$.
    -   Calculate linear index $L = c \times 12 + e$.
    -   Calculate row/col.
    -   Store $\max(scalars[p_{ia}], scalars[p_{ib}])$ in R channel.
    -   Store $edgeLod$ in G channel.

4.  **Construct Faces Texture:**
    For each cell $c$ and each of its 6 faces $f$:
    -   Calculate $faceId = c \times 6 + f$.
    -   Get face geometry: `face.corners` (4 vertex indices relative to cell), `face.edges` (4 edge indices relative to cell).
    -   Calculate texel indices $t0linear, t1linear$.
    -   Store $c \times 8 + face.corners[0..3]$ in Texel 0 (RGBA).
    -   Store $c \times 12 + face.edges[0..3]$ in Texel 1 (RGBA).

5.  **Construct Cell Scalar Texture:**
    For each cell $c$ and each of its 6 faces $f$:
    -   Calculate $faceId = c \times 6 + f$.
    -   Calculate row/col based on $W_s, H_s$.
    -   Store $cellScalarAvg$ at the calculated position.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_clearview_buffers(mesh):
    """
    mesh: object with .positions (8*N_c), .scalars (N_c or 8*N_c), .connectivity
    Returns: buffers dict with 'edges', 'faces', 'cellScalar'
    """
    N_c = mesh.num_cells
    scalars = mesh.scalars # Assume N_c scalars, averaged per cell for this implementation
    
    # 1. Scalar Range
    s_min = min(scalars)
    s_max = max(scalars)
    s_range = max(s_max - s_min, 1e-9)
    
    # 2. Precompute Cell Scalar Avg and LOD
    cell_scalars_avg = []
    edge_lods = []
    for c in range(N_c):
        # Assuming scalars are per-cell or averaged from corners
        # If per-vertex, sum 8 corners. Here assuming per-cell input for simplicity
        # based on "sum(scalars[8 corners])" note, but input is "mesh.scalars".
        # Let's assume mesh.scalars is per-cell for the avg calculation source.
        avg = scalars[c] 
        cell_scalars_avg.append(avg)
        
        norm_scalar = (avg - s_min) / s_range
        edge_lods.append(1.0 - norm_scalar)

    # 3. Edges Texture
    # 12 edges per cell
    edge_count = N_c * 12
    edges_data = [0.0] * (edge_count * 2) # RG32F -> 2 floats per texel
    
    # Helper to get texel index
    def get_edge_texel_index(linear_idx, width):
        row = linear_idx // width
        col = linear_idx % width
        return (row * width + col) * 2

    # Determine width/height for edges (power of 2 or optimal)
    # Not specified, using generic texDims logic placeholder
    width_e, height_e = tex_dims(edge_count) 
    
    for c in range(N_c):
        lod = edge_lods[c]
        for e in range(12):
            linear_idx = c * 12 + e
            ti = get_edge_texel_index(linear_idx, width_e)
            
            # Get endpoint scalars
            # Assuming mesh provides edge-to-vertex mapping
            p_a, p_b = mesh.edge_vertices(c, e)
            scalar_a = scalars[p_a] if len(scalars) > 8*N_c else scalars[c] # Fallback logic
            scalar_b = scalars[p_b] if len(scalars) > 8*N_c else scalars[c]
            
            # If scalars are per-cell, max is just the cell scalar
            # If per-vertex, take max of endpoints
            if len(scalars) > 8*N_c:
                val = max(scalar_a, scalar_b)
            else:
                val = scalars[c]
                
            edges_data[ti] = val
            edges_data[ti+1] = lod

    # 4. Faces Texture
    # 6 faces per cell, 2 texels per face
    face_count = N_c * 6
    faces_data = [0] * (face_count * 2 * 4) # RG32UI -> 4 uints per texel, 2 texels per face
    
    width_f = 2
    height_f = face_count
    
    for c in range(N_c):
        for f in range(6):
            face_id = c * 6 + f
            t0_linear = face_id * 2 + 0
            t1_linear = face_id * 2 + 1
            
            ti0 = (t0_linear // width_f + (t0_linear % width_f) * height_f) * 4 # Note: Source uses row/col logic differently for width=2?
            # Source: col = tlinear % width, row = floor(tlinear / width). ti = (row * width + col) * 4
            # With width=2:
            # t0_linear: col=0, row=face_id. ti0 = (face_id * 2 + 0) * 4
            # t1_linear: col=1, row=face_id. ti1 = (face_id * 2 + 1) * 4
            
            ti0 = (face_id * 2 + 0) * 4
            ti1 = (face_id * 2 + 1) * 4
            
            # Get face corners and edges
            corners = mesh.face_corners(c, f) # 4 vertex indices relative to cell? Or absolute?
            edges = mesh.face_edges(c, f)     # 4 edge indices relative to cell?
            
            # Source: c*8 + face.corners[0..3]
            # Assuming corners are 0-7 relative to cell
            faces_data[ti0+0] = c * 8 + corners[0]
            faces_data[ti0+1] = c * 8 + corners[1]
            faces_data[ti0+2] = c * 8 + corners[2]
            faces_data[ti0+3] = c * 8 + corners[3]
            
            # Source: c*12 + face.edges[0..3]
            # Assuming edges are 0-11 relative to cell
            faces_data[ti1+0] = c * 12 + edges[0]
            faces_data[ti1+1] = c * 12 + edges[1]
            faces_data[ti1+2] = c * 12 + edges[2]
            faces_data[ti1+3] = c * 12 + edges[3]

    # 5. Cell Scalar Texture
    # 1 texel per face
    scalar_tex_count = face_count
    scalar_tex_data = [0.0] * scalar_tex_count
    
    width_s, height_s = tex_dims(scalar_tex_count)
    
    for c in range(N_c):
        for f in range(6):
            face_id = c * 6 + f
            row = face_id // width_s
            col = face_id % width_s
            scalar_tex_data[row * width_s + col] = cell_scalars_avg[c]

    return {
        'edges': edges_data,
        'faces': faces_data,
        'cellScalar': scalar_tex_data
    }
```

**Parameters:**
-   `s_range` minimum: $10^{-9}$ to prevent division by zero [S1, unofficial].
-   `focusMask` exponent: 4.0 (used in focus controls, not buffer construction directly, but related to the pipeline) [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Zero Scalar Range:** If $s_{max} == s_{min}$, $s_{range}$ is clamped to $10^{-9}$ [S1, unofficial].
-   **Vertex/Edge Indexing:** The layout assumes relative indexing within the cell ($c \times 8$ for vertices, $c \times 12$ for edges). If the mesh uses shared vertices/edges, this layout requires expansion or a different indexing scheme [S1, unofficial].
-   **Texture Dimensions:** `texDims` function is not defined in the source. Implementers must choose dimensions (e.g., power-of-2) that fit the data [S1, unofficial].

## Subtopics

### Device Volume Snapshot
Creates immutable snapshots of volume data for thread-safe sharing. Use when data must not change during rendering. [Device Volume Snapshot](2246-device-volume-snapshot.md)

### GPU Data Layout
Defines general principles for mapping host data to GPU memory. Use for understanding the underlying memory hierarchy. [GPU Data Layout](2469-gpu-data-layout.md)

### Volume Upload via Staging Buffer
Handles data transfer from host to device. Use when direct mapping is unsupported or staging is required. [Volume Upload via Staging Buffer](15388-volume-upload-via-staging-buffer.md)

## Operating characteristics
-   **Time Complexity:** $O(N_c)$, where $N_c$ is the number of cells. Each cell processes a constant number of edges (12) and faces (6).
-   **Memory Complexity:** $O(N_c)$. Specifically, $24 N_c$ floats for edges, $32 N_c$ unsigned ints for faces, and $6 N_c$ floats for cell scalars.
-   **Performance:** Optimized for coalesced memory access in the ClearView rendering pipeline by packing connectivity into 2D textures.
-   **Strengths:** Explicitly encodes topology for fragment-based rendering; avoids complex lookups during shading.
-   **Limitations:** Requires hexahedral meshes; assumes non-shared or easily expandable connectivity; texture dimensions must be carefully chosen.
-   **When to use:** ClearView volume rendering pipeline with hexahedral data.
-   **When not to use:** Tetrahedral meshes, non-hexahedral volumes, or when shared topology is critical and not expanded.

## Background
The ClearView Unified Buffer Construction method originates from the `geresim-spike` project, an unofficial implementation of a volume rendering technique focusing on "focus+context" visualization [S1, unofficial]. It was developed to support a specific WebGL2-based rendering pipeline that uses fragment shaders for rendering hexahedral meshes. The technique is part of a broader effort to optimize volume rendering on consumer GPUs by leveraging texture-based data structures for topology and scalar data.

## Sources
[S1] Amory0709/geresim-spike, https://github.com/Amory0709/geresim-spike, fetched 2026-10-27 (unofficial)
