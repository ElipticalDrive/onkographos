# Voxels Grid to Compressed Mesh Conversion

Also called: VoxelsGridToMeshCompressed

## Overview
This method converts a boolean voxel occupancy grid into a compressed triangular mesh by detecting faces and vertices at voxel boundaries and sharing them across adjacent voxels to reduce redundancy [S1, unofficial]. It differs from [Voxel Grid to Triangle Mesh Conversion](6291-voxel-grid-to-triangle-mesh-conversion.md) by explicitly optimizing for vertex and face sharing to produce a "compressed" representation, and from [Greedy Triangle Meshing](15263-greedy-triangle-meshing.md) by relying on a deterministic, axis-aligned face generation strategy rather than greedy region growing [S1, unofficial].

## How it works
**Inputs:** A boolean voxel occupancy grid (implicitly or explicitly defined by `VoxelsPerSide` and occupancy status per voxel).
**Outputs:** A triangular mesh consisting of vertices, normals, and face indices.

**Data Structures:**
1.  **`vertices_marked`**: A hash map (`std::unordered_map<uint, std::pair<uint, Position>>`) mapping a unique vertex ID to a pair containing the face index it belongs to and its 3D position [S1, unofficial].
2.  **`faces_marked`**: An array of three vectors (`std::array<std::vector<bool>, 3>`), where each vector has size `max_faces_num` and tracks whether a face has already been added for the X, Y, and Z axes respectively [S1, unofficial].
3.  **Mesh Storage**: Implicitly holds vertices, normals, and face indices.

**Parameters:**
*   `grid.VoxelsPerSide()`: The number of voxels along one side of the grid, denoted as $N$.
*   `max_vertices_num`: Calculated as $(N + 1)^3$.
*   `max_faces_num`: Calculated as $N^2 \times (N + 1)$.
*   Normals: Six predefined unit normals: $(0,0,1), (0,1,0), (1,0,0), (0,0,-1), (0,-1,0), (-1,0,0)$ [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization**:
    *   Clear the mesh.
    *   Calculate `max_vertices_num` and `max_faces_num` based on grid dimensions.
    *   Initialize `faces_marked` with three boolean vectors of size `max_faces_num`, all set to `false`.
    *   Initialize `vertices_marked` as empty.

2.  **Iteration**:
    *   Iterate through all voxel coordinates $(x, y, z)$ in the grid.
    *   If the voxel at $(x, y, z)$ is empty (false), skip to the next voxel.
    *   If the voxel is occupied (true), call the following helper functions to generate geometry:
        *   `AddFacesVertexXY`
        *   `AddFacesVertexXZ`
        *   `AddFacesVertexYZ`
    *   These helpers are called for both "false" and "true" orientations to ensure correct boundary detection [S1, unofficial].

3.  **Finalization**:
    *   Assign a white color $(1.0f, 1.0f, 1.0f, 1.0f)$ to all vertices in the mesh [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxels_grid_to_compressed_mesh(grid):
    # 1. Initialization
    N = grid.VoxelsPerSide()
    max_vertices_num = (N + 1) ** 3
    max_faces_num = (N ** 2) * (N + 1)
    
    mesh = Mesh()
    vertices_marked = {}  # Map: uint -> (face_index, Position)
    faces_marked = [
        [False] * max_faces_num,  # X-axis faces
        [False] * max_faces_num,  # Y-axis faces
        [False] * max_faces_num   # Z-axis faces
    ]
    
    # Predefined normals
    normals = [
        (0, 0, 1), (0, 1, 0), (1, 0, 0),
        (0, 0, -1), (0, -1, 0), (-1, 0, 0)
    ]
    
    # 2. Iterate through voxels
    for x in range(N):
        for y in range(N):
            for z in range(N):
                if not grid.is_occupied(x, y, z):
                    continue
                
                # Generate faces for occupied voxel
                # The source specifies calling these for both false and true orientations
                # This implies checking boundaries with neighbors or internal logic within helpers
                AddFacesVertexXY(x, y, z, grid, mesh, vertices_marked, faces_marked)
                AddFacesVertexXZ(x, y, z, grid, mesh, vertices_marked, faces_marked)
                AddFacesVertexYZ(x, y, z, grid, mesh, vertices_marked, faces_marked)
                
    # 3. Finalization
    for vertex in mesh.vertices:
        vertex.color = (1.0, 1.0, 1.0, 1.0)
        
    return mesh

def AddFacesVertexXY(x, y, z, grid, mesh, vertices_marked, faces_marked):
    # Implementation details for XY plane face generation
    # This function must handle vertex sharing via vertices_marked
    # and face deduplication via faces_marked
    pass # Specific internal logic for AddFacesVertexXY is not fully detailed in source [S1, unofficial]

def AddFacesVertexXZ(x, y, z, grid, mesh, vertices_marked, faces_marked):
    # Implementation details for XZ plane face generation
    pass

def AddFacesVertexYZ(x, y, z, grid, mesh, vertices_marked, faces_marked):
    # Implementation details for YZ plane face generation
    pass
```

**Edge Cases and Pitfalls:**
*   **Precision**: The method relies on integer grid coordinates. Ensure grid boundaries are handled correctly to avoid out-of-bounds access when checking neighbors [G].
*   **Redundancy**: The `faces_marked` and `vertices_marked` structures are critical for compression. Failure to correctly update these will result in a non-compressed, redundant mesh [S1, unofficial].

## Subtopics

### [Vosh (Voxel-Mesh Hybrid Representation)](19051-vosh-voxel-mesh-hybrid-representation.md)
Vosh is a hybrid scene representation for NeRF-based real-time view synthesis that combines a sparse voxel grid with a compressed mesh to balance rendering quality and computational speed. It differs from pure voxel methods by extracting a mesh for high-frequency geometry and from pure mesh methods by retaining voxels for volumetric details, using differentiable rendering to optimize the mesh geometry and topology.

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of occupied voxels, $O(V)$, where $V$ is the number of voxels. Each voxel is processed once, and face/vertex operations are $O(1)$ on average due to hash map lookups [G].
*   **Memory Complexity**: Proportional to the number of vertices and faces in the output mesh. The `faces_marked` array requires $O(N^3)$ space in the worst case (where $N$ is grid side length), which can be significant for large grids [S1, unofficial].
*   **Strengths**: Produces a compact mesh by sharing vertices and faces. Deterministic and simple to implement.
*   **Limitations**: The resulting mesh is blocky and axis-aligned, lacking smooth curvature. The pre-allocation of `faces_marked` can be memory-intensive for very large grids.
*   **When to use**: When a compact, blocky representation of a voxel grid is needed for rendering or further processing, and smoothness is not required.
*   **When not to use**: When smooth surfaces are required, or when memory constraints prevent pre-allocating the face tracking arrays.

## Background
This method is described in the context of a CUDA-based mesh voxelization library [S1, unofficial]. It is part of a pipeline that includes mesh voxelization, CSG operations, and SDF calculation. The specific algorithm for compressed mesh conversion is attributed to the implementation details of the `cuda-mesh-voxelization` project [S1, unofficial].

## Sources
[S1] bigmat18/cuda-mesh-voxelization, https://github.com/bigmat18/cuda-mesh-voxelization, fetched 2026-10-26 (unofficial)
