# Boolean Voxel Meshing

Also called: VoxelGrid to Mesh, VoxelGrid To Mesh Hull, VoxelGrid Mesh Hull Generation, voxel-to-mesh, VoxMeshHull

## Overview
Boolean Voxel Meshing converts a 3D boolean voxel occupancy grid into a triangular mesh by detecting faces and vertices at voxel boundaries [S1, unofficial]. It solves the problem of generating a closed, watertight surface representation from discrete occupancy data, specifically by sharing vertices and faces across adjacent voxels to minimize redundancy [S6, unofficial]. It differs from [Blocky meshing](31-blocky-meshing.md) variants like [Cuberille Method](1718-cuberille-method.md) by explicitly constructing a shared-vertex mesh rather than independent quads, and from [VoxelGrid Mesh Generation](2714-voxelgrid-mesh-generation.md) by ensuring topological connectivity (no T-junctions) through boundary detection rather than instantiating independent blocks [G].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D boolean occupancy grid `occ` of dimensions $(N_x, N_y, N_z)$.
*   **Output:** A mesh defined by a vertex array `vertices` and an index array `faces`.

**Data Structures:**
*   `occ`: 3D boolean array.
*   `v`: 3D boolean array of shape $(N_x+1, N_y+1, N_z+1)$ representing vertex existence.
*   `v_idx`: 3D integer array of shape $(N_x+1, N_y+1, N_z+1)$ mapping grid coordinates to linear vertex indices. Initialized to -1.
*   `vertices`: List of 3D coordinates.
*   `faces`: List of 4-element integer arrays (quads) or 6-element integer arrays (triangles).

**Steps:**

1.  **Padding:** Pad the occupancy grid with a constant boundary of `False` (empty) to handle edge cases cleanly.
    ```python
    occ = np.pad(occ, 1, 'constant', constant_values=False)
    ```
    Let the padded dimensions be $(N_x+2, N_y+2, N_z+2)$. The internal grid indices $x \in [1, N_x+1]$, $y \in [1, N_y+1]$, $z \in [1, N_z+1]$ correspond to the original voxels.

2.  **Boundary Face Detection:** Compute the boolean masks for faces on the boundaries between occupied and unoccupied voxels. A face exists if one adjacent voxel is occupied and the other is not.
    *   **X-faces (Normals $\pm \hat{x}$):**
        $$f1\_r = occ[:-1, 1:-1, 1:-1] \ \& \ \sim occ[1:, 1:-1, 1:-1]$$
        $$f1\_l = \sim occ[:-1, 1:-1, 1:-1] \ \& \ occ[1:, 1:-1, 1:-1]$$
        $$f1 = f1\_r \ | \ f1\_l$$
        Shape: $(N_x+1, N_y, N_z)$.
    *   **Y-faces (Normals $\pm \hat{y}$):**
        $$f2\_r = occ[1:-1, :-1, 1:-1] \ \& \ \sim occ[1:-1, 1:, 1:-1]$$
        $$f2\_l = \sim occ[1:-1, :-1, 1:-1] \ \& \ occ[1:-1, 1:, 1:-1]$$
        $$f2 = f2\_r \ | \ f2\_l$$
        Shape: $(N_x, N_y+1, N_z)$.
    *   **Z-faces (Normals $\pm \hat{z}$):**
        $$f3\_r = occ[1:-1, 1:-1, :-1] \ \& \ \sim occ[1:-1, 1:-1, 1:]$$
        $$f3\_l = \sim occ[1:-1, 1:-1, :-1] \ \& \ occ[1:-1, 1:-1, 1:]$$
        $$f3 = f3\_r \ | \ f3\_l$$
        Shape: $(N_x, N_y, N_z+1)$.

