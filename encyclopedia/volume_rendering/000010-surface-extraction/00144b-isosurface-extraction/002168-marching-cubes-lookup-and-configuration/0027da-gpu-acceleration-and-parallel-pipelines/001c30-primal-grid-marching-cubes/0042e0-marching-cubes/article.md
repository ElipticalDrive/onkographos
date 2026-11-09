# Marching Cubes

Also called: Lorensen and Cline Marching Cubes, Lorensen and Cline algorithm, MC algorithm, Polygonising a Scalar Field, isosurface reconstruction, マーチングキューブ法

## Overview

Marching Cubes is a computer graphics algorithm published in 1987 by Lorensen and Cline for extracting a polygonal mesh (specifically a triangular mesh) of an isosurface from a three-dimensional discrete scalar field [S6]. The algorithm solves the problem of visualizing constant density surfaces from discrete 3D array data by dividing the volume into cubic cells and approximating the surface within each cell [S6][S115, unofficial]. It sets itself apart from its siblings by using a lookup table based on the 256 possible binary configurations of a cube's 8 vertices to determine triangulation, whereas [Dual Marching Cubes](906-dual-marching-cubes.md) operates on the dual grid and [GPU Marching Cubes with Lookup Tables](3690-gpu-marching-cubes-with-lookup-tables.md) focuses on parallel implementation details. Unlike [Marching Squares](918-marching-squares.md), which is the 2D analog, Marching Cubes operates in 3D and suffers from topological ambiguities that the 2D version does not [S1, unofficial][S6].

## How it works

**Inputs and Outputs:**
*   **Input:** A 3D scalar field $f$ sampled on a regular Cartesian grid. The grid has dimensions $N_x \times N_y \times N_z$ vertices. Each vertex $v_{i,j,k}$ has an associated scalar value $f(v_{i,j,k})$. An isovalue $\alpha$ (threshold) is specified [S5][S115, unofficial].
*   **Output:** A triangular mesh consisting of a list of vertices and a list of triangle indices. The vertices are located at the intersections of the isosurface with the grid edges [S5, S6].

**Data Structures:**
1.  **Grid:** A 3D array of scalar values.
2.  **Edge Table (`edgeTable`):** An array of 256 integers (32-bit). Each integer is a bitmask indicating which of the 12 edges of a cube are intersected by the isosurface. If bit $e$ is set, edge $e$ is intersected [S6][S97, unofficial].
3.  **Triangulation Table (`triTable`):** An array of $256 \times 16$ integers. For each of the 256 cases, it lists the indices of the edges that form the triangles. The data is structured as a sequence of triplets (each triplet defines one triangle), terminated by a sentinel value (typically -1) [S6][S97, unofficial].
4.  **Vertex Sharing Index (Optional but recommended for manifold output):** A 4D array `vertexSharingIndexes` of size $[N_x, N_y, N_z, 3]$ initialized to -1. This tracks which vertex index (0-2) of a cube corner is currently owned by a specific voxel position to avoid duplicate vertices at shared corners [S88, unofficial].

**Vertex and Edge Indexing:**
The 8 vertices of a cube at grid coordinates $(x, y, z)$ are indexed $0$ to $7$. The standard indexing (Lorensen and Cline) is:
*   0: $(x, y, z)$
*   1: $(x+1, y, z)$
*   2: $(x+1, y+1, z)$
*   3: $(x, y+1, z)$
*   4: $(x, y, z+1)$
*   5: $(x+1, y, z+1)$
*   6: $(x+1, y+1, z+1)$
*   7: $(x, y+1, z+1)$ [S5, S6]

The 12 edges are indexed $0$ to $11$. The mapping from edge index to the two vertex indices it connects is fixed. For example, edge 0 connects vertices 0 and 1; edge 1 connects 1 and 2; edge 2 connects 2 and 3; edge 3 connects 3 and 0; edge 4 connects 4 and 5; edge 5 connects 5 and 6; edge 6 connects 6 and 7; edge 7 connects 7 and 4; edge 8 connects 0 and 4; edge 9 connects 1 and 5; edge 10 connects 2 and 6; edge 11 connects 3 and 7 [S6][S97, unofficial].

**Algorithm Steps:**

1.  **Iterate over Cubes:** Loop through all cubes in the grid. A cube is defined by its minimum corner $(x, y, z)$ where $0 \le x < N_x-1$, $0 \le y < N_y-1$, $0 \le z < N_z-1$ [S6][S111, unofficial].
2.  **Sample Vertices:** Retrieve the scalar values $v_0, \dots, v_7$ at the 8 corners of the current cube [S48, unofficial].
3.  **Compute Cube Index:** Create an 8-bit integer `cubeIndex` by checking if each vertex value is below the isovalue $\alpha$. If $v_i < \alpha$, set bit $i$ in `cubeIndex` [S39, S58, unofficial].
    $$ \text{cubeIndex} = \sum_{i=0}^{7} \begin{cases} 2^i & \text{if } v_i < \alpha \\ 0 & \text{otherwise} \end{cases} $$
    *Note: Some implementations use $v_i \ge \alpha$ to set the bit. The lookup tables must be consistent with this choice. The Lorensen and Cline original table assumes $v_i < \alpha$ sets the bit [S58, S61, unofficial].*
