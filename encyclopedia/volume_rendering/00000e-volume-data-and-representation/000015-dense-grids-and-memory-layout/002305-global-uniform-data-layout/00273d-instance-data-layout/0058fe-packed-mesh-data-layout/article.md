# Packed mesh data layout
Also called: MeshData layout, Tightly packed GPU buffers for mesh data

## Overview
Packed mesh data layout is a method for storing mesh and square (quad) data in tightly packed GPU buffers to minimize memory bandwidth usage and simplify compute shader access [S1, unofficial]. It builds on [Instance Data Layout](3992-instance-data-layout.md) by organizing instance-specific geometry into fixed-size records and separate contiguous arrays, enabling efficient indirect rendering and frustum culling [S1, unofficial].

## How it works
**Inputs:** A collection of input meshes, each defined by its position, size, and constituent squares (quads).
**Outputs:** Populated GPU buffers: `meshDataBuffer`, `squaresBuffer`, `commandsBuffer`, and `paramsBuffer`.

**Data Structures:**
1.  **MeshData**: A fixed-size structure of 32 bytes per mesh [S1, unofficial].
    *   Fields: Mesh position, size, rectangle indices (offset into `squaresBuffer`), and square count [S1, unofficial].
2.  **Square**: A fixed-size structure of 8 bytes per rectangle [S1, unofficial].
    *   Fields: Indices defining the quad vertices (exact field names not specified in sources, but implied to be vertex indices or similar quad-defining data) [S1, unofficial].

**Parameters:**
*   `quadsInterleaving = 0.05f`: A floating-point constant used to remove small (1 pixel) gaps between triangles during rendering [S1, unofficial].
*   `threadGroupSize = 64`: Used for padding mesh data [S1, unofficial].

**Steps:**
1.  **Iterate and Pack**: Iterate over input meshes. For each mesh, create a `MeshData` struct containing the mesh's position, size, and a reference to its squares. Update the `startSquare` index for the next mesh.
2.  **Pad for Alignment**: Pad the `meshData` array with empty `MeshData` entries until its total size is a multiple of `threadGroupSize` (64) [S1, unofficial].
3.  **Buffer Creation**: Create GPU buffers for `squaresBuffer`, `commandsBuffer`, `meshDataBuffer`, and `paramsBuffer` [S1, unofficial].
4.  **Population**: Populate these buffers with the packed mesh data and square data [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
THREAD_GROUP_SIZE = 64
QUADS_INTERLEAVING = 0.05f

# Data Structures
struct MeshData:
    position: vec3  # Not explicitly detailed, implied by "mesh position"
    size: vec3      # Not explicitly detailed, implied by "size"
    rect_indices: int # Offset into squaresBuffer
    square_count: int

struct Square:
    # 8 bytes per rectangle. Exact fields not specified in sources.
    # Likely vertex indices or quad definition data.
    data: uint32[2] # Placeholder for 8 bytes

def addMeshes(meshes: List[MeshInput]) -> Tuple[List[MeshData], List[Square]]:
    mesh_data_list = []
    squares_list = []
    current_square_index = 0
    
    for mesh in meshes:
        # Extract squares for this mesh
        mesh_squares = extract_squares(mesh)
        squares_list.extend(mesh_squares)
        
        # Create MeshData entry
        md = MeshData(
            position=mesh.position,
            size=mesh.size,
            rect_indices=current_square_index,
            square_count=len(mesh_squares)
        )
        mesh_data_list.append(md)
        
        # Update startSquare for next mesh
        current_square_index += len(mesh_squares)
        
    return mesh_data_list, squares_list

def prepare_render(mesh_data_list: List[MeshData]) -> List[MeshData]:
    # Pad meshData with empty MeshData entries until size is multiple of threadGroupSize
    current_size = len(mesh_data_list)
    padding_needed = (THREAD_GROUP_SIZE - (current_size % THREAD_GROUP_SIZE)) % THREAD_GROUP_SIZE
    
    empty_mesh = MeshData(
        position=vec3(0,0,0),
        size=vec3(0,0,0),
        rect_indices=0,
        square_count=0
    )
    
    for _ in range(padding_needed):
        mesh_data_list.append(empty_mesh)
        
    return mesh_data_list

def create_buffers(mesh_data_list: List[MeshData], squares_list: List[Square]):
    # Create GPU buffers
    mesh_data_buffer = create_gpu_buffer(mesh_data_list)
    squares_buffer = create_gpu_buffer(squares_list)
    commands_buffer = create_gpu_buffer([]) # Populated elsewhere
    params_buffer = create_gpu_buffer([QUADS_INTERLEAVING]) # Example parameter
    
    return mesh_data_buffer, squares_buffer, commands_buffer, params_buffer
```

**Edge Cases and Pitfalls:**
*   **Precision**: The `quadsInterleaving` value is a small float; precision issues may cause gaps or overlaps if not handled correctly in the shader [S1, unofficial].
*   **Boundaries**: Padding ensures alignment but increases memory usage if the number of meshes is not a multiple of 64 [S1, unofficial].
*   **Degenerate Input**: Empty meshes or meshes with zero squares should be handled gracefully, potentially by skipping or creating empty `MeshData` entries [S1, unofficial].

## Subtopics

### Tightly Packed Vertex/Topology Arrays
A data structure for fast display on modern graphics hardware using tightly packed vertex and topology arrays, with auxiliary arrays for extra vertex data [S2, unofficial]. Construction involves a single pass over an octree to generate vertices in a Vertex Array with node pointers populated, followed by multi-resolution contouring. For each generated polygon, it is appended to the end of a Topology Array, and its index is added to the polygon list of the four constituent vertices [S2, unofficial]. The Vertex Array Entry contains data sent to the GPU (position, normal, texcoords), a list of indices into the Topology Array for quads containing this vertex, and a pointer to the generating Node. The Topology Array Entry contains a list of four indices into the Vertex Array for the vertices making up a quadrilateral [S2, unofficial]. This method is chosen when dynamic CSG operations require constant time insertion and deletion of polygons and vertices [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: 32 bytes per mesh + 8 bytes per square [S1, unofficial].
*   **Measured Performance**: Not given in the sources.
*   **Scaling**: Not given in the sources.
*   **Strengths**: Minimizes memory bandwidth, simplifies compute shader access, enables indirect rendering [S1, unofficial].
*   **Limitations**: Fixed-size structures may lead to padding overhead [S1, unofficial].
*   **When to use**: When rendering voxel terrain with many small meshes and squares, requiring efficient GPU access [S1, unofficial].
*   **When not**: When dynamic topology changes are frequent and require complex vertex/quad relationships (use Tightly Packed Vertex/Topology Arrays instead) [S2, unofficial].

## Background
Originated in the context of optimized voxel terrain rendering, specifically in the GLVoxelTerrain project [S1, unofficial]. It is used for indirect rendering with minimal CPU-GPU interactions, frustum culling in compute shaders, and fast greedy mesher applications [S1, unofficial].

## Sources
[S1] Gamengineer314/GLVoxelTerrain, https://github.com/Gamengineer314/GLVoxelTerrain, fetched 2026-10-27 (unofficial)
[S2] PDF Dual Contouring: "The Secret Sauce", https://people.eecs.berkeley.edu/~jrs/meshpapers/SchaeferWarren2.pdf, fetched 2026-10-27 (unofficial)