3.  **Vertex Grid Construction:** Initialize a vertex existence grid `v` of shape $(N_x+1, N_y+1, N_z+1)$ to `False`. Mark vertices as existing if they are part of any detected face. Each face contributes to 4 vertices.
    *   For $f1$ (X-faces):
        ```python
        v[:, :-1, :-1] |= f1
        v[:, :-1, 1:] |= f1
        v[:, 1:, :-1] |= f1
        v[:, 1:, 1:] |= f1
        ```
    *   For $f2$ (Y-faces):
        ```python
        v[:-1, :, :-1] |= f2
        v[:-1, :, 1:] |= f2
        v[1:, :, :-1] |= f2
        v[1:, :, 1:] |= f2
        ```
    *   For $f3$ (Z-faces):
        ```python
        v[:-1, :-1, :] |= f3
        v[:-1, 1:, :] |= f3
        v[1:, :-1, :] |= f3
        v[1:, 1:, :] |= f3
        ```

4.  **Vertex Index Mapping:** Create a linear index for each existing vertex.
    ```python
    v_idx = np.full((N_x+1, N_y+1, N_z+1), -1)
    v_idx[v] = np.arange(np.sum(v))
    ```

5.  **Vertex Coordinate Extraction:** Extract coordinates for existing vertices and normalize them to $[-0.5, 0.5]$.
    ```python
    x_coords, y_coords, z_coords = np.where(v)
    # Normalize to [-0.5, 0.5] based on grid dimensions
    # Note: The source uses nx, ny, nz as the original dimensions.
    # The padded grid has dimensions (nx+2, ny+2, nz+2).
    # The vertices correspond to the grid points.
    # Formula from source: v_x = v_x/nx - 0.5
    vertices_x = x_coords / N_x - 0.5
    vertices_y = y_coords / N_y - 0.5
    vertices_z = z_coords / N_z - 0.5
    vertices = np.stack([vertices_x, vertices_y, vertices_z], axis=1)
    ```

6.  **Face Index Construction:** Extract the coordinates of the faces and map them to vertex indices.
    *   Get coordinates for each face type:
        ```python
        f1_l_x, f1_l_y, f1_l_z = np.where(f1_l)
        f1_r_x, f1_r_y, f1_r_z = np.where(f1_r)
        f2_l_x, f2_l_y, f2_l_z = np.where(f2_l)
        f2_r_x, f2_r_y, f2_r_z = np.where(f2_r)
        f3_l_x, f3_l_y, f3_l_z = np.where(f3_l)
        f3_r_x, f3_r_y, f3_r_z = np.where(f3_r)
        ```
    *   Map to vertex indices using `v_idx`.
    *   Construct quad indices (4 vertices per face):
        *   **f1_l (Left X):** `[v_idx[x,y,z], v_idx[x,y,z+1], v_idx[x,y+1,z+1], v_idx[x,y+1,z]]`
        *   **f1_r (Right X):** `[v_idx[x,y,z], v_idx[x,y+1,z], v_idx[x,y+1,z+1], v_idx[x,y,z+1]]`
        *   **f2_l (Left Y):** `[v_idx[x,y,z], v_idx[x+1,y,z], v_idx[x+1,y,z+1], v_idx[x,y,z+1]]`
        *   **f2_r (Right Y):** `[v_idx[x,y,z], v_idx[x,y,z+1], v_idx[x+1,y,z+1], v_idx[x+1,y,z]]`
        *   **f3_l (Left Z):** `[v_idx[x,y,z], v_idx[x,y+1,z], v_idx[x+1,y+1,z], v_idx[x+1,y,z]]`
        *   **f3_r (Right Z):** `[v_idx[x,y,z], v_idx[x+1,y,z], v_idx[x+1,y+1,z], v_idx[x,y+1,z]]`
    *   Concatenate all face arrays:
        ```python
        faces = np.concatenate([faces_1_l, faces_1_r, faces_2_l, faces_2_r, faces_3_l, faces_3_r], axis=0)
        ```

7.  **Transformation (Optional):** Transform vertices to world space using a location `loc` and scale `scale`.
    ```python
    vertices = loc + scale * vertices
    ```

8.  **Mesh Creation:** Create the final mesh object.
    ```python
    mesh = trimesh.Trimesh(vertices, faces, process=False)
    ```

