# Naive Voxel Meshing

Also called: Classic Voxel Meshing, Face-by-face meshing, Naive Meshing, Naive Surface-Net Smoothing

## Overview
Naive Voxel Meshing generates a mesh by emitting a quad for every exposed face of every solid voxel in a chunk, without sharing vertices or merging adjacent faces [S1, unofficial][S3, unofficial]. It converts discrete voxel data into a renderable triangle soup by prioritizing implementation simplicity over geometric efficiency [S1, unofficial]. Unlike [Voxel Mesh Generation with Face Culling](2411-voxel-mesh-generation-with-face-culling.md), which suppresses faces between two solid voxels, the unculled variant of Naive Voxel Meshing emits all six faces for every solid voxel, ignoring neighbors entirely [S1, unofficial]. This produces a mesh with significant redundancy but guarantees exact voxel boundary representation without complex neighbor lookups [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A voxel chunk data structure containing a 3D grid of voxel IDs and chunk dimensions $S \times S \times S$ [S1, unofficial]. A voxel ID of 0 denotes empty/transparent space, while non-zero IDs denote solid material [S1, unofficial][S3, unofficial].
- **Output:** A list of vertices and indices representing the surface geometry [S2, unofficial][S3, unofficial].

**Data Structures:**
The method relies on static lookup tables to define the geometry of a single unit voxel [S3, unofficial]:
1.  **`voxelVertices`**: An array of 8 3D coordinates representing the corners of a unit voxel [S3, unofficial].
    -   Indices 0–3: Top face ($y=1$).
    -   Indices 4–7: Bottom face ($y=0$).
    -   Specific layout [S3, unofficial]:
        -   0: $(0,1,0)$, 1: $(1,1,0)$, 2: $(1,1,1)$, 3: $(0,1,1)$
        -   4: $(0,0,0)$, 5: $(1,0,0)$, 6: $(1,0,1)$, 7: $(0,0,1)$
2.  **`voxelVertexIndex`**: A $6 \times 4$ table mapping each of the 6 face directions to the 4 vertex indices from `voxelVertices` that form the quad [S3, unofficial].
    -   Top: $(0,1,2,3)$, Bottom: $(7,6,5,4)$
    -   Left: $(7,4,0,3)$, Right: $(5,6,2,1)$
    -   Front: $(6,7,3,2)$, Back: $(4,5,1,0)$
3.  **`voxelTriangles`**: A fixed array of 6 indices $\{0, 3, 2, 0, 2, 1\}$ used to split a quad into two triangles [S3, unofficial].
4.  **`voxelFaceChecks`**: Direction vectors for the 6 faces [S3, unofficial]:
    -   Top $(0,1,0)$, Bottom $(0,-1,0)$, Left $(-1,0,0)$, Right $(1,0,0)$, Front $(0,0,1)$, Back $(0,0,-1)$.

**Parameters:**
-   $S$: Chunk dimension (assumed cubic). Typical value: 32 [S1, unofficial][S3, unofficial].
-   `CHUNK_AREA`: $S \times S$, used for linear indexing [S3, unofficial].

**Algorithm Steps:**
1.  Initialize empty lists for vertices and indices [S2, unofficial][S3, unofficial].
2.  Iterate through every voxel coordinate $(x, y, z)$ in the chunk, where $0 \le x, y, z < S$ [S1, unofficial][S3, unofficial].
3.  Retrieve the voxel ID at $(x, y, z)$ using linear indexing: $idx = x + (z \cdot S) + (y \cdot S^2)$ [S3, unofficial].
4.  If the voxel ID is 0 (empty), skip to the next voxel [S1, unofficial][S3, unofficial].
5.  For each of the 6 faces of the current voxel:
    a. Determine the neighbor's coordinate $(nx, ny, nz)$ by adding the face direction vector to $(x, y, z)$ [S3, unofficial].
    b. Check if the neighbor is transparent (empty).
        -   If the neighbor is outside the chunk boundaries, apply a boundary policy (e.g., treat as solid or empty) [S3, unofficial].
        -   If the neighbor is solid, skip this face (it is internal) [S3, unofficial].
    c. If the neighbor is empty or the boundary policy dictates exposure:
        -   Retrieve the 4 vertex indices from `voxelVertexIndex` for this face [S3, unofficial].
        -   For each vertex index, compute the world position by adding the local corner coordinate from `voxelVertices` to $(x, y, z)$ [S3, unofficial].
        -   Append the 4 vertices to the vertex list and record their base index [S3, unofficial].
        -   Append the 2 triangle indices (mapped from `voxelTriangles` to the new vertex indices) to the index list [S3, unofficial].
6.  Return the vertex and index lists [S2, unofficial][S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def naive_voxel_meshing(chunk_size, get_voxel_id, boundary_policy="empty"):
    """
    chunk_size: int S
    get_voxel_id: function(x, y, z) -> int (0 is empty)
    boundary_policy: "empty" or "solid"
    """
    vertices = []
    indices = []
    
    # Static geometry tables from [S3, unofficial]
    voxel_vertices = [
        (0,1,0), (1,1,0), (1,1,1), (0,1,1), # 0-3 Top
        (0,0,0), (1,0,0), (1,0,1), (0,0,1)  # 4-7 Bottom
    ]
    
    # Face vertex indices: Top, Bottom, Left, Right, Front, Back
    voxel_vertex_index = [
        [0, 1, 2, 3], # Top
        [7, 6, 5, 4], # Bottom
        [7, 4, 0, 3], # Left
        [5, 6, 2, 1], # Right
        [6, 7, 3, 2], # Front
        [4, 5, 1, 0]  # Back
    ]
    
    # Face directions: Top, Bottom, Left, Right, Front, Back
    voxel_face_checks = [
        (0, 1, 0), (0, -1, 0), (-1, 0, 0), (1, 0, 0), (0, 0, 1), (0, 0, -1)
    ]
    
    # Triangle split: 0,3,2 and 0,2,1
    voxel_triangles = [0, 3, 2, 0, 2, 1]
    
    vertex_count = 0
    
    for x in range(chunk_size):
        for y in range(chunk_size):
            for z in range(chunk_size):
                # Get voxel ID
                voxel_id = get_voxel_id(x, y, z)
                if voxel_id == 0:
                    continue
                
                # Check 6 faces
                for face_idx in range(6):
                    dx, dy, dz = voxel_face_checks[face_idx]
                    nx, ny, nz = x + dx, y + dy, z + dz
                    
                    # Check neighbor solidity
                    is_neighbor_solid = False
                    if 0 <= nx < chunk_size and 0 <= ny < chunk_size and 0 <= nz < chunk_size:
                        neighbor_id = get_voxel_id(nx, ny, nz)
                        is_neighbor_solid = (neighbor_id != 0)
                    else:
                        # Boundary handling [S3, unofficial]
                        if boundary_policy == "solid":
                            is_neighbor_solid = True
                        else:
                            is_neighbor_solid = False
                            
                    if is_neighbor_solid:
                        continue # Face is internal
                    
                    # Face is exposed, generate quad
                    face_vertex_indices = voxel_vertex_index[face_idx]
                    local_vertex_positions = []
                    
                    for v_idx in face_vertex_indices:
                        vx, vy, vz = voxel_vertices[v_idx]
                        # World position
                        wx = x + vx
                        wy = y + vy
                        wz = z + vz
                        local_vertex_positions.append((wx, wy, wz))
                        
                    # Add vertices
                    for pos in local_vertex_positions:
                        vertices.append(pos)
                        
                    # Add triangles
                    # Map local indices 0-3 to global vertex indices
                    base_idx = vertex_count
                    for i in range(6):
                        indices.append(base_idx + voxel_triangles[i])
                        
                    vertex_count += 4
                    
    return vertices, indices
```

**Edge Cases and Pitfalls:**
-   **Boundary Checks:** Neighbors outside the chunk must be handled. If the chunk is part of a larger world, the algorithm must query the global world data or adjacent chunks [S3, unofficial]. If treated as a standalone chunk, a boundary policy (empty or solid) must be chosen [S3, unofficial].
-   **Vertex Duplication:** This method does not share vertices between adjacent quads. A single internal edge between two exposed faces will result in 8 vertices (4 per quad) instead of 4 [S3, unofficial].
-   **Precision:** Using integer coordinates for vertices is standard for voxel meshes, but floating-point positions may be needed for smoothing variants like Naive Surface Net [S3, unofficial].

## Subtopics

### [Naive Surface Net](3679-naive-surface-net.md)
A surface reconstruction technique that estimates vertex positions by averaging the midpoints of voxel edges crossing the implicit surface boundary, producing a smoothed mesh compared to standard Marching Cubes or simple voxel extraction. It generates geometry for all six faces of every solid voxel in a chunk, ignoring neighbors and performing no culling, building on [Naive Voxel Meshing](2376-naive-voxel-meshing.md).

## Operating characteristics
-   **Time Complexity:** $O(N)$, where $N$ is the number of voxels in the chunk ($S^3$). Each voxel is processed once, and 6 faces are checked [S1, unofficial][S3, unofficial].
-   **Memory Complexity:** $O(F)$, where $F$ is the number of exposed faces [S3, unofficial].
-   **Measured Performance:** On an Apple M2 Pro (macOS 26.5.1, Node v22.22.2, three@0.185.1), naive meshing median time is 5.60 ms (range 5.12–6.30 ms, 21% spread) on a quiet machine for a 32³ terrain chunk with 17,191 solid voxels [S1, unofficial].
-   **Mesh Size:** On a solid 32³ cube, it produces 196,608 quads [S1, unofficial]. On a 32³ terrain chunk with 17,191 solid voxels, it produces 103,146 quads, 412,584 vertices, and 206,292 triangles [S1, unofficial].
-   **Strengths:** Extremely simple to implement; deterministic output; no complex data structures required [S1, unofficial].
-   **Limitations:** High vertex and index count due to lack of vertex sharing and face culling; poor performance for large solid volumes [S1, unofficial][S3, unofficial].
-   **When to use:** Prototyping, educational purposes, or as a baseline for comparison with more advanced meshing algorithms like Greedy Meshing [S1, unofficial].
-   **When not to use:** Production environments requiring efficient rendering of large voxel worlds [S1, unofficial].

## Background
Naive Voxel Meshing serves as a foundational technique in voxel terrain generation, often acting as the baseline for more advanced surface extraction methods [S2, unofficial][S3, unofficial]. It is implemented in various voxel game prototypes and educational resources to demonstrate the conversion of voxel data to mesh geometry [S2, unofficial][S3, unofficial]. The method provides the direct geometric input for smoothing algorithms like Naive Surface Net [S3, unofficial].

## Sources
[S1] m2-md/voxel-greedy-meshing-typescript, https://github.com/m2-md/voxel-greedy-meshing-typescript, fetched 2026-10-26 (unofficial)
[S2] Justinj68/OM3D_PROJECT, https://github.com/Justinj68/OM3D_PROJECT, fetched 2026-10-27 (unofficial)
[S3] MrAlvaroRamirez/Unity-SurfaceNet-VoxelTerrain, https://github.com/MrAlvaroRamirez/Unity-SurfaceNet-VoxelTerrain, fetched 2026-10-27 (unofficial)
