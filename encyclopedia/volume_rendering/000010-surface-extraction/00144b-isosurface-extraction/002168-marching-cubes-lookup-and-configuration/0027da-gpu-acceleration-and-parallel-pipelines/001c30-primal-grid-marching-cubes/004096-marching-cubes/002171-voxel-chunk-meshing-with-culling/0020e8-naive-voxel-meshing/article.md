# Naive Voxel Meshing

Also called: Classic Voxel Meshing, Face-by-face meshing, Naive Meshing, Naïve Voxel Mesh Generation

## Overview

Naive Voxel Meshing is a brute-force surface extraction algorithm that generates a mesh by iterating through every voxel in a chunk and emitting geometry for all six faces of every solid voxel, regardless of adjacency [S2, S5, unofficial]. Unlike its sibling [Face Culling Voxel Meshing](2268-face-culling-voxel-meshing.md), which checks neighbors to omit internal faces, Naive Voxel Meshing ignores occlusion and adjacency, producing a redundant mesh where every solid voxel contributes up to six quads [S1, S5, unofficial]. This method serves as a baseline for correctness and simplicity, often used in educational contexts or as a fallback before applying optimization techniques like greedy meshing [S1, unofficial].

## How it works

**Inputs and Outputs:**
*   **Input:** A voxel chunk defined by dimensions $D_x \times D_y \times D_z$ (typically cubic, $N \times N \times N$) and a 3D array or linear buffer storing voxel states (e.g., material IDs or solid/empty flags) [S2, S4, unofficial].
*   **Output:** A list of mesh primitives (quads or triangles) representing the voxel surfaces [S2, S5, unofficial].

**Data Structures:**
*   **Voxel Grid:** A 3D array `voxels[x][y][z]` or a linear array `voxels[x + z*D_x + y*D_x*D_y]` [S4, unofficial]. A value of 0 typically denotes empty/transparent space; non-zero denotes solid [S2, S4, unofficial].
*   **Vertex Buffer:** A list of 3D coordinates. In the naive approach, vertices are often not shared between adjacent quads to simplify indexing, or a global counter is used for unique vertex indices [S4, S5, unofficial].
*   **Index Buffer:** A list of integer indices referencing the vertex buffer [S3, unofficial].
*   **Lookup Tables:**
    *   `voxelVertices`: An array of 8 unit vectors representing the corners of a unit cube. Common layout (from [S4, unofficial]):
        *   Indices 0-3 (Top, y=1): `(0,1,0), (1,1,0), (1,1,1), (0,1,1)`
        *   Indices 4-7 (Bottom, y=0): `(0,0,0), (1,0,0), (1,0,1), (0,0,1)`
    *   `voxelVertexIndex`: A $6 \times 4$ array mapping each face direction to the 4 vertex indices forming the quad [S4, unofficial].
    *   `voxelTriangles`: An array `{0, 3, 2, 0, 2, 1}` defining two triangles per quad [S4, unofficial].

**Algorithm Steps:**

1.  **Initialize** empty lists for vertices and indices [S5, unofficial].
2.  **Iterate** through every voxel coordinate $(x, y, z)$ within the chunk bounds $[0, N)^3$ using three nested loops [S2, S4, S5, unofficial].
3.  **Check State:** Retrieve the voxel state at $(x, y, z)$. If the state is 0 (empty/transparent), skip to the next voxel [S2, S4, unofficial].
4.  **Generate Faces:** For each of the 6 faces of the current solid voxel, generate a quad [S2, S5, unofficial].
    *   *Note:* In the strict "Naive" definition, all 6 faces are generated without neighbor checks [S2, S5, unofficial]. However, some implementations labeled "Naive" in source code still perform basic neighbor checks for culling (see [S3, S4, unofficial]). The pure naive method ignores these checks. The following pseudocode describes the pure naive approach (no culling). If culling is desired, see [Face Culling Voxel Meshing](2268-face-culling-voxel-meshing.md).