**Parameters:**
*   `desiredStep` (float): Target voxel edge length used to compute optimal grid dimensions. Default: 0.2f [S2, unofficial].
*   `Fake Shadow` (bool): Adds a fake shadow to the mesh for visualization. Default: false [S4, unofficial].
*   `Light Direction` (Vector3d): Fixed vector for fake shadow calculation: $(-0.495633, 0.142501, 0.856762)$ [S4, unofficial].
*   `Shadow Colors`: Highlight: `Color.White`, Shadow: `Color.Black` [S4, unofficial].
*   `color` (RGBA): Mesh color. Default: null [S5, unofficial].
*   `convertToTriangles` (bool): Convert quads to triangles. Default: true [S5, unofficial].
*   `flatten` (bool): Flatten output structure. Default: true [S5, unofficial].

**Edge Cases and Pitfalls:**
*   **T-junctions:** If shared faces are not properly culled or vertices not consolidated, T-junctions may appear, causing visible discontinuities [S5, unofficial].
*   **Closed Mesh:** A closed mesh is required for subsequent smoothing algorithms like Catmull-Clark [S5, unofficial].
*   **Directional Meshes:** If generating 6 directional plane meshes, an error occurs if the count is not exactly 6 [S4, unofficial].
*   **Hardcoded Indices:** Cube faces are hardcoded; manual indexing is required for all 12 triangles per cube if using the independent cube variant [S2, unofficial].

## Subtopics

### [Voxel Occupancy Boundary Detection](1011-voxel-occupancy-boundary-detection.md)
Voxel Occupancy Boundary Detection identifies voxels in a 3D occupancy grid that lie on the interface between occupied and unoccupied space. It solves the problem of converting a discrete boolean volume into a set of boundary indicators required for surface extraction algorithms, such as Boolean Voxel Meshing [G]. It differs from [Octree Traversal for Surface Extraction](2113-octree-traversal-for-surface-extraction.md) by operating on a fixed-resolution grid rather than a hierarchical tree, and from [VoxelGrid Mesh Generation](2714-voxelgrid-mesh-generation.md) by outputting a boolean mask of boundary voxels rather than a mesh directly [G].

### [Octree Traversal for Surface Extraction](2113-octree-traversal-for-surface-extraction.md)
This method extracts a triangular mesh from a boolean voxel occupancy grid by recursively traversing an octree structure and generating geometry for leaf nodes. It differs from interpolation-based methods like [Marching Cubes](52-marching-cubes.md) by outputting axis-aligned cube meshes directly from octree leaves, resulting in simpler but blocky geometry unless voxel resolution is high. It builds on [Boolean Voxel Meshing](971-boolean-voxel-meshing.md) by handling hierarchical data structures rather than flat grids.

### [VoxelGrid Mesh Generation](2714-voxelgrid-mesh-generation.md)
VoxelGrid Mesh Generation constructs a combined triangle mesh from a sparse set of voxel linear IDs or a predicate-defined subset within a `VoxelGrid`. It transforms a base cell mesh into world space for each selected voxel and adjusts triangle indices to maintain connectivity across the flattened output array. Unlike boundary-detection methods, it does not share vertices between voxels and generates a full independent geometry block for every selected voxel.

### [Voxel Grid to Triangle Mesh Conversion](6291-voxel-grid-to-triangle-mesh-conversion.md)
This method converts a discrete boolean voxel occupancy grid into a triangular mesh by instantiating a box primitive for every occupied voxel. Unlike smooth surface extraction techniques (e.g., [Marching Cubes](52-marching-cubes.md)) or boundary-aware methods (e.g., [Voxel Occupancy Boundary Detection](1011-voxel-occupancy-boundary-detection.md)), this approach preserves the exact blocky geometry of the input grid without attempting to smooth or merge adjacent faces. It is primarily used when the voxel structure itself is the desired output, such as for animation or when the input data is already voxelized and no underlying scalar field exists.

### [Blocky Surface Extraction](6418-blocky-surface-extraction.md)
Blocky Surface Extraction generates a voxel-aligned triangle mesh by iterating over voxels inside an isosurface and adding faces to the exterior boundary where adjacent voxels are outside the shape. It differs from [Boolean Voxel Meshing](971-boolean-voxel-meshing.md) by explicitly constructing quad faces from voxel corners rather than detecting boundary edges, resulting in a "blocky" aesthetic where faces are strictly axis-aligned.

