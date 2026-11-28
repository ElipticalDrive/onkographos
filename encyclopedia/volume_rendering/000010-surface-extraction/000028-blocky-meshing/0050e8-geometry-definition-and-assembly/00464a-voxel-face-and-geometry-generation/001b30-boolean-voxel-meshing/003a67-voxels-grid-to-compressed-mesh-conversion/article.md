# Voxels Grid to Compressed Mesh Conversion

Also called: VoxelsGridToMeshCompressed

## Overview
This method converts a boolean voxel occupancy grid into a compressed triangular mesh by detecting faces and vertices at voxel boundaries and sharing them across adjacent voxels to reduce redundancy [S1, unofficial]. It differs from [Voxel Grid to Triangle Mesh Conversion](6291-voxel-grid-to-triangle-mesh-conversion.md) by explicitly optimizing for vertex and face sharing to produce a compressed representation [S1, unofficial].

## How it works
**Inputs:** A boolean voxel occupancy grid defined by `VoxelsPerSide` and per-voxel occupancy status [S1, unofficial].
**Outputs:** A triangular mesh containing vertices, normals, and face indices [S1, unofficial].

**Data Structures:**
1.  **`vertices_marked`**: `std::unordered_map<uint, std::pair<uint, Position>>` mapping a unique vertex ID to a pair of (face index, 3D position) [S1, unofficial].
2.  **`faces_marked`**: `std::array<std::vector<bool>, 3>` containing three boolean vectors of size `max_faces_num`, tracking which faces have been added for the X, Y, and Z axes respectively [S1, unofficial].
3.  **Mesh Storage**: Implicitly holds vertices, normals, and face indices [S1, unofficial].

**Parameters:**
*   `N = grid.VoxelsPerSide()`
*   `max_vertices_num = (N + 1)^3`
*   `max_faces_num = N^2 \times (N + 1)`
*   Six predefined unit normals: $(0,0,1), (0,1,0), (1,0,0), (0,0,-1), (0,-1,0), (-1,0,0)$ [S1, unofficial].

**Algorithm Steps:**
1.  Clear the mesh and calculate `max_vertices_num` and `max_faces_num` [S1, unofficial].
2.  Initialize `faces_marked` with three boolean vectors of size `max_faces_num`, all set to `false` [S1, unofficial].
3.  Initialize `vertices_marked` as empty [S1, unofficial].
4.  Iterate through all voxel coordinates $(x, y, z)$ in the grid [S1, unofficial].
5.  If the voxel at $(x, y, z)$ is empty (false), skip to the next voxel [S1, unofficial].
6.  If the voxel is occupied (true), call `AddFacesVertexXY`, `AddFacesVertexXZ`, and `AddFacesVertexYZ` for both false and true orientations to generate faces and vertices [S1, unofficial].
7.  Reserve and assign white color $(1.0f, 1.0f, 1.0f, 1.0f)$ to all vertices in the mesh [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Position:
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z
    def __hash__(self): return hash((self.x, self.y, self.z))
    def __eq__(self, other): return (self.x, self.y, self.z) == (other.x, other.y, other.z)

class Mesh:
    def __init__(self):
        self.vertices = []
        self.normals = []
        self.faces = []
    def add_vertex(self, pos, normal):
        self.vertices.append(pos)
        self.normals.append(normal)
        return len(self.vertices) - 1
    def add_face(self, v0, v1, v2):
        self.faces.append((v0, v1, v2))

def voxels_grid_to_compressed_mesh(grid):
    N = grid.VoxelsPerSide()
    max_vertices_num = (N + 1) ** 3
    max_faces_num = (N ** 2) * (N + 1)

    mesh = Mesh()
    vertices_marked = {}
    faces_marked = [
        [False] * max_faces_num,
        [False] * max_faces_num,
        [False] * max_faces_num
    ]

    normals = [
        (0, 0, 1), (0, 1, 0), (1, 0, 0),
        (0, 0, -1), (0, -1, 0), (-1, 0, 0)
    ]

    for x in range(N):
        for y in range(N):
            for z in range(N):
                if not grid.is_occupied(x, y, z):
                    continue
                AddFacesVertexXY(x, y, z, grid, mesh, vertices_marked, faces_marked, normals)
                AddFacesVertexXZ(x, y, z, grid, mesh, vertices_marked, faces_marked, normals)
                AddFacesVertexYZ(x, y, z, grid, mesh, vertices_marked, faces_marked, normals)

    for vertex in mesh.vertices:
        vertex.color = (1.0, 1.0, 1.0, 1.0)
    return mesh

def AddFacesVertexXY(x, y, z, grid, mesh, vertices_marked, faces_marked, normals):
    # The source specifies calling this for both false and true orientations [S1, unofficial]
    # Exact internal logic for orientation handling and face indexing is not detailed in source [S1, unofficial]
    pass

def AddFacesVertexXZ(x, y, z, grid, mesh, vertices_marked, faces_marked, normals):
    # The source specifies calling this for both false and true orientations [S1, unofficial]
    # Exact internal logic for orientation handling and face indexing is not detailed in source [S1, unofficial]
    pass

def AddFacesVertexYZ(x, y, z, grid, mesh, vertices_marked, faces_marked, normals):
    # The source specifies calling this for both false and true orientations [S1, unofficial]
    # Exact internal logic for orientation handling and face indexing is not detailed in source [S1, unofficial]
    pass
```

**Edge Cases and Pitfalls:**
*   **Precision**: The method relies on integer grid coordinates. Ensure grid boundaries are handled correctly to avoid out-of-bounds access when checking neighbors [G].
*   **Redundancy**: The `faces_marked` and `vertices_marked` structures are critical for compression. Correctly updating these ensures the mesh remains compressed [G].

## Subtopics

### [Vosh (Voxel-Mesh Hybrid Representation)](19051-vosh-voxel-mesh-hybrid-representation.md)
Vosh is a hybrid scene representation for NeRF-based real-time view synthesis that combines a sparse voxel grid with a compressed mesh to balance rendering quality and computational speed. It differs from pure voxel methods by extracting a mesh for high-frequency geometry and from pure mesh methods by retaining voxels for volumetric details, using differentiable rendering to optimize the mesh geometry and topology.

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Strengths**: Produces a compact mesh by sharing vertices and faces. Deterministic and simple to implement.
*   **Limitations**: The resulting mesh is blocky and axis-aligned, lacking smooth curvature.
*   **When to use**: When a compact, blocky representation of a voxel grid is needed for rendering or further processing, and smoothness is not required.
*   **When not to use**: When smooth surfaces are required, or when memory constraints prevent pre-allocating the face tracking arrays.

## Background
This method is described in the `cuda-mesh-voxelization` project [S1, unofficial].

## Sources
[S1] bigmat18/cuda-mesh-voxelization, https://github.com/bigmat18/cuda-mesh-voxelization, fetched 2026-10-26 (unofficial)