5.  **Vertex Calculation:** For each face, calculate the 4 vertex positions by adding the voxel's integer coordinates $(x, y, z)$ to the local face vertex offsets [S5, unofficial].
    *   Back Face (Normal $(0,0,-1)$): $V_0=(x,y,z), V_1=(x,y+1,z), V_2=(x+1,y+1,z), V_3=(x+1,y,z)$ [S5, unofficial].
    *   Front Face (Normal $(0,0,1)$): $V_0=(x,y,z+1), V_1=(x+1,y,z+1), V_2=(x+1,y+1,z+1), V_3=(x,y+1,z+1)$ [S5, unofficial].
    *   Left Face (Normal $(-1,0,0)$): $V_0=(x,y,z), V_1=(x,y,z+1), V_2=(x,y+1,z+1), V_3=(x,y+1,z)$ [S5, unofficial].
    *   Right Face (Normal $(1,0,0)$): $V_0=(x+1,y,z+1), V_1=(x+1,y,z), V_2=(x+1,y+1,z), V_3=(x+1,y+1,z+1)$ [S5, unofficial].
    *   Bottom Face (Normal $(0,-1,0)$): $V_0=(x,y,z), V_1=(x+1,y,z), V_2=(x+1,y,z+1), V_3=(x,y,z+1)$ [S5, unofficial].
    *   Top Face (Normal $(0,1,0)$): $V_0=(x,y+1,z+1), V_1=(x+1,y+1,z+1), V_2=(x+1,y+1,z), V_3=(x,y+1,z)$ [S5, unofficial].
6.  **Indexing:** Add the 4 vertex indices to the index buffer. In a non-shared vertex scheme, 4 new unique indices are added per face [S5, unofficial]. In a shared vertex scheme (as seen in some "Naive" implementations like [S4, unofficial]), a global vertex counter is incremented, and the 4 vertices are added to the vertex buffer with new unique indices [S4, unofficial].
7.  **Return:** Construct the mesh from the vertex and index lists [S5, unofficial].

**Pseudocode (Pure Naive, No Culling):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mesh_naive(chunk_size, voxels):
    """
    Generates a mesh for a voxel chunk by emitting all 6 faces for every solid voxel.
    voxels: 3D array or flat list of voxel states (0 = empty, >0 = solid)
    chunk_size: int, dimension N of the N x N x N chunk
    """
    vertices = []
    indices = []
    
    # Predefined unit cube corners for reference if needed, 
    # but direct calculation is simpler for naive approach.
    
    for x in range(chunk_size):
        for y in range(chunk_size):
            for z in range(chunk_size):
                # Get voxel state. Handle flat vs 3D indexing as needed.
                # Assuming flat indexing: x + z*chunk_size + y*chunk_size*chunk_size
                voxel_idx = x + z * chunk_size + y * chunk_size * chunk_size
                if voxels[voxel_idx] == 0:
                    continue
                
                # Generate 6 faces. 
                # Face definitions: (normal_x, normal_y, normal_z, v0_offset, v1_offset, v2_offset, v3_offset)
                # Offsets are relative to (x,y,z)
                faces = [
                    # Back (0,0,-1)
                    (0, 0, -1, (0,0,0), (0,1,0), (1,1,0), (1,0,0)),
                    # Front (0,0,1)
                    (0, 0, 1, (0,0,1), (1,0,1), (1,1,1), (0,1,1)),
                    # Left (-1,0,0)
                    (-1, 0, 0, (0,0,0), (0,0,1), (0,1,1), (0,1,0)),
                    # Right (1,0,0)
                    (1, 0, 0, (1,0,1), (1,0,0), (1,1,0), (1,1,1)),
                    # Bottom (0,-1,0)
                    (0, -1, 0, (0,0,0), (1,0,0), (1,0,1), (0,0,1)),
                    # Top (0,1,0)
                    (0, 1, 0, (0,1,1), (1,1,1), (1,1,0), (0,1,0))
                ]
                
                for nx, ny, nz, v0, v1, v2, v3 in faces:
                    # Calculate absolute vertex positions
                    p0 = (x + v0[0], y + v0[1], z + v0[2])
                    p1 = (x + v1[0], y + v1[1], z + v1[2])
                    p2 = (x + v2[0], y + v2[1], z + v2[2])
                    p3 = (x + v3[0], y + v3[1], z + v3[2])
                    
                    # Add vertices (Naive approach often does not share vertices)
                    idx0 = len(vertices)
                    idx1 = len(vertices) + 1
                    idx2 = len(vertices) + 2
                    idx3 = len(vertices) + 3
                    
                    vertices.extend([p0, p1, p2, p3])
                    
                    # Add two triangles per quad (0,3,2 and 0,2,1)
                    indices.extend([idx0, idx3, idx2, idx0, idx2, idx1])
                    
    return vertices, indices