### [VoxelBodyMeshing](6947-voxelbodymeshing.md)
VoxelBodyMeshing is a blocky surface extraction method that converts a boolean voxel occupancy grid into a triangular mesh by processing sub-chunks of a `VoxelBody`. It builds on [Boolean Voxel Meshing](971-boolean-voxel-meshing.md) by handling the meshing of individual chunks in a multi-threaded worker loop, supporting both hard-edged (face-merging) and smooth (Marching Cubes) modes. It is distinct from sibling methods by its specific integration with the `VoxelEngine` architecture, using a thread-local `MeshGrid` and `MeshBuilder` to output `ChunkMeshData` parts that are transformed into body-local coordinates to allow shared model matrices.

### Voxels Grid to Compressed Mesh Conversion
Converts a voxel grid into a compressed mesh representation using a Marching Cubes-like approach, sharing vertices and faces across adjacent voxels to reduce redundancy [S6, unofficial]. It initializes six predefined normals for the mesh: $(0,0,1), (0,1,0), (1,0,0), (0,0,-1), (0,-1,0), (-1,0,0)$ [S6, unofficial]. It iterates through all voxels; if a voxel is empty, it is skipped. Otherwise, it calls `AddFacesVertexXY`, `AddFacesVertexXZ`, and `AddFacesVertexYZ` for both false and true orientations to generate faces and vertices [S6, unofficial]. It reserves and assigns white color $(1.0, 1.0, 1.0, 1.0)$ to all vertices [S6, unofficial]. Data structures include `vertices_marked`: `std::unordered_map<uint, std::pair<uint, Position>>` mapping a unique vertex ID to a pair of (face index, Position), and `faces_marked`: `std::array<std::vector<bool>, 3>` containing three vectors of booleans, each of size `max_faces_num`, tracking which faces have been added for each axis (X, Y, Z) [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_x N_y N_z)$ for grid traversal and face detection. Vertex and index construction is proportional to the surface area.
*   **Memory Complexity:** $O(N_x N_y N_z)$ for the occupancy grid and intermediate boolean masks. The output mesh size is proportional to the surface area.
*   **Performance:** Culling shared faces reduces mesh size and rendering time compared to independent voxel cubes [S5, unofficial].
*   **Strengths:** Produces watertight, closed meshes suitable for smoothing and physics. Efficiently handles large grids.
*   **Limitations:** Geometry is strictly axis-aligned (blocky). High resolution grids require significant memory for intermediate arrays.
*   **When to use:** When a clean, shared-vertex mesh from a boolean grid is needed, especially for further processing like smoothing.
*   **When not to use:** When smooth, curved surfaces are required (use Marching Cubes) or when hierarchical data structures are preferred (use Octree Traversal).

## Background
Boolean Voxel Meshing is a technique used in voxel-based world generation and 3D modeling to convert discrete voxel data into polygonal meshes [S1, unofficial]. It is used in tools like OBJ-Solidizer for exporting voxelized geometry as OBJ files [S2, unofficial]. The method is also referenced in WebGL contexts for converting voxel arrays to mesh data [S5, unofficial]. Variants like MicroBlockBuilder allow for complex shapes (stairs, slabs) by subdividing voxels, though this is a distinct geometry builder approach [S3, unofficial]. The technique is fundamental to blocky meshing pipelines, serving as a bridge between discrete occupancy data and continuous surface representations.

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] antonmartynov/OBJ-Solidizer-And-Voxelizer, https://github.com/antonmartynov/OBJ-Solidizer-And-Voxelizer, fetched 2026-10-27 (unofficial)
[S3] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
[S4] voxeltool, https://github.com/arendvw/voxeltools, fetched 2026-10-29 (unofficial)
[S5] voxel-to-mesh | Converts voxel array to webgl mesh data, https://jzwood.github.io/voxel-to-mesh/, fetched 2026-10-30 (unofficial)
[S6] bigmat18/cuda-mesh-voxelization, https://github.com/bigmat18/cuda-mesh-voxelization, fetched 2026-10-26 (unofficial)
