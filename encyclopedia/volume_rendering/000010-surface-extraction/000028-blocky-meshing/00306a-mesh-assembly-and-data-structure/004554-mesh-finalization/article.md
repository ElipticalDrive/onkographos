# Mesh Finalization

Also called: VTK Mesh Finalization

## Overview
Mesh Finalization prepares a raw geometric representation (typically from a CFD or unstructured grid solver) for rendering by validating data integrity, extrapolating cell-centered fields to vertices, and computing derived attributes like normals and bounds [S1, unofficial]. It differs from blocky meshing assembly, which aggregates pre-computed quads into buffers, by operating on topological cell definitions to generate the actual vertex positions, connectivity, and scalar/vector data structures required by the rendering pipeline [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A solver mesh object `m` containing vertices, cells, faces, and neighbor relationships.
*   **Output:** A `RenderMesh` structure containing flattened vertex arrays, index buffers, cell connectivity, cell centers, and computed normals/bounds.

**Data Structures:**
The `RenderMesh` structure includes:
*   `vertices`: `std::vector<float>` (3 floats per vertex: x, y, z).
*   `indices`: `std::vector<std::uint32_t>` (triangular indices).
*   `cellConnectivity`: `std::vector<std::uint32_t>` (vertex indices per cell).
*   `cellTypes`: `std::vector<std::uint8_t>` (topology type per cell).
*   `cellOffsets`: `std::vector<std::uint32_t>` (start index in `cellConnectivity` for each cell).
*   `cellCenters`: `std::vector<Vec3>` (x, y, z per cell).
*   `normals`: `std::vector<float>` (3 floats per vertex).
*   `bounds`: Computed bounding box.
*   `pointScalars`, `cellScalars`, `pointVectors`, `cellVectors`: Field data containers.

**Steps:**
1.  **Validation:** If `mesh.vertices` is empty, log an error and return [S1, unofficial]. If `mesh.indices` is empty and the mesh is not `POLYDATA` and not `renderAsPoints`, log an error and return [S1, unofficial].
2.  **Index Validation:** Validate that all indices are `< vertexCount`. If any index is out of range, log an error, clear indices, and return to prevent GPU out-of-range fetches [S1, unofficial].
3.  **Vertex Extraction:** Iterate over solver vertices `m.vertices()`. For each vertex `v`, push `v[0]`, `v[1]`, and `v[2]` (or `0.0f` for 2D) into `rm.vertices` [S1, unofficial].
4.  **Index Generation:**
    *   **2D Case:** For each cell, fan-triangulate the polygon. For a cell with vertices `c[0...n]`, add indices `(c[0], c[i], c[i+1])` for `i` from 1 to `n-2` [S1, unofficial].
    *   **3D Case:** Identify external faces (faces where `m.neighbor(f) == kNoNeighbor`). For each external face loop, fan-triangulate using `loop[0]`, `loop[i]`, `loop[i+1]`. Collect unique edges for `cellEdgeIndices` [S1, unofficial].
5.  **Cell Topology & Connectivity:** For each cell:
    *   Determine canonical vertex order and type using `cell_topology::canonicalize` [S1, unofficial].
    *   Push canonical vertex indices to `rm.cellConnectivity`.
    *   Push type to `rm.cellTypes`.
    *   Push current `cellConnectivity.size()` to `rm.cellOffsets` [S1, unofficial].
6.  **Cell Centers:** Compute `m.cellCenter(c)` for each cell `c` and store in `rm.cellCenters` [S1, unofficial].
7.  **Field Extrapolation:** Call `mesh_utils::extrapolateCellDataToPoints` to move cell-centered data to vertices [S1, unofficial].
8.  **Vector Flattening:**
    *   Flatten point vectors into `mesh.pointVectorsData` with offsets `mesh.pointVectorOffset`. Set `mesh.pointVectorCount` to `perVertex` [S1, unofficial].
    *   Flatten cell vectors into `mesh.cellVectorsData`. Set `mesh.cellVectorCount` to `cellCount`. Record available cell vector names [S1, unofficial].
    *   If no `cellVectorName` is set, default to the first available cell vector name [S1, unofficial].
9.  **Scalar Selection:**
    *   If `scalarName` is empty or not in `pointScalars`, default to the first point scalar [S1, unofficial].
    *   Validate scalar size matches vertex count; if not, clear scalars [S1, unofficial].
10. **UI Population:** Sort available scalar and vector names alphabetically for UI display [S1, unofficial].
11. **Vector Name Default:** If no `vectorName` is set, default to the first available vector name [S1, unofficial].
12. **Normals & Bounds:** Compute normals and bounds using `mesh_utils::computeNormals` and `mesh_utils::computeBounds` [S1, unofficial]. If normals are empty but vertices exist, assign default normals [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def finalize_mesh(solver_mesh):
    rm = RenderMesh()
    rm.datasetType = "CFD MESH"
    rm.fileFormat = "MSH"
    
    # 1. Validation
    if len(solver_mesh.vertices) == 0:
        log_error("Empty vertices")
        return None
    
    # 2. Index Validation (if indices exist)
    if solver_mesh.indices and len(solver_mesh.indices) > 0:
        for idx in solver_mesh.indices:
            if idx >= len(solver_mesh.vertices):
                log_error("Index out of range")
                rm.indices = []
                return None
    
    # 3. Vertex Extraction
    for v in solver_mesh.vertices:
        rm.vertices.append(v[0])
        rm.vertices.append(v[1])
        if solver_mesh.dim == 3:
            rm.vertices.append(v[2])
        else:
            rm.vertices.append(0.0)
            
    # 4. Index Generation
    if solver_mesh.dim == 2:
        for cell in solver_mesh.cells:
            if len(cell) < 3: continue
            for i in range(1, len(cell) - 1):
                rm.indices.append(cell[0])
                rm.indices.append(cell[i])
                rm.indices.append(cell[i + 1])
    else:
        seen_edges = set()
        edges = []
        for f in range(solver_mesh.nFaces()):
            if solver_mesh.neighbor(f) != kNoNeighbor: continue
            loop = solver_mesh.faceVertices(f)
            if len(loop) < 3: continue
            for i in range(1, len(loop) - 1):
                rm.indices.append(loop[0])
                rm.indices.append(loop[i])
                rm.indices.append(loop[i + 1])
            for i in range(len(loop)):
                a = loop[i]
                b = loop[(i + 1) % len(loop)]
                edge_key = min(a, b) * 100000 + max(a, b) # Simplified key
                if edge_key not in seen_edges:
                    seen_edges.add(edge_key)
                    edges.append(a)
                    edges.append(b)
        rm.cellEdgeIndices = edges

    # 5. Cell Topology
    cell_index = 0
    for cell in solver_mesh.cells:
        canonical = []
        type_code = kPolyhedron
        canonical_cell(solver_mesh, cell_index, canonical, type_code)
        for vi in canonical:
            rm.cellConnectivity.append(vi)
        rm.cellTypes.append(type_code)
        rm.cellOffsets.append(len(rm.cellConnectivity))
        cell_index += 1

    # 6. Cell Centers
    for c in range(solver_mesh.nCells()):
        cc = solver_mesh.cellCenter(c)
        rm.cellCenters.append(cc)
    rm.cellVectorCount = solver_mesh.nCells()

    # 7. Field Extrapolation
    extrapolate_cell_data_to_points(solver_mesh, rm)

    # 8. Vector Flattening
    flatten_point_vectors(solver_mesh, rm)
    flatten_cell_vectors(solver_mesh, rm)
    if not rm.cellVectorName:
        rm.cellVectorName = get_first_available_cell_vector(solver_mesh)

    # 9. Scalar Selection
    if not rm.scalarName or rm.scalarName not in rm.pointScalars:
        rm.scalarName = get_first_point_scalar(solver_mesh)
    if len(rm.pointScalars[rm.scalarName]) != len(rm.vertices) / 3:
        rm.pointScalars.clear()

    # 10. UI Population
    rm.available_scalars = sort_alphabetically(get_scalar_names(solver_mesh))
    rm.available_vectors = sort_alphabetically(get_vector_names(solver_mesh))

    # 11. Vector Name Default
    if not rm.vectorName:
        rm.vectorName = get_first_available_vector(solver_mesh)

    # 12. Normals & Bounds
    compute_bounds(rm)
    compute_normals(rm)
    if not rm.normals and rm.vertices:
        rm.normals = assign_default_normals(rm.vertices)
        
    return rm
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of vertices, cells, and faces. Topology canonicalization and neighbor checks are constant per cell/face.
*   **Memory Complexity:** Proportional to the size of the mesh data. Flattening vectors adds overhead proportional to the number of vector fields.
*   **Strengths:** Handles arbitrary unstructured cell types (polyhedra, polygons). Ensures data consistency for rendering by validating indices and extrapolating fields.
*   **Limitations:** Requires solver to provide valid topology. 2D meshes are treated as surface polygons.
*   **When to use:** When converting solver output (CFD, FEM) to a renderable format. Not suitable for blocky voxel meshes which use assembly.

## Background
Originated in the SciRender project (thermeng/SciRender) for visualizing CFD meshes [S1, unofficial]. It adapts VTK-like concepts (cell data extrapolation, unstructured grid handling) to a custom C++ rendering pipeline.

## Sources
[S1] thermeng/SciRender, https://github.com/thermeng/SciRender, fetched 2026-10-26 (unofficial)
