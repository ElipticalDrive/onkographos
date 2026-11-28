# On-the-fly Shell Geometry Construction

Also called: GPU shell construction, Parallel prism construction, Real-time shell geometry construction

## Overview
On-the-fly shell geometry construction is a method for generating shell geometry (tetrahedrons or prisms) dynamically on the GPU during runtime, enabling shell mapping techniques to be applied to animated surfaces rather than requiring static, offline CPU preprocessing [S1]. It differs from traditional shell mapping by eliminating data-dependent preprocessing steps; instead, it uses a deterministic, data-independent rule based on vertex indices to ensure consistency between adjacent prisms, allowing for full parallelization in compute shaders [S1].

## How it works
The method constructs shell geometry in two distinct stages to optimize memory and time consumption, ensuring that the resulting geometry is consistent across adjacent faces without requiring random assignment or correction of edge patterns [S1].

### Inputs and Outputs
*   **Input:** A base mesh $S$ defined by vertices $V$, vertex normals $N$, and face connectivity (triangles) [S1].
*   **Output:** Three buffers:
    1.  **Vertex Data Buffer:** Contains vertices for the transformed base mesh and the offset surface [S1].
    2.  **Shell Data Buffer:** Contains the constructed tetrahedrons/prisms with intersection data [S1].
    3.  **Index Buffer:** Maps indices from the Shell Data Buffer to the Vertex Data Buffer for vertex shader access [S1].