```

**Edge Cases and Pitfalls:**
*   **Boundary Checks:** If the algorithm is applied to a single chunk without neighbor data, it must decide how to handle faces at the chunk boundary. The pure naive method emits them regardless [S5, unofficial]. If culling is added, boundary neighbors must be defined (e.g., "Air" or "Solid" policy) [S2, S4, unofficial].
*   **Vertex Sharing:** The pure naive method does not share vertices, leading to high memory usage. Some implementations labeled "Naive" do share vertices but still emit all faces, which is inefficient [S4, unofficial].
*   **Performance:** The algorithm generates $6 \times N_{solid}$ quads. For a solid $32^3$ chunk, this results in 196,608 quads [S2, unofficial].

## Subtopics

### [Naive Surface Net](3679-naive-surface-net.md)
A variant that averages intersection points on voxel edges to create smoothed feature points before constructing quads, producing a smoother surface than standard voxel extraction.

## Operating characteristics

*   **Time Complexity:** $O(N^3)$, where $N$ is the chunk dimension. The algorithm iterates through every voxel and performs constant work per voxel [S2, S5, unofficial].
*   **Memory Complexity:** $O(N^3)$ for the output mesh. The number of vertices and indices is proportional to the number of solid voxels times 6 [S2, unofficial].
*   **Measured Performance:** On an Apple M2 Pro (macOS 26.5.1, Node v22.22.2, three@0.185.1), naive meshing a 32³ terrain chunk with 17,191 solid voxels took a median of 5.60 ms (range 5.12–6.30 ms) [S2, unofficial].
*   **Mesh Statistics:**
    *   Solid $32^3$ cube: 196,608 quads [S2, unofficial].
    *   32³ terrain chunk (17,191 solid voxels): 103,146 quads, 412,584 vertices, 206,292 triangles [S2, unofficial].
*   **Strengths:** Extremely simple to implement; no neighbor lookups or complex data structures required; deterministic output [S1, S5, unofficial].
*   **Limitations:** High polygon count due to lack of face culling; high draw call count if not batched; inefficient memory usage due to non-shared vertices [S1, S4, unofficial].
*   **When to Use:** Prototyping, educational purposes, or as a baseline for testing other meshing algorithms [S1, unofficial].
*   **When Not to Use:** Production environments with large worlds or performance constraints, where culling or greedy meshing is required [S1, unofficial].

## Background

Naive Voxel Meshing is a foundational technique in voxel rendering, often cited as the first step in converting voxel data into GPU-renderable geometry [S1, unofficial]. It is described in various educational resources and open-source projects as a starting point for voxel engines [S1, S4, unofficial]. The method is contrasted with more advanced techniques like Culled Meshing and Greedy Meshing, which reduce polygon counts by omitting internal faces or merging adjacent faces [S1, unofficial]. The term "Naive" highlights the lack of optimization in face generation [S4, S5, unofficial].

## Sources

[S1] Presentation on the noa voxel game engine, made and presented in-engine (2015), http://andyhall.github.io/noa-lt/, fetched 2026-10-26 (unofficial)
[S2] m2-md/voxel-greedy-meshing-typescript, https://github.com/m2-md/voxel-greedy-meshing-typescript, fetched 2026-10-26 (unofficial)
[S3] Justinj68/OM3D_PROJECT, https://github.com/Justinj68/OM3D_PROJECT, fetched 2026-10-27 (unofficial)
[S4] MrAlvaroRamirez/Unity-SurfaceNet-VoxelTerrain, https://github.com/MrAlvaroRamirez/Unity-SurfaceNet-VoxelTerrain, fetched 2026-10-27 (unofficial)
[S5] GitHub - nicolas-goyon/Voxel-Mesh-Optimization-Library, https://github.com/nicolas-goyon/Voxel-Mesh-Optimization-Library, fetched 2026-10-29 (unofficial)