4.  **Check for Empty/Full Cube:** If `cubeIndex` is 0 or 255, the cube is entirely inside or outside the surface. Skip to the next cube [S29, S30, unofficial].
5.  **Lookup Edge Intersections:** Use `cubeIndex` to index into `edgeTable`. The result is a bitmask `edgeMask` indicating which of the 12 edges are intersected [S97, unofficial].
6.  **Find Intersection Points:** For each edge $e$ where bit $e$ is set in `edgeMask`:
    *   Identify the two vertices $A$ and $B$ connected by edge $e$.
    *   Let $val_A$ and $val_B$ be their scalar values.
    *   Compute the intersection point $P$ using linear interpolation:
        $$ P = A + \frac{\alpha - val_A}{val_B - val_A} (B - A) $$
        *G: If $val_A == val_B$, the denominator is zero. This is a degenerate case. A common guard is to treat the intersection as the midpoint or skip, though strictly speaking, the isosurface passes through the vertex if $val_A = \alpha$ [G].*
    *   Store $P$ in a temporary array `intersectPoints[edgeIndex]`.
7.  **Lookup Triangulation:** Use `cubeIndex` to index into `triTable`. Read the sequence of edge indices until the sentinel (-1) is reached. The data is read in groups of 3 to form triangles [S31, S97, unofficial].
8.  **Generate Vertices and Triangles:**
    *   For each triangle defined by edge indices $(e_1, e_2, e_3)$:
        *   Retrieve the intersection points $P_1, P_2, P_3$ from `intersectPoints`.
        *   To ensure a manifold mesh (no duplicate vertices), use the `vertexSharingIndexes` array. For each point $P_k$ on an edge shared by multiple cubes, check if a vertex index has already been created for this geometric location. If not, add $P_k$ to the global vertex list and store its index in `vertexSharingIndexes`. If yes, reuse the existing index [S88, unofficial].
        *   Add a triangle to the output mesh using the three vertex indices [S64, S88, unofficial].