### Data Structures
The implementation relies on three specific buffers [S1]:
1.  **Vertex Data Buffer:** Stores the positions of the transformed base mesh vertices ($S_t$) and the offset surface vertices ($S'_t$) [S1].
2.  **Shell Data Buffer:** Stores the geometric data for each tetrahedron or prism, including the calculated intersections with tetrahedron planes required for ray tracing [S1].
3.  **Index Buffer:** Stores integer indices pointing to entries in the Vertex Data Buffer [S1]. Each entry in the Shell Data Buffer references these indices to retrieve the correct vertex positions during rendering [S1].

### Algorithm Steps
The construction is performed per face of the base mesh in parallel [S1].

**Step 1: Vertex Transformation**
1.  Transform the base mesh $S$ into $S_t$ using standard vertex shader transformations (model-view-projection) [S1].
2.  Construct an offset surface $S'_t$ by shifting each vertex of $S_t$ along its vertex normal $N$ by a predefined distance $d$ [S1].
3.  Store all resulting vertices ($S_t$ and $S'_t$) into the Vertex Data Buffer [S1].

**Step 2: Shell Construction**
1.  For each face of the base mesh, construct the shell tetrahedrons or prisms [S1].
2.  Determine the direction of the splitting edge for each prism side to preserve consistency between adjacent prisms [S1]. This is done using the **Edge Direction Rule**:
    *   An edge is defined as **'rising'** if it goes from a base mesh vertex with a lower index to a vertex with a higher index, and **'falling'** otherwise [S1].
3.  Use the determined edge directions to split the prism sides consistently [S1].
4.  Calculate intersections with tetrahedron planes, which are required for further ray tracing during rendering [S1].
5.  Store the resulting shell geometry data into the Shell Data Buffer [S1].
6.  Populate the Index Buffer with indices linking the Shell Data Buffer entries to the corresponding vertices in the Vertex Data Buffer [S1].

### Pseudocode
The following pseudocode illustrates the parallel construction process per face [S1]. It assumes a compute shader environment where each work group or thread handles one or more faces [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
OFFSET_DISTANCE = 0.01  # Predefined distance for offset surface [S1]

# Input: Base mesh data (vertices, normals, faces)
# Output: VertexDataBuffer, ShellDataBuffer, IndexBuffer

def construct_shell_geometry(base_mesh):
    # Stage 1: Vertex Transformation
    # Parallelized over all vertices [S1]
    transformed_vertices = []
    offset_vertices = []
    
    for vertex in base_mesh.vertices:
        # Transform base vertex (e.g., model-view-projection) [S1]
        st_pos = transform_vertex(vertex.position)
        
        # Calculate offset vertex
        # Shift along vertex normal by predefined distance [S1]
        st_normal = transform_normal(vertex.normal)
        st_normal = normalize(st_normal)
        s_prime_t_pos = st_pos + (st_normal * OFFSET_DISTANCE)
        
        transformed_vertices.append(st_pos)
        offset_vertices.append(s_prime_t_pos)
        
    # Store in Vertex Data Buffer [S1]
    vertex_data_buffer = list(transformed_vertices + offset_vertices)
    
    # Stage 2: Shell Construction
    # Parallelized over all faces [S1]
    shell_data = []
    index_data = []
    
    for face in base_mesh.faces:
        v0, v1, v2 = face.vertices
        idx0, idx1, idx2 = face.vertex_indices
        
        # Get transformed positions for base vertices
        p0 = transformed_vertices[idx0]
        p1 = transformed_vertices[idx1]
        p2 = transformed_vertices[idx2]
        
        # Get offset positions for base vertices
        o0 = offset_vertices[idx0]
        o1 = offset_vertices[idx1]
        o2 = offset_vertices[idx2]
        
        # Determine edge directions for consistency [S1]
        # Edge is 'rising' if index goes from lower to higher
        # Edge is 'falling' if index goes from higher to lower [S1]
        
        # Example: Constructing a prism/tetrahedron structure
        # The specific geometric decomposition depends on the shell mapping variant,
        # but the splitting edge direction is determined by the index rule.
        
        # Determine direction of edge (v0, v1)
        if idx0 < idx1:
            edge_01_direction = 'rising'
        else:
            edge_01_direction = 'falling'
            
        # Determine direction of edge (v1, v2)
        if idx1 < idx2:
            edge_12_direction = 'rising'
        else:
            edge_12_direction = 'falling'
            
        # Determine direction of edge (v2, v0)
        if idx2 < idx0:
            edge_20_direction = 'rising'
        else:
            edge_20_direction = 'falling'
            
        # Construct tetrahedrons/prisms based on these directions
        # This ensures adjacent prisms share consistent splitting edges
        # The exact geometric construction of the tetrahedrons from these vertices
        # and directions follows the standard shell mapping decomposition,
        # adapted to use the determined edge directions.
        
        # Construct shell geometry data (positions, normals, intersections)
        # Intersections with tetrahedron planes are calculated here [S1]
        shell_entry = construct_tetrahedron(p0, p1, p2, o0, o1, o2, 
                                            edge_01_direction, edge_12_direction, edge_20_direction)
        
        shell_data.append(shell_entry)
        
        # Create index mapping
        # Indices into vertex_data_buffer for the vertices of this shell entry
        # The exact indices depend on the layout of vertex_data_buffer
        # Assuming transformed vertices are first, then offset vertices
        base_indices = [idx0, idx1, idx2]
        offset_indices = [idx0 + len(base_mesh.vertices), 
                          idx1 + len(base_mesh.vertices), 
                          idx2 + len(base_mesh.vertices)]
        
        # Store indices for this shell entry [S1]
        index_data.append(base_indices + offset_indices)
        
    # Store results [S1]
    shell_data_buffer = list(shell_data)
    index_buffer = list(index_data)
    
    return vertex_data_buffer, shell_data_buffer, index_buffer

def construct_tetrahedron(p0, p1, p2, o0, o1, o2, dir_01, dir_12, dir_20):
    """
    Constructs the shell geometry (tetrahedrons/prisms) for a face.
    The specific decomposition depends on the shell mapping algorithm,
    but uses the edge directions to ensure consistency.
    """
    # The exact geometric decomposition of the tetrahedrons/prisms from the six vertices
    # and edge directions follows standard shell mapping [S1].
    # This part is implementation-specific to the shell mapping variant.
    # The key is that the splitting edge is determined by the index rule.
    return {
        'base': [p0, p1, p2],
        'offset': [o0, o1, o2],
        'edge_directions': [dir_01, dir_12, dir_20]
    }

def transform_vertex(v):
    # Applies standard model-view-projection transformation as in a vertex shader [S1]
    return v

def transform_normal(n):
    # Applies inverse-transpose normal transformation as in a vertex shader [S1]
    return n
```

### Parameters
*   **Offset Distance ($d$):** The predefined distance by which vertices are shifted along their normals to create the offset surface $S'_t$ [S1]. The value is not specified in the sources [S1].
*   **Number of Depth Peeling Passes:** Empirically, 3-4 passes are enough to produce good quality results for order-independent transparency [S1].

### Edge Cases and Pitfalls
*   **Animated Surfaces:** The method is specifically designed to handle animated surfaces, avoiding the static limitation of offline preprocessing [S1].
*   **Consistency:** The edge direction rule guarantees consistency between adjacent prisms, preventing gaps or overlaps that could occur with random edge pattern assignment [S1].
*   **Rasterization Overhead:** All tetrahedrons must be rasterized each pass, even with culling [S1]. The vertex processing and rasterization work occur before the z-test, which can be a performance bottleneck [S1].
*   **Mip-map Construction:** Simple mip-map construction with linear downscaling does not produce correct results for 3D textures, especially at very low scales [S1].

## Operating characteristics
*   **Time Complexity:** The method is data-independent and parallelizable, allowing tetrahedrons to be constructed in parallel for each face of the base mesh [S1]. The process is split into two stages to optimize time consumption [S1].
*   **Memory Complexity:** The method uses three buffers (Vertex Data, Shell Data, Index Buffer) to store the geometry and connectivity [S1]. This structure is designed to reduce memory usage compared to storing full geometry data redundantly [S1].
*   **Performance:** The method was tested on multiple scenes against shell mapping techniques, showing applicability for real-time rendering [S1]. Performance gains are expected with advanced lighting calculations due to reduced overdraw [S1].
*   **Strengths:**
    *   Enables real-time rendering of animated surfaces with small-scale volumetric structures [S1].
    *   Data-independent and parallelizable, suitable for GPU compute shaders [S1].
    *   Ensures consistency between adjacent prisms without complex preprocessing [S1].
*   **Limitations:**
    *   Requires rasterization of all tetrahedrons each pass, which can be inefficient [S1].
    *   Simple mip-map construction is not suitable for 3D textures in this context [S1].
    *   May benefit from hardware ray tracing in the future, as current rasterization-based approach has overhead [S1].
*   **When to Use:** Suitable for real-time rendering of animated surfaces with small-scale volumetric features (e.g., knitwear, fur, grass, forests) where offline preprocessing is not feasible [S1].
*   **When Not to Use:** Not suitable for static surfaces where offline preprocessing is acceptable and may offer better optimization opportunities [S1]. Also, simple mip-map techniques are not appropriate for 3D textures in this context [S1].

## Background
The method was introduced in the paper "Real-time Rendering of Small-scale Volumetric Structure on Animated Surfaces" (2021) [S1]. It addresses the limitation of previous shell mapping techniques, which required offline CPU preprocessing and static surfaces [S1]. Previous methods assigned edge patterns randomly and corrected inconsistencies, making the process data-dependent and non-parallelizable [S1]. This method simplifies the prism pattern defining process by using vertex indices to determine edge directions, enabling on-the-fly construction on the GPU [S1]. It is part of the broader field of real-time rendering, shell mapping, and levels of detail [S1].

## Sources
[S1] Real-time Rendering of Small-scale Volumetric Structure on Animated Surfaces, https://doi.org/10.20948/graphicon-2021-3027-64-73, fetched 2026-10-26
