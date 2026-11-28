# Boolean Voxel Meshing

Also called: Blocky Mesh, VoxelGrid to Mesh, VoxelGrid To Mesh Hull, VoxelGrid Mesh Hull Generation

## Overview
Boolean Voxel Meshing converts a 3D boolean occupancy grid into an explicit triangular mesh by generating geometry for every occupied voxel [S1, S2, S3, unofficial]. Builds on [Voxel Face and Geometry Generation](11724-voxel-face-and-geometry-generation.md). Unlike [Voxel Face Mesh Generation](4212-voxel-face-mesh-generation.md), which culls internal faces to produce a watertight manifold, this method outputs all 12 triangles for every active voxel, resulting in redundant geometry for internal faces [S1, S3, unofficial]. It is distinguished by its simplicity and lack of adjacency checks, making it suitable for rapid prototyping or when the voxel grid is sparse and internal redundancy is negligible [S1, S7, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D boolean occupancy grid (voxel field) and a scaling factor `dWidth` (or `s`) defining the voxel edge length in world space [S1, S3, unofficial].
*   **Output:** Two arrays: a list of vertex positions (`vecVerts`) and a list of triangle indices (`vecTris`) [S1, unofficial].

**Data Structures:**
*   **Voxel Grid:** Stored as a 3D array of booleans [S3, unofficial]. In C++, this may be a dynamically allocated jagged array `bool *** data` or a flattened 1D array accessed via an index helper `at(i, j, k)` [S1, S3, unofficial].
*   **Vertex Buffer:** `std::vector<std::array<GLfloat, 3>>` or equivalent, storing 3D coordinates [S1, unofficial].
*   **Index Buffer:** `std::vector<std::array<GLuint, 3>>` or equivalent, storing indices into the vertex buffer [S1, unofficial].

**Steps:**
1.  **Initialize Buffers:** Create empty vectors for vertices and triangle indices [S1, unofficial].
2.  **Iterate Grid:** Loop through all voxel coordinates $(i, j, k)$ within the grid dimensions [S1, unofficial].
3.  **Check Occupancy:** If the voxel at $(i, j, k)$ is occupied (true):
    a.  **Generate Vertices:** Compute the 8 corner vertices of the unit cube scaled by `dWidth` [S1, unofficial]. The world coordinates for a corner defined by bit mask $v \in [0, 7]$ are calculated as [S3, unofficial]:
        $$
        \begin{aligned}
        v_x &= i \cdot dWidth + \frac{dWidth}{2} \cdot ((v \ \& \ 4) ? 1 : -1) \\
        v_y &= j \cdot dWidth + \frac{dWidth}{2} \cdot ((v \ \& \ 2) ? 1 : -1) \\
        v_z &= k \cdot dWidth + \frac{dWidth}{2} \cdot ((v \ \& \ 1) ? 1 : -1)
        \end{aligned}
        $$
        *Note: The source [S1, unofficial] uses a simpler offset approach where vertices are at $i \cdot dWidth$ and $(i+1) \cdot dWidth$. The formula in [S3, unofficial] centers the voxel. Implementers must choose one convention and apply it consistently [G]. The following pseudocode uses the [S1, unofficial] convention for direct mapping.*
    b.  **Add Vertices:** Append the 8 vertices to `vecVerts` [S1, unofficial].
    c.  **Generate Indices:** Append 12 triangle indices (forming 6 quads) to `vecTris` using local vertex indices $0..7$ relative to the current voxel's base vertex index in `vecVerts` [S1, unofficial].
        *   Face 1 (Bottom): $(0, 1, 3), (0, 3, 2)$ [G]
        *   Face 2 (Top): $(4, 6, 7), (4, 7, 5)$ [G]
        *   Face 3 (Front): $(0, 4, 5), (0, 5, 1)$ [G]
        *   Face 4 (Back): $(2, 3, 7), (2, 7, 6)$ [G]
        *   Face 5 (Left): $(0, 2, 6), (0, 6, 4)$ [G]
        *   Face 6 (Right): $(1, 5, 7), (1, 7, 3)$ [G]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def boolean_voxel_meshing(voxel_grid, d_width):
    """
    voxel_grid: 3D list or array of booleans, dimensions (nx, ny, nz)
    d_width: float, size of one voxel edge
    Returns: vertices (list of [x,y,z]), indices (list of [i,j,k])
    """
    vertices = []
    indices = []
    
    nx, ny, nz = len(voxel_grid), len(voxel_grid[0]), len(voxel_grid[0][0])
    
    for i in range(nx):
        for j in range(ny):
            for k in range(nz):
                if voxel_grid[i][j][k]:
                    base_idx = len(vertices)
                    
                    # Generate 8 vertices for the cube at (i, j, k)
                    # Using [S1, unofficial] convention: corners at i*d and (i+1)*d
                    x0, x1 = i * d_width, (i + 1) * d_width
                    y0, y1 = j * d_width, (j + 1) * d_width
                    z0, z1 = k * d_width, (k + 1) * d_width
                    
                    # Vertex order:
                    # 0: (x0, y0, z0)
                    # 1: (x1, y0, z0)
                    # 2: (x0, y1, z0)
                    # 3: (x1, y1, z0)
                    # 4: (x0, y0, z1)
                    # 5: (x1, y0, z1)
                    # 6: (x0, y1, z1)
                    # 7: (x1, y1, z1)
                    
                    vertices.append([x0, y0, z0])
                    vertices.append([x1, y0, z0])
                    vertices.append([x0, y1, z0])
                    vertices.append([x1, y1, z0])
                    vertices.append([x0, y0, z1])
                    vertices.append([x1, y0, z1])
                    vertices.append([x0, y1, z1])
                    vertices.append([x1, y1, z1])
                    
                    # Add 12 triangles (6 faces, 2 triangles each)
                    # Face 0 (Z=0, Bottom): 0,1,3 and 0,3,2
                    indices.append([base_idx + 0, base_idx + 1, base_idx + 3])
                    indices.append([base_idx + 0, base_idx + 3, base_idx + 2])
                    
                    # Face 1 (Z=1, Top): 4,6,7 and 4,7,5
                    indices.append([base_idx + 4, base_idx + 6, base_idx + 7])
                    indices.append([base_idx + 4, base_idx + 7, base_idx + 5])
                    
                    # Face 2 (Y=0, Front): 0,4,5 and 0,5,1
                    indices.append([base_idx + 0, base_idx + 4, base_idx + 5])
                    indices.append([base_idx + 0, base_idx + 5, base_idx + 1])
                    
                    # Face 3 (Y=1, Back): 2,3,7 and 2,7,6
                    indices.append([base_idx + 2, base_idx + 3, base_idx + 7])
                    indices.append([base_idx + 2, base_idx + 7, base_idx + 6])
                    
                    # Face 4 (X=0, Left): 0,2,6 and 0,6,4
                    indices.append([base_idx + 0, base_idx + 2, base_idx + 6])
                    indices.append([base_idx + 0, base_idx + 6, base_idx + 4])
                    
                    # Face 5 (X=1, Right): 1,5,7 and 1,7,3
                    indices.append([base_idx + 1, base_idx + 5, base_idx + 7])
                    indices.append([base_idx + 1, base_idx + 7, base_idx + 3])
                    
    return vertices, indices
```

**Parameters:**
*   `dWidth` (or `s`): Scaling factor for voxel size. Typical value depends on scene scale; [S3, unofficial] uses a `desiredStep` of 0.2f for optimal grid dimensions.
*   `Fake Shadow`: A boolean flag (default false) that, when true, applies a fake shadow visualization to the mesh using a fixed light direction `(-0.495633, 0.142501, 0.856762)` [S6, unofficial].

**Edge Cases and Pitfalls:**
*   **Redundant Geometry:** The method draws 12 triangles for every voxel regardless of adjacency. Internal faces between adjacent occupied voxels are duplicated, leading to higher triangle counts and potential z-fighting if not handled by the renderer [S1, unofficial].
*   **T-Junctions:** If adjacent voxels are culled (in variants) or processed independently, unconnected vertices (T-junctions) may cause visible discontinuities. [S7, unofficial] notes that consolidating these vertices is important for closed meshes required by smoothing algorithms.
*   **Non-Watertight:** Because internal faces are not removed, the resulting mesh is not a watertight manifold in the traditional sense of shared edges, though it represents the volume correctly [S1, S3, unofficial].

## Subtopics

### [Voxel Occupancy Boundary Detection](1011-voxel-occupancy-boundary-detection.md)
Identifies voxels on the interface between occupied and unoccupied space, producing a boundary mask used by Boolean Voxel Meshing to generate triangular geometry. It differs from siblings by operating directly on the raw boolean grid to flag boundary voxels via neighborhood checks, whereas [Octree Traversal for Surface Extraction](2113-octree-traversal-for-surface-extraction.md) traverses hierarchical structures, and [VoxelGrid Mesh Generation](2714-voxelgrid-mesh-generation.md) or [Greedy Triangle Meshing](15263-greedy-triangle-meshing.md) focus on immediate mesh assembly rather than explicit boundary flagging. Choose when preprocessing is needed before mesh generation.

### [Octree Traversal for Surface Extraction](2113-octree-traversal-for-surface-extraction.md)
Extracts a triangular mesh from a boolean voxel occupancy grid by recursively traversing an octree structure and generating geometry for leaf nodes. It differs from interpolation-based methods like Marching Cubes by outputting axis-aligned cube meshes directly from octree leaves, resulting in simpler but blocky geometry unless voxel resolution is high. Choose when working with hierarchical or sparse voxel data.

### [VoxelGrid Mesh Generation](2714-voxelgrid-mesh-generation.md)
Constructs a combined triangle mesh from a sparse set of voxels within a `VoxelGrid` by instantiating a base cell geometry for each selected voxel and transforming it into world space. Unlike [Boolean Voxel Meshing](971-boolean-voxel-meshing.md), which detects shared faces at voxel boundaries to produce a watertight manifold, this method treats each voxel as an independent object, resulting in a non-manifold mesh where vertices are not shared between adjacent voxels [G]. Choose when explicit per-voxel transformation is required.

### [Voxel Grid to Triangle Mesh Conversion](6291-voxel-grid-to-triangle-mesh-conversion.md)
Extracts a watertight triangular mesh that represents the exact volume of a boolean voxel occupancy grid. It solves the problem of preserving the discrete geometry of the voxel grid without the interpolation artifacts of marching cubes, ensuring that every edge in the resulting mesh connects to exactly two faces. Choose when volume preservation and watertightness are mandatory.

### [Blocky Surface Extraction](6418-blocky-surface-extraction.md)
Generates a voxel-aligned triangle mesh by iterating over voxels inside the isosurface and adding faces to the exterior boundary where adjacent voxels are outside the shape. Choose when a balance between blocky aesthetics and reduced geometry is needed.

### [VoxelBodyMeshing](6947-voxelbodymeshing.md)
Is a blocky surface extraction method that converts a boolean voxel occupancy grid into a triangular mesh by processing sub-chunks of a `VoxelBody`. It builds on [Boolean Voxel Meshing](971-boolean-voxel-meshing.md) by handling the meshing of individual chunks in a multi-threaded worker loop, supporting both hard-edged (face-merging) and smooth (Marching Cubes) modes. Choose when integrating with `VoxelEngine` architectures or requiring multi-threaded chunk processing.

### [Surface Nets Mesh Generation](11602-surface-nets-mesh-generation.md)
Generates a smooth triangle mesh from a binary density field or Signed Distance Field (SDF) by computing vertex positions at voxel centers based on local gradients, producing significantly fewer vertices and smoother geometry than blocky voxel-based methods. Choose when smooth surfaces are preferred over blocky geometry.

### [Greedy Triangle Meshing](15263-greedy-triangle-meshing.md)
Is a surface extraction algorithm that converts a boolean voxel occupancy grid into an optimized triangular mesh by merging adjacent coplanar faces into larger polygons, which are then triangulated. It solves the problem of high vertex and index counts in naive voxel rendering by reducing the number of draw calls and vertices. Choose when minimizing draw calls and vertex count is a priority.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of occupied voxels. Each occupied voxel generates a constant number of vertices and indices (8 vertices, 12 triangles) [G].
*   **Memory Complexity:** $O(N)$ for vertex and index storage. Specifically, $8N$ vertices and $36N$ indices (if stored as individual triangles) or $12N$ indices (if stored as quads/indices) [G].
*   **Performance:** The method is extremely fast to generate due to the lack of neighbor checks and culling logic [S1, unofficial]. However, the resulting mesh has significantly higher triangle counts than culling-based methods, leading to higher GPU fill-rate usage and larger memory footprint [S1, S7, unofficial].
*   **Strengths:** Simple implementation; no complex data structures or adjacency checks required; suitable for very sparse voxel grids where internal redundancy is low [S1, S7, unofficial].
*   **Limitations:** High redundancy for dense or solid objects; non-watertight manifold (internal faces); potential for z-fighting; T-junctions if not consolidated [S1, S7, unofficial].
*   **When to use:** Rapid prototyping, sparse voxel clouds, or when the renderer can handle redundant geometry efficiently [G].
*   **When not to use:** Dense solid objects, applications requiring watertight meshes for physics or CSG, or performance-critical scenarios with high voxel counts [G].

## Background
Boolean Voxel Meshing is one of the earliest and simplest methods for converting voxel data to polygonal meshes, often associated with the aesthetic of games like Minecraft [S1, S4, unofficial]. The method is used in various open-source projects for OBJ export [S3, unofficial] and WebGL mesh generation [S7, unofficial]. Variants like MicroBlockBuilder extend this concept to allow complex shapes (stairs, slabs) by subdividing voxels into smaller boolean arrays [S5, unofficial].

## Sources
[S1] VBO Generate Verts for Chunks (Voxel Engine, Cpp), https://stackoverflow.com/questions/23427243/vbo-generate-verts-for-chunks-voxel-engine-cpp, fetched 2026-10-26 (unofficial)
[S2] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S3] antonmartynov/OBJ-Solidizer-And-Voxelizer, https://github.com/antonmartynov/OBJ-Solidizer-And-Voxelizer, fetched 2026-10-27 (unofficial)
[S4] lijiemark/floating-islands-generator, https://github.com/lijiemark/floating-islands-generator, fetched 2026-10-27 (unofficial)
[S5] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
[S6] voxeltool, https://github.com/arendvw/voxeltools, fetched 2026-10-29 (unofficial)
[S7] voxel-to-mesh | Converts voxel array to webgl mesh data, https://jzwood.github.io/voxel-to-mesh/, fetched 2026-10-30 (unofficial)