9.  **Compute Normals (Optional):** The normal at an intersection point can be estimated by interpolating the gradients of the scalar field at the two vertices of the intersected edge. The gradient $\nabla f$ at a vertex can be approximated by finite differences [S6][S108, book].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes(grid, isovalue, grid_spacing, grid_origin):
    """
    grid: 3D numpy array of shape (Nx, Ny, Nz)
    isovalue: float
    grid_spacing: tuple (dx, dy, dz)
    grid_origin: tuple (ox, oy, oz)
    Returns: vertices (N, 3), triangles (M, 3)
    """
    Nx, Ny, Nz = grid.shape
    # Pre-defined edgeTable and triTable from Lorensen and Cline [S6][S97, unofficial]
    # edgeTable: list of 256 ints
    # triTable: list of 256 lists of ints (terminated by -1)
    
    vertices = []
    triangles = []
    
    # Vertex sharing cache: [x, y, z, vertex_corner_index] -> global_vertex_index
    # vertex_corner_index 0-7 corresponds to the 8 corners of the cube at (x,y,z)
    # However, standard implementation caches by edge intersection or uses a simpler 
    # corner-sharing array as described in [S88, unofficial]. Here we use a simplified 
    # corner-sharing approach for manifold output.
    # shared_vertices[x, y, z, corner] stores the global index of the vertex 
    # at grid corner (x,y,z) if it has been added.
    # We need to handle the fact that a vertex is shared by 8 cubes.
    # A robust way is to store vertices by their grid coordinate.
    # Let's use a dictionary or a padded array.
    # For simplicity in pseudocode, we assume a function get_vertex_index(x,y,z)
    # that returns the global index of the grid vertex (x,y,z), creating it if needed.
    
    # Note: The standard MC algorithm places vertices on EDGES, not grid corners.
    # To share vertices, we must share by the geometric point on the edge.
    # Since edges are shared between adjacent cubes, we can cache by edge.
    # Edge (x,y,z, edge_id) is shared.
    
    # Simplified: We will just generate vertices and triangles. 
    # For a true manifold, one must implement the vertex sharing logic 
    # described in [S88, unofficial] or similar.
    
    for x in range(Nx - 1):
        for y in range(Ny - 1):
            for z in range(Nz - 1):
                # 1. Sample 8 vertices
                v = [
                    grid[x, y, z],
                    grid[x+1, y, z],
                    grid[x+1, y+1, z],
                    grid[x, y+1, z],
                    grid[x, y, z+1],
                    grid[x+1, y, z+1],
                    grid[x+1, y+1, z+1],
                    grid[x, y+1, z+1]
                ]
                
                # 2. Compute cube index
                cube_index = 0
                for i in range(8):
                    if v[i] < isovalue:
                        cube_index |= (1 << i)
                
                # 3. Check empty/full
                if cube_index == 0 or cube_index == 255:
                    continue
                
                # 4. Get edge mask
                edge_mask = edgeTable[cube_index]
                
                # 5. Find intersections
                # Map edge index to (v1_idx, v2_idx)
                edge_connections = [
                    (0, 1), (1, 2), (2, 3), (3, 0), # Bottom face
                    (4, 5), (5, 6), (6, 7), (7, 4), # Top face
                    (0, 4), (1, 5), (2, 6), (3, 7)  # Vertical edges
                ]
                
                intersections = {}
                for edge_idx in range(12):
                    if (edge_mask >> edge_idx) & 1:
                        v1_idx, v2_idx = edge_connections[edge_idx]
                        val1 = v[v1_idx]
                        val2 = v[v2_idx]
                        
                        # Linear interpolation
                        # t = (isovalue - val1) / (val2 - val1)
                        # Avoid division by zero
                        if abs(val2 - val1) < 1e-9:
                            t = 0.5
                        else:
                            t = (isovalue - val1) / (val2 - val1)
                        
                        # Get grid coordinates of the two vertices
                        # Vertex indices 0-7 map to (dx, dy, dz) offsets
                        offsets = [
                            (0, 0, 0), (1, 0, 0), (1, 1, 0), (0, 1, 0),
                            (0, 0, 1), (1, 0, 1), (1, 1, 1), (0, 1, 1)
                        ]
                        
                        p1 = grid_origin + (x + offsets[v1_idx][0])*dx, (y + offsets[v1_idx][1])*dy, (z + offsets[v1_idx][2])*dz
                        p2 = grid_origin + (x + offsets[v2_idx][0])*dx, (y + offsets[v2_idx][1])*dy, (z + offsets[v2_idx][2])*dz
                        
                        # Interpolate position
                        ix = p1[0] + t * (p2[0] - p1[0])
                        iy = p1[1] + t * (p2[1] - p1[1])
                        iz = p1[2] + t * (p2[2] - p1[2])
                        
                        intersections[edge_idx] = (ix, iy, iz)
                
                # 6. Lookup triangles
                tri_data = triTable[cube_index]
                i = 0
                while i < len(tri_data) and tri_data[i] != -1:
                    e1 = tri_data[i]
                    e2 = tri_data[i+1]
                    e3 = tri_data[i+2]
                    
                    p1 = intersections[e1]
                    p2 = intersections[e2]
                    p3 = intersections[e3]
                    
                    # Add vertices and triangle
                    # In a real implementation, use shared vertex indices here
                    vertices.append(p1)
                    vertices.append(p2)
                    vertices.append(p3)
                    
                    base_idx = len(vertices) - 3
                    triangles.append((base_idx, base_idx + 1, base_idx + 2))
                    
                    i += 3
                    
    return vertices, triangles
