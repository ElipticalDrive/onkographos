# Mesh Finalization

Also called: VTK Mesh Finalization

## Overview
Mesh Finalization is the post-assembly stage that converts raw, potentially fragmented mesh data into a validated, render-ready `RenderMesh` structure. It solves the problem of ensuring data consistency for downstream rendering and visualization by validating indices, extrapolating cell-centered data to vertices, and computing geometric attributes like normals and bounds. It is distinct from [Chunk Mesh Assembly](4100-chunk-mesh-assembly.md), which focuses on buffer allocation and layer iteration, by handling the topological and attribute-level integrity required for unstructured or CFD-style meshes.

## How it works
**Inputs:** A raw mesh structure containing `vertices` (float array), `indices` (uint array), `cellConnectivity`, `cellTypes`, and optional scalar/vector fields.
**Outputs:** A finalized `RenderMesh` object with populated `normals`, `bounds`, `cellCenters`, and validated attribute buffers.

**Data Structures:**
*   `RenderMesh`:
    *   `vertices`: `std::vector<float>`, size $N_{verts} \times 3$.
    *   `indices`: `std::vector<uint32_t>`, size $N_{tris} \times 3$.
    *   `normals`: `std::vector<float>`, size $N_{verts} \times 3$.
    *   `cellCenters`: `std::vector<float>`, size $N_{cells} \times 3$.
    *   `pointScalars`: `std::map<std::string, std::vector<float>>`.
    *   `cellVectors`: `std::map<std::string, std::vector<float>>`.
    *   `pointVectorOffset`: `std::vector<std::size_t>`, offsets into `pointVectorsData`.
    *   `cellVectorOffset`: `std::vector<std::size_t>`, offsets into `cellVectorsData`.

**Steps:**

1.  **Validation:**
    *   If `vertices` is empty, log error and return [S1, unofficial].
    *   If `indices` is empty and the mesh is not `POLYDATA` and not `renderAsPoints`, log error and return [S1, unofficial].
    *   Validate indices: For each index $i$ in `indices`, if $i \ge N_{verts}$, log error, clear `indices`, and return to prevent GPU out-of-range fetches [S1, unofficial].

2.  **Cell Data Extrapolation:**
    *   Call `mesh_utils::extrapolateCellDataToPoints` to transfer cell-centered values to vertex locations [S1, unofficial]. This ensures point-based attributes (like colors or temperatures) are available for vertex shaders.

3.  **Vector Buffer Flattening:**
    *   **Point Vectors:** Flatten all point vector fields into a contiguous buffer `mesh.pointVectorsData`. Populate `mesh.pointVectorOffset` with per-field byte offsets. Set `mesh.pointVectorCount` to the number of vertices per point (typically 3) [S1, unofficial].
    *   **Cell Vectors:** Flatten all cell vector fields into `mesh.cellVectorsData`. Populate `mesh.cellVectorOffset`. Set `mesh.cellVectorCount` to the number of cells [S1, unofficial]. Record available cell vector names for UI selection [S1, unofficial].
    *   **Default Selection:** If no `cellVectorName` is explicitly set, default to the first available cell vector name [S1, unofficial].

4.  **Cell Center Computation:**
    *   Compute cell centers by averaging the vertex positions of each cell's corners [S1, unofficial].
    *   For a cell with vertices $V_1, V_2, ..., V_k$, the center $C$ is:
        $$ C = \frac{1}{k} \sum_{j=1}^{k} V_j $$
    *   Store results in `mesh.cellCenters` [S1, unofficial].

5.  **Active Scalar Selection:**
    *   If `scalarName` is empty or not found in `pointScalars`, default to the first available point scalar [S1, unofficial].
    *   Validate size: If the selected scalar's size does not match $N_{verts}$, clear the scalars and log an error [S1, unofficial].

6.  **Attribute Population:**
    *   Populate lists of available scalar and vector names, sorted alphabetically, for UI display [S1, unofficial].
    *   If no `vectorName` is set, default to the first available vector name [S1, unofficial].