```

**Edge Cases and Pitfalls:**
*   **Topological Ambiguities:** The original 256-case table does not resolve face ambiguities (where all 4 edges of a face are intersected and diagonals have opposite signs) or internal ambiguities (case 4). This can lead to non-manifold geometry or incorrect topology (e.g., tunnels becoming solid) [S6, S3]. Variants like [Marching Cubes 2](1393-marching-cubes-2.md) or [Topological Correctness Guarantees](1796-topological-correctness-guarantees.md) address this.
*   **Degenerate Cases:** If a vertex value equals the isovalue exactly, the interpolation denominator is zero. This requires a guard [G].
*   **Boundary Conditions:** The algorithm iterates up to $N-1$. Vertices on the boundary of the volume are not processed as cube corners for subsequent cubes. Padding the volume with a constant value (e.g., -1e6) is often used to handle boundaries correctly [S46, S105, unofficial].

## Subtopics

### [Marching Squares](918-marching-squares.md)
Marching Squares is a 2D contouring algorithm that extracts line segments approximating the zero-level set (or any isovalue $L$) of a scalar field $f$ within a uniform grid of square cells. It determines the topology of the isosurface intersection with each cell by classifying the scalar values at the four corners relative to the isovalue, then interpolating intersection points on the edges. Unlike its 3D counterpart, Marching Cubes, Marching Squares does not suffer from topological ambiguities (such as the saddle point ambiguity) because the intersection of a line with a convex quadrilateral is always unique given the corner signs. It serves as the foundational 2D case for the [Marching Cubes](774-marching-cubes.md) algorithm.

### [Cubical Marching Squares Algorithm](941-cubical-marching-squares-algorithm.md)
The Cubical Marching Squares Algorithm (CMS) is a surface extraction method for 3D scalar fields that preserves sharp features and maintains consistent topology without generating cracks between adjacent cells. Unlike standard Marching Cubes, which may produce topological errors or lose sharp features, CMS uses Hermite data and resolves cell ambiguities by analyzing 2D features on cell faces and 3D features within cells. It eliminates inter-cell dependency, making it suitable for parallel implementation, and generates adaptive surfaces without the need for post-processing crack patching.

### [Marching Cubes 2](1393-marching-cubes-2.md)
Marching Cubes 2 is a variant of the Marching Cubes algorithm that utilizes a different lookup table or implementation strategy for isosurface extraction. It serves as a direct replacement for the standard Marching Cubes core logic, maintaining the same input/output interface and voxel-wise processing paradigm while altering the internal topology generation or vertex placement logic. It is distinct from [Marching Cubes 3](1394-marching-cubes-3.md) and [Extended Marching Cubes](1630-extended-marching-cubes.md) in its specific approach to handling cube configurations, often focusing on robustness or specific performance optimizations rather than topological correctness proofs or degenerate triangle elimination via 3-label systems.

### [Marching Cubes 3](1394-marching-cubes-3.md)
Marching Cubes 3 is a specific implementation variant of the Marching Cubes algorithm for extracting polygonal meshes from 3D scalar fields. It builds on the standard 256-entry lookup table topology but is distinguished by its explicit C++/Python binding interface that passes array bounds, value accessors, and output containers directly. Unlike generic wrappers, it exposes the core extraction loop through a typed function signature expecting `double` precision and explicit dimension counts.

### [Extended Marching Cubes](1630-extended-marching-cubes.md)
Extended Marching Cubes (EMC) is a surface extraction method that enhances the classic Marching Cubes algorithm by improving the reproduction of sharp edges and corners, even when they are not aligned with the grid. It achieves this by modifying the field function to return a 3D distance vector for exact intersection computation and employing Quadric Error Functions (QEF) to reconstruct sharp features detected via normal vector analysis. Unlike standard Marching Cubes, which produces smooth approximations, EMC explicitly places new vertices at feature locations to preserve geometric detail.

### [Marching Tetrahedra](1733-marching-tetrahedra.md)
Marching Tetrahedra is an isosurface extraction algorithm that decomposes each cubic voxel into tetrahedra to resolve the topological ambiguities inherent in [Marching Cubes](774-marching-cubes.md). By processing each tetrahedron independently, it guarantees a unique, unambiguous intersection for every configuration, producing a crack-free mesh without the need for special case handling.

### [Topological Correctness Guarantees](1796-topological-correctness-guarantees.md)
Topological Correctness Guarantees constitute a family of algorithms designed to resolve the topological ambiguities inherent in standard Marching Cubes [774-marching-cubes.md]. Standard Marching Cubes relies on linear interpolation along cell edges, which can produce incorrect connectivity (e.g., disconnected sheets instead of a tunnel) in ambiguous cases where the isosurface passes through a saddle point. These methods ensure the extracted mesh is homeomorphic to the underlying interpolant (trilinear or bilinear) or strictly adheres to discrete connectivity rules, producing a valid 2-manifold. They differ from siblings like [Marching Squares](918-marching-squares.md) by operating in 3D and from [Marching Cubes 2](1393-marching-cubes-2.md) by focusing specifically on the topological validity of the output rather than just geometric accuracy or case expansion.

### [Node Mesh Generation](2112-node-mesh-generation.md)
Node Mesh Generation constructs the explicit geometric vertex and face data for a voxel grid by defining the 8 corner vertices and 12 triangular faces of each voxel cube. It differs from topological extraction methods like [Marching Cubes](774-marching-cubes.md) by generating a full volumetric mesh regardless of scalar field content, primarily used as a base for ray-triangle intersection tests or LOD transitions.

### [Isosurface Extraction with Topological Optimization](2132-isosurface-extraction-with-topological-optimization.md)
Isosurface Extraction with Topological Optimization is a global strategy for extracting a polygonal mesh from a 3D scalar field that optimizes topological and combinatorial measures—specifically triangle count, genus, and number of shells—by resolving topological ambiguities globally rather than locally. Unlike standard Marching Cubes, which uses local lookup tables to resolve ambiguous configurations, this method identifies "X-faces" and "X-cubes" and uses auxiliary data structures (X-face graph, Merge Tree) to make consistent decisions that minimize global complexity.

### [Voxel Chunk Meshing with Culling](2523-voxel-chunk-meshing-with-culling.md)
Voxel Chunk Meshing with Culling is a surface extraction technique that generates a triangle mesh from a 3D voxel grid by iterating through every voxel and emitting geometry only for faces adjacent to empty space (air) or the chunk boundary. It solves the problem of excessive geometry and draw calls inherent in naive per-voxel meshing by culling internal faces shared between adjacent solid voxels. Unlike [Greedy Meshing](3670-gpu-voxel-mesh-generation.md), which merges adjacent coplanar faces into larger rectangles to reduce triangle count, this method generates exactly one quad (two triangles) per exposed face, resulting in a simpler, faster algorithm that produces a larger number of triangles. It serves as the "oracle" or ground-truth reference for verifying the correctness of more complex merging algorithms.

## Operating characteristics

**Time Complexity:** $O(N)$, where $N$ is the number of voxels (cubes) in the grid. Each cube is processed independently in constant time (lookup table access and interpolation) [S3, S7].
**Memory Complexity:** $O(N)$ for the input grid. The output mesh size is proportional to the surface area, which is typically $O(N^{2/3})$ for a solid object, but can be $O(N)$ for complex, fractal-like isosurfaces [S1, unofficial].
**Measured Performance:** The algorithm is highly parallelizable because the processing of each cube is independent of its neighbors [S41, unofficial]. This makes it suitable for GPU acceleration [S94, unofficial]. However, the number of triangles can become very large for high-resolution data, potentially exceeding one billion triangles for large datasets, which is challenging for modern GPUs [S1, unofficial].
**Strengths:** Simple to implement, fast, produces a smooth approximation of the isosurface, widely supported in graphics libraries [S6][S113, unofficial].
**Limitations:** The original algorithm produces non-manifold geometry in ambiguous cases (face and internal ambiguities) [S6]. It may produce "cracks" or topological errors if not handled by variants [S2, unofficial][S6]. The output mesh can be dense and noisy if the scalar field is not smooth [S98, unofficial].

## Background

The algorithm was developed by William E. Lorensen and Harvey E. Cline at General Electric in the mid-1980s as part of their research for visualizing data from CT and MRI devices [S6]. It was published in the 1987 SIGGRAPH proceedings [S6]. The algorithm was patented as United States Patent 4,710,876, which expired in 2005 [S6]. It is considered the most popular indirect volume rendering approach [S13, S23, unofficial]. The algorithm was extended in many ways to address topological ambiguities and improve performance [S1, unofficial].

## Sources

[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] An Overview of Rendering from Volume Data --- including Surface and Volume Rendering, https://core.ac.uk/download/63614.pdf, fetched 2026-10-30 (unofficial)
[S3] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S4] (No) More Marching Cubes, https://doi.org/10.2312/vg/vg07/081-090, fetched 2026-10-26
[S5] A Evolucao do Algoritmo Marching Cubes, https://doi.org/10.5540/03.2017.005.01.0107, fetched 2026-10-26
[S6] Marching Cubes algorithm, https://en.wikipedia.org/wiki/Marching_cubes, fetched 2026-10-26
[S7] Parallel Isosurface Extraction for 3D Data Analysis Workflows in Distributed Environments, https://doi.org/10.1002/cpe.1710, fetched 2026-10-26
[S8] Unity Marching Cubes triangulation bug, https://gamedev.stackexchange.com/questions/142332/unity-marching-cubes-triangulation-bug, fetched 2026-10-26 (unofficial)
[S9] Make yourself a Voxel Engine THIS weekend, https://daymare.net/blogs/voxel-engine-in-a-weekend/, fetched 2026-10-26 (unofficial)
[S10] Adaptation of Dual Contouring and Marching Cubes Algorithms for 3D Reconstruction of the Human Lumbar Spine, https://doi.org/10.35596/1729-7648-2023-21-6-99-105, fetched 2026-10-27
[S11] DTIC ADA582182: Isosurface Display of 3-D Scalar Fields from a Meteorological Model on Google Earth, https://archive.org/details/DTIC_ADA582182, fetched 2026-10-26 (unofficial)
[S12] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S13] Interactive Volume-Rendering Techniques for Medical Data Visualization, https://doi.org/10.2312/diss.20018138, fetched 2026-10-26 (unofficial)
[S14] Volume rendering techniques for general purpose graphics hardware, https://opus4.kobv.de/opus4-fau/files/9/Dissertation.pdf, fetched 2026-10-26 (unofficial)
[S15] Volume-Enclosing Surface Extraction, https://arxiv.org/abs/1011.1787v4, fetched 2026-10-26 (unofficial)
[S16] Verifiable Visualization for Isosurface Extraction, https://doi.org/10.1109/tvcg.2009.194, fetched 2026-10-26
[S17] A Case Study of Isosurface Extraction Algorithm Performance, https://doi.org/10.1007/978-3-7091-6783-0_26, fetched 2026-10-26
[S18] Isosurface, https://en.wikipedia.org/wiki/Isosurface, fetched 2026-10-26
[S19] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
[S20] Volumetric Terrain Rendering with WebGL, https://doi.org/10.2352/issn.2470-1173.2018.06.mobmu-136, fetched 2026-10-26
[S21] jaanonim/volume-rendering, https://github.com/jaanonim/volume-rendering, fetched 2026-10-26 (unofficial)
[S22] DTIC ADA202661: Volumetric Rendering Techniques for the Display of Three-Dimensional Aerodynamic Flow Field Data, https://archive.org/details/DTIC_ADA202661, fetched 2026-10-26 (unofficial)
[S23] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S24] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S25] Visualization Lecture 09: VolVis Part 3-Direct Volume Rendering, Ray and Transfer Functions, https://www.youtube.com/watch?v=FYAJQgYdeqQ, fetched 2026-10-27 (unofficial)
[S26] pmneila/PyMCubes, https://github.com/pmneila/PyMCubes, fetched 2026-10-26 (unofficial)
[S27] Scrawk/Marching-Cubes, https://github.com/Scrawk/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S28] BorisTheBrave/mc-dc, https://github.com/BorisTheBrave/mc-dc, fetched 2026-10-26 (unofficial)
[S29] tatsy/torchmcubes, https://github.com/tatsy/torchmcubes, fetched 2026-10-26 (unofficial)
[S30] Magnus2/MeshReconstruction, https://github.com/Magnus2/MeshReconstruction, fetched 2026-10-26 (unofficial)
[S31] tdhooper/glsl-marching-cubes, https://github.com/tdhooper/glsl-marching-cubes, fetched 2026-10-26 (unofficial)
[S32] millerc3/Marching-Cubes, https://github.com/millerc3/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S33] SebLague/Godot-Marching-Cubes, https://github.com/SebLague/Godot-Marching-Cubes, fetched 2026-10-26 (unofficial)
[S34] ilastik/marching_cubes, https://github.com/ilastik/marching_cubes, fetched 2026-10-26 (unofficial)
[S35] Twinklebear/webgl-marching-cubes, https://github.com/Twinklebear/webgl-marching-cubes, fetched 2026-10-26 (unofficial)
[S36] CHINA-JIGE/MarchingCube, https://github.com/CHINA-JIGE/MarchingCube, fetched 2026-10-26 (unofficial)
[S37] lzhnb/CuMCubes, https://github.com/lzhnb/CuMCubes, fetched 2026-10-26 (unofficial)
[S38] nsf/mc, https://github.com/nsf/mc, fetched 2026-10-26 (unofficial)
[S39] nihaljn/marching-cubes, https://github.com/nihaljn/marching-cubes, fetched 2026-10-26 (unofficial)
[S40] aparis69/MarchingCubeCpp, https://github.com/aparis69/MarchingCubeCpp, fetched 2026-10-26 (unofficial)
[S41] mwalczyk/marching-cubes, https://github.com/mwalczyk/marching-cubes, fetched 2026-10-26 (unofficial)
[S42] ForeverDavid/marching-cubes-cuda, https://github.com/ForeverDavid/marching-cubes-cuda, fetched 2026-10-26 (unofficial)
[S43] mutantbob/blender-marching-cubes, https://github.com/mutantbob/blender-marching-cubes, fetched 2026-10-26 (unofficial)
[S44] Add foundational voxel engine: generic, terrain, and smooth voxel support, https://github.com/Five-Squared-Interactive/StraightFour/pull/40, fetched 2026-10-26 (unofficial)
[S45] A Novel 3D Reconstruction Algorithm of CT Images Based on Improved Marching Cubes Algorithm, https://doi.org/10.4108/eai.27-8-2020.2295136, fetched 2026-10-26
[S46] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S47] A parallel isosurface extraction component for visualization pipelines executing on GPU clusters, https://doi.org/10.1016/j.cam.2014.05.019, fetched 2026-10-27
[S48] nithinp7/MRI-Imaging, https://github.com/nithinp7/MRI-Imaging, fetched 2026-10-26 (unofficial)
[S49] ivandrodri/gpu-voxelizer, https://github.com/ivandrodri/gpu-voxelizer, fetched 2026-10-26 (unofficial)
[S50] HeartofTheForce/Isosurface, https://github.com/HeartofTheForce/Isosurface, fetched 2026-10-26 (unofficial)
[S51] heckj/IsosurfaceExtraction, https://github.com/heckj/IsosurfaceExtraction, fetched 2026-10-26 (unofficial)
[S52] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
[S53] pi-GAN: Periodic Implicit Generative Adversarial Networks for 3D-Aware Image Synthesis, https://doi.org/10.1109/cvpr46437.2021.00574, fetched 2026-10-26
[S54] marching cubes, https://www.sciencedirect.com/topics/engineering/marching-cube, fetched 2026-10-27 (unofficial)
[S55] Convex contouring of volumetric data, https://doi.org/10.1007/s00371-003-0216-0, fetched 2026-10-28
[S56] Aug-NeRF: Training Stronger Neural Radiance Fields with Triple-Level Physically-Grounded Augmentations, https://doi.org/10.1109/cvpr52688.2022.01476, fetched 2026-10-26
[S57] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
[S58] How to generate 3D mesh from CT Slices (.raw) using marching cubes algorithm in C#?, https://stackoverflow.com/questions/65498236/how-to-generate-3d-mesh-from-ct-slices-raw-using-marching-cubes-algorithm-in, fetched 2026-10-27 (unofficial)
[S59] Jay2645/VoxelGenerator, https://github.com/Jay2645/VoxelGenerator, fetched 2026-10-27 (unofficial)
[S60] lockre35/Volume-Rendering, https://github.com/lockre35/Volume-Rendering, fetched 2026-10-27 (unofficial)
[S61] マーチングキューブ法を用いた簡易的なメタボール, https://qiita.com/mirai1999/items/0ee93f42be7c0f4f4981, fetched 2026-10-27 (unofficial)
[S62] Unityでボクセルデータからメッシュを生成する方法(Marching Cubes), https://qiita.com/GossiperLoturot/items/15750b9c452cdf0fcd73, fetched 2026-10-27 (unofficial)
[S63] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S64] Marching Cubes: A Quick Overview | Every Algorithm, https://every-algorithm.github.io/2024/10/21/marching_cubes.html, fetched 2026-10-27 (unofficial)
[S65] Marching Cubes — skimage 0.23.2 documentation, https://scikit-image.org/docs/0.23.x/auto_examples/edges/plot_marching_cubes.html, fetched 2026-10-27
[S66] Visualize 3D Meshes Using the Marching Cubes Algorithm, https://spin.atomicobject.com/3d-meshes-marching-cubes/, fetched 2026-10-27 (unofficial)
[S67] mkazhdan/IsoSurfaceExtraction, https://github.com/mkazhdan/IsoSurfaceExtraction, fetched 2026-10-27 (unofficial)
[S68] JuliaGeometry/Meshing.jl, https://github.com/JuliaGeometry/Meshing.jl, fetched 2026-10-27 (unofficial)
[S69] nff747/marching-cubes-wgsl, https://github.com/nff747/marching-cubes-wgsl, fetched 2026-10-27 (unofficial)
[S70] samsfisher/3d-model-xray, https://github.com/samsfisher/3d-model-xray, fetched 2026-10-27 (unofficial)
[S71] LSU-7433-Data-Visiualization/multiple_isosurface, https://github.com/LSU-7433-Data-Visiualization/multiple_isosurface, fetched 2026-10-27 (unofficial)
[S72] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S73] upupming/marching-cubes, https://github.com/upupming/marching-cubes, fetched 2026-10-27 (unofficial)
[S74] RobertBoettcherSF/Ada-Marching-Cubes, https://github.com/RobertBoettcherSF/Ada-Marching-Cubes, fetched 2026-10-27 (unofficial)
[S75] Real-Time GPU-Based Voxelizer - GitHub Pages, https://freder.github.io/UnityGraphicsProgrammingBook1/html-translated/vol2/Chapter%201%20_%20Real-Time%20GPU-Based%20Voxelizer.html, fetched 2026-10-27
[S76] Building a High-Performance Voxel Engine in Unity: A Step ..., https://medium.com/@adamy1558/building-a-high-performance-voxel-engine-in-unity-a-step-by-step-guide-part-6-high-resolution-e0217efd56d2, fetched 2026-10-27 (unofficial)
[S77] feat(tools): voxel_isosurface — marching-cubes isosurface extraction from a 3D voxel field (ArcGIS Export Voxel Isosurface), https://github.com/opengeos/geolibre-rust/issues/499, fetched 2026-10-27 (unofficial)
[S78] Build a 3D Voxel Engine in WebGL | Tutorials, https://www.mysimulator.uk/content/tutorials/voxel-engine.html, fetched 2026-10-27 (unofficial)
[S79] AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, https://github.com/AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, fetched 2026-10-27 (unofficial)
[S80] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S81] Holographic display method for volume data by volume rendering, https://www.researchgate.net/publication/330223296_Holographic_display_method_for_volume_data_by_volume_rendering, fetched 2026-10-27 (unofficial)
[S82] Kaefaell/VoxelDemo, https://github.com/Kaefaell/VoxelDemo, fetched 2026-10-27 (unofficial)
[S83] uranix/mcubes, https://github.com/uranix/mcubes, fetched 2026-10-27 (unofficial)
[S84] Entendore/MarchingCubesDualContouring, https://github.com/Entendore/MarchingCubesDualContouring, fetched 2026-10-27 (unofficial)
[S85] 2453852330/VoxelHelper, https://github.com/2453852330/VoxelHelper, fetched 2026-10-27 (unofficial)
[S86] Level-of-Detail Independent Voxel-Based Surface ..., https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
[S87] Improve surface rendering, https://github.com/CauldronDevelopmentLLC/CAMotics/issues/1, fetched 2026-10-27 (unofficial)
[S88] Verdant2202/TransvoxelUnityImplementation, https://github.com/Verdant2202/TransvoxelUnityImplementation, fetched 2026-10-27 (unofficial)
[S89] vispy.geometry.isosurface module, https://vispy.org/api/vispy.geometry.isosurface.html, fetched 2026-10-27
[S90] STEVE - Space-Time-Enclosing Volume Extraction, https://arxiv.org/abs/1302.5683, fetched 2026-10-27 (unofficial)
[S91] Marching-Cubes, http://www.cs.jhu.edu/~misha/ReadingSeminar/Papers/Lorensen87.pdf, fetched 2026-10-27 (unofficial)
[S92] Volume Rendering Volume Rendering, https://docslib.org/doc/1400821/volume-rendering, fetched 2026-10-28 (unofficial)
[S93] TRTUZ8/TRTUZ8.github.io, https://github.com/TRTUZ8/TRTUZ8.github.io, fetched 2026-10-28 (unofficial)
[S94] 3merillon/WebGL2-GPU-Marching-Cubes, https://github.com/3merillon/WebGL2-GPU-Marching-Cubes, fetched 2026-10-28 (unofficial)
[S95] instantreality 1.0 - tutorial - Volume Rendering, https://doc.instantreality.org/tutorial/volume-rendering/, fetched 2026-10-28 (unofficial)
[S96] Procedural Generation of Voxel Worlds with Castles, https://researchcommons.waikato.ac.nz/entities/publication/70d98783-1629-43df-9d01-b973ddbb5d75, fetched 2026-10-28 (unofficial)
[S97] Rabidus4k/UnityVoxelTerrain, https://github.com/Rabidus4k/UnityVoxelTerrain, fetched 2026-10-28 (unofficial)
[S98] New in CGAL: Isosurfacing, https://www.cgal.org/2025/10/10/Isosurfacing/, fetched 2026-10-28 (unofficial)
[S99] Marching Cubes Tutorial - unice.fr, https://users.polytech.unice.fr/~lingrand/MarchingCubes/applet.html, fetched 2026-10-28 (unofficial)
[S100] Gallery of Volume Graphics, https://people.compute.dtu.dk/janba/gallery/polygonization.html, fetched 2026-10-28
[S101] Smooth Voxel Terrain (Part 2) – 0 FPS, https://0fps.net/2012/07/12/smooth-voxel-terrain-part-2/, fetched 2026-10-28 (unofficial)
[S102] Differentiable Ray Marching: Definition & How It Works, https://inferensys.com/glossary/neural-radiance-fields-and-spatial-computing/implicit-surface-representations/differentiable-ray-marching, fetched 2026-10-28 (unofficial)
[S103] Voxel-Based Procedural Terrain Generation | PDF | Voxel | Rendering ..., https://www.scribd.com/document/840779393/Procedural-Terrain-Generation-Using-Voxels, fetched 2026-10-29 (unofficial)
[S104] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
[S105] Volume Renderer with VTK - yuta-hi, https://github.com/yuta-hi/volume-renderer, fetched 2026-10-29 (unofficial)
[S106] A Comparison of Fundamental Methods for Iso-surface Extraction, https://doi.org/10.48550/arxiv.2301.01715, fetched 2026-10-29 (unofficial)
[S107] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
[S108] marching cubes algorithm, https://www.sciencedirect.com/topics/computer-science/marching-cube-algorithm, fetched 2026-10-30 (book)
[S109] Marching Cubes Marching Cubes, https://daac.hpc.mil/gettingStarted/Marching_Cubes.html, fetched 2026-10-29
[S110] http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, fetched 2026-10-30 (unofficial)
[S111] Marching Cubes algorithm, https://github.com/thejasonfisher/marchingCubes/blob/master/MarchingCubes.cs, fetched 2026-10-30 (unofficial)
[S112] 2 Marching Cubes isovalue, https://stackoverflow.com/questions/36106475/marching-cubes-isovalue?rq=4, fetched 2026-10-30 (unofficial)
[S113] Really Fast Isocontouring, https://www.kitware.com/really-fast-isocontouring/, fetched 2026-10-30 (unofficial)
[S114] Marching Cubes | Michael Walczyk, https://michaelwalczyk.com/project-marching-cubes.html, fetched 2026-10-30 (unofficial)
[S115] Marching Cubes - Alschain, https://alschain.com/2022/06/25/MarchingCubes/, fetched 2026-10-30 (unofficial)
[S116] 水泡动画模拟（Marching Cubes） - 算法小丑, https://www.cnblogs.com/shushen/p/5542131.html, fetched 2026-10-30 (unofficial)
[S117] Fourier volume rendering, https://shiftleft.com/mirrors/www.hpl.hp.com/techreports/95/HPL-95-73.pdf, fetched 2026-10-31 (unofficial)