7.  **Geometric Attribute Computation:**
    *   Compute vertex normals using `mesh_utils::computeNormals` [S1, unofficial].
    *   Compute bounding box using `mesh_utils::computeBounds` [S1, unofficial].
    *   If normals are empty but vertices exist, assign default normals (e.g., [0,0,1]) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def finalize_mesh(mesh: RenderMesh) -> bool:
    # 1. Validation
    if len(mesh.vertices) == 0:
        log_error("Mesh vertices are empty")
        return False
    
    if len(mesh.indices) == 0 and not mesh.is_polydata and not mesh.render_as_points:
        log_error("Mesh indices are empty for non-polydata/non-point mesh")
        return False
    
    n_verts = len(mesh.vertices) // 3
    for idx in mesh.indices:
        if idx >= n_verts:
            log_error(f"Index {idx} out of range for {n_verts} vertices")
            mesh.indices = []
            return False

    # 2. Extrapolate Cell Data
    mesh_utils.extrapolate_cell_data_to_points(mesh)

    # 3. Flatten Vector Buffers
    # Flatten point vectors
    mesh.point_vectors_data = []
    mesh.point_vector_offset = []
    current_offset = 0
    for field_name, field_data in mesh.point_vectors.items():
        mesh.point_vector_offset.append(current_offset)
        mesh.point_vectors_data.extend(field_data)
        current_offset += len(field_data)
    mesh.point_vector_count = 3  # Per vertex components

    # Flatten cell vectors
    mesh.cell_vectors_data = []
    mesh.cell_vector_offset = []
    current_offset = 0
    for field_name, field_data in mesh.cell_vectors.items():
        mesh.cell_vector_offset.append(current_offset)
        mesh.cell_vectors_data.extend(field_data)
        current_offset += len(field_data)
    mesh.cell_vector_count = len(mesh.cells)
    
    # Default cell vector
    if not mesh.cell_vector_name and mesh.cell_vectors:
        mesh.cell_vector_name = sorted(mesh.cell_vectors.keys())[0]

    # 4. Compute Cell Centers
    mesh.cell_centers = []
    for cell in mesh.cells:
        center = [0.0, 0.0, 0.0]
        k = len(cell)
        for v_idx in cell:
            v = mesh.vertices[v_idx*3 : v_idx*3+3]
            center[0] += v[0]
            center[1] += v[1]
            center[2] += v[2]
        mesh.cell_centers.append([c / k for c in center])

    # 5. Select Active Scalar
    if not mesh.scalar_name or mesh.scalar_name not in mesh.point_scalars:
        if mesh.point_scalars:
            mesh.scalar_name = sorted(mesh.point_scalars.keys())[0]
        else:
            mesh.scalar_name = None
    
    if mesh.scalar_name:
        if len(mesh.point_scalars[mesh.scalar_name]) != n_verts:
            log_error("Scalar size mismatch")
            mesh.point_scalars = {}
            mesh.scalar_name = None

    # 6. Populate UI Names
    mesh.available_scalars = sorted(mesh.point_scalars.keys())
    mesh.available_vectors = sorted(mesh.cell_vectors.keys())
    
    if not mesh.vector_name and mesh.cell_vectors:
        mesh.vector_name = sorted(mesh.cell_vectors.keys())[0]

    # 7. Compute Normals and Bounds
    mesh_utils.compute_bounds(mesh)
    mesh_utils.compute_normals(mesh)
    
    if not mesh.normals and len(mesh.vertices) > 0:
        # Default normal if computation fails or returns empty
        mesh.normals = [0.0, 0.0, 1.0] * n_verts

    return True
```

## Operating characteristics
*   **Time Complexity:** Dominated by cell data extrapolation and normal computation, typically $O(N_{verts} + N_{cells})$. Flattening vectors is $O(N_{fields} \times N_{points})$.
*   **Memory Complexity:** Increases memory usage by storing flattened vector buffers and computed normals/centers. The flattening step may double memory usage for vector fields temporarily.
*   **Strengths:** Ensures data integrity for complex unstructured meshes; handles mixed cell types (polyhedra, polygons); provides robust fallbacks for missing attributes.
*   **Limitations:** Extrapolation can introduce artifacts if cell topology is degenerate; validation checks add overhead but are necessary for safety.
*   **When to use:** After assembling raw mesh data from CFD solvers or unstructured grid generators, before passing to a renderer.
*   **When not to use:** For simple, already-validated structured grids where performance is critical and data integrity is guaranteed by the source.

## Background
Mesh Finalization is a core component of the SciRender library, developed by thermeng [S1, unofficial]. It is designed to bridge the gap between raw finite volume method (FVM) mesh data and visualization backends, handling the complexities of unstructured data that standard blocky meshing methods do not address. It is particularly relevant in scientific visualization where cell-centered data is common.

## Sources
[S1] thermeng/SciRender, https://github.com/thermeng/SciRender, fetched 2026-10-26 (unofficial)
