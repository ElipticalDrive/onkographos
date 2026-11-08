# Naive Surface Net

Also called: GPU Surface Nets, Surface nets surface extraction algorithm, Surface-Net smoothing

## Overview
Naive Surface Net is a GPU-accelerated surface extraction algorithm that reconstructs a mesh by averaging intersection points on voxel edges to create smoothed feature points, then constructing quads from these points [S1, S2, unofficial]. It differs from Marching Cubes by generating feature points first and then quads, rather than triangles directly from cube configurations, and produces a smoother surface than standard voxel extraction by averaging edge midpoints instead of using raw integer grid coordinates [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D chunk of voxel data. The voxel data is stored in a 1D byte array `voxels` of size `CHUNK_VOL` ($CHUNK\_SIZE^3$) [S2, unofficial]. A voxel is considered "void" if its value is 0 [S2, unofficial]. The chunk dimensions are given by `Size` ($x, y, z$) [S1, unofficial].
*   **Output:** A mesh consisting of vertices, triangle indices, colors, and UVs, stored in a `ChunkMeshData` structure [S2, unofficial].

**Data Structures:**
*   **Voxel:** Contains a single field `float Density` in the GPU implementation [S1, unofficial]. In the CPU implementation, it is a `byte` where 0 indicates void [S2, unofficial].
*   **Edge:** Contains two integer fields `Index1` and `Index2` representing indices into the 8-vertex cube array [S1, unofficial].
*   **QuadVertices:** Contains four `float3` fields `vertice1`, `vertice2`, `vertice3`, `vertice4` [S1, unofficial].
*   **ChunkMeshData:** Contains `List<Vector3> vertices`, `List<int> triangles`, `List<Color> colors`, `List<Vector2> uvs`, and a `Mesh` object [S2, unofficial]. Triangles are stored as a flat list of vertex indices [S2, unofficial].
*   **GPU Buffers:**
    *   `Voxels`: `StructuredBuffer<Voxel>` [S1, unofficial].
    *   `FeaturePoints`: `RWStructuredBuffer<float3>` [S1, unofficial].
    *   `Quads`: `AppendStructuredBuffer` (or similar) storing `QuadVertices` [S1, unofficial].
    *   `Indices`: `RWStructuredBuffer<int>` storing triangle indices [S1, unofficial].
    *   `VerticesArgumentBuffer`: `Buffer` of type `IndirectArguments` for indirect dispatch [S1, unofficial].

**Index Calculation:**
The index for a voxel at coordinates $(x, y, z)$ in a 1D array is calculated as [S1, S2, unofficial]:
$$ index = x + y \cdot (Size.x \cdot Size.z) + z \cdot Size.z $$
Note: Source [S2, unofficial] uses a slightly different order for the byte array: `index = x + (z * CHUNK_SIZE) + (y * CHUNK_AREA)`. Both map 3D coordinates to 1D, but the stride differs. The GPU kernel [S1, unofficial] uses the first formula.

**Core Formulas:**
1.  **Inverse Linear Interpolation (finding $t$):** To find the normalized distance $t$ along an edge where the density is zero [S1, unofficial]:
    $$ t = \frac{0 - d_0}{d_1 - d_0} $$
    Implemented as `(t - a) / (b - a)` with $a=d_0, b=d_1, t=0$ [S1, unofficial]. If $a == b$, $t$ returns 0 to prevent division by zero [S1, unofficial].
2.  **Linear Interpolation (finding position $p$):**
    $$ p = (1 - t) \cdot p_0 + t \cdot p_1 $$
    where $p_0, p_1$ are the endpoints of the edge [S1, unofficial].
3.  **Smoothed Vertex Position ($v_{smooth}$):** The position of a vertex is the average of the midpoints of all voxel edges that share that corner and cross the surface boundary [S2, unofficial]:
    $$ v_{smooth} = \frac{1}{N} \sum_{k=1}^{N} \frac{e_{k,1} + e_{k,2}}{2} $$
    where $e_{k,1}$ and $e_{k,2}$ are the integer coordinates of the endpoints of the $k$-th voxel edge crossing the surface, and $N$ is the count of such edges [S2, unofficial].

**Algorithm Steps:**

1.  **Dispatch Kernel 0 (FindFeaturePoints):**
    *   Thread group size: $8 \times 8 \times 8$ [S1, unofficial].
    *   Dispatch count: $(Size.x/8, Size.y/8, Size.z/8)$ [S1, unofficial].
    *   For each voxel at `position` $(x, y, z)$:
        *   Check boundary: If $x+1 \ge Size.x$, $y+1 \ge Size.y$, or $z+1 \ge Size.z$, skip (return) [S1, unofficial].
        *   Initialize `total = float3(0,0,0)` and `count = 0` [S2, unofficial].
        *   Iterate through the 12 edges of the voxel cube [S2, unofficial].
        *   For each edge, determine the positions of its two endpoints: `edgePos` and `edgeNeighborPos` [S2, unofficial].
        *   Check if the edge crosses the surface: If `IsVoid(edgePos) != IsVoid(edgeNeighborPos)` [S2, unofficial].
        *   If it crosses, add the midpoint to the sum: `total += (edgePos + edgeNeighborPos) / 2.0f` and increment `count` [S2, unofficial].
        *   After iterating all 12 edges, if `count > 0`, write `total / count` to `FeaturePoints[GetIndex(position)]` [S2, unofficial].

2.  **Dispatch Kernel 1 (FindVertices):**
    *   Thread group size: $8 \times 8 \times 8$ [S1, unofficial].
    *   Dispatch count: $(Size.x/8, Size.y/8, Size.z/8)$ [S1, unofficial].
    *   For each voxel at `localPos` $(x, y, z)$:
        *   Iterate through the 3 positive axis directions ($x, y, z$) [S2, unofficial].
        *   For each axis, calculate `neighborPos = localPos + axis` [S2, unofficial].
        *   Determine if the current voxel exists (is not void) and if the neighbor exists [S2, unofficial].
        *   If exactly one of the voxel or neighbor exists (boundary case), create a surface quad [S2, unofficial].
        *   Compute the 4 vertices of the quad face using `AddQuad` [S2, unofficial].
            *   `AddQuad` calculates the smoothed position for each of the 4 corners of the quad face by averaging the midpoints of all 12 voxel edges that share that corner and cross the surface boundary [S2, unofficial].
        *   Split the quad into two triangles (6 vertices total) [S2, unofficial].
        *   Use `voxelTriangles` indices if the current voxel is solid, otherwise use `voxelTrianglesReverse` to maintain consistent winding order [S2, unofficial].
        *   Append the 6 triangle indices to the `Indices` buffer and the 4 vertices to the `Quads` buffer [S1, unofficial].

3.  **Copy Count:**
    *   Copy the count of generated quads from the append buffer (`Quads`) to an indirect argument buffer (`VerticesArgumentBuffer`) [S1, unofficial].

4.  **Dispatch Kernel 2 (FindIndices):**
    *   Thread count equal to the number of quads [S1, unofficial].
    *   For each quad, generate the 6 triangle indices based on the quad's vertex indices and append them to the `Indices` buffer [S1, unofficial].

5.  **Construct Mesh:**
    *   Read the vertices from the `Quads` buffer [S1, unofficial].
    *   Read the indices from the `Indices` buffer [S1, unofficial].
    *   Assign to `mesh.vertices` and `mesh.triangles` [S1, unofficial].
    *   Recalculate normals [S1, unofficial].

**Pseudocode (CPU Implementation of FindVertices logic):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
CHUNK_SIZE = 16 # Example
CHUNK_AREA = CHUNK_SIZE * CHUNK_SIZE
CHUNK_VOL = CHUNK_SIZE * CHUNK_SIZE * CHUNK_SIZE

# Lookup tables
voxelAxis = [
    [1, 0, 0],
    [0, 1, 0],
    [0, 0, 1]
]

voxelQuads = [
    [[0, 0, -1], [0, -1, -1], [0, -1, 0], [0, 0, 0]],
    [[0, 0, -1], [0, 0, 0], [-1, 0, 0], [-1, 0, -1]],
    [[0, 0, 0], [0, -1, 0], [-1, -1, 0], [-1, 0, 0]]
]

edgeOffsets = [
    # Edges on min Z axis
    [[0, 0, 0], [1, 0, 0]],
    [[1, 0, 0], [1, 1, 0]],
    [[1, 1, 0], [0, 1, 0]],
    [[0, 1, 0], [0, 0, 0]],
    # Edges on max Z axis
    [[0, 0, 1], [1, 0, 1]],
    [[1, 0, 1], [1, 1, 1]],
    [[1, 1, 1], [0, 1, 1]],
    [[0, 1, 1], [0, 0, 1]],
    # Edges connecting min Z to max Z
    [[0, 0, 0], [0, 0, 1]],
    [[1, 0, 0], [1, 0, 1]],
    [[1, 1, 0], [1, 1, 1]],
    [[0, 1, 0], [0, 1, 1]]
]

voxelTriangles = [0, 3, 2, 0, 2, 1]
voxelTrianglesReverse = [0, 2, 3, 0, 1, 2]

def IsVoid(pos, voxels):
    # Check if pos is inside chunk bounds
    if not (0 <= pos[0] < CHUNK_SIZE and 0 <= pos[1] < CHUNK_SIZE and 0 <= pos[2] < CHUNK_SIZE):
        return True # Treat out-of-bounds as void or handle via neighbor check
    index = pos[0] + (pos[2] * CHUNK_SIZE) + (pos[1] * CHUNK_AREA)
    return voxels[index] == 0

def AverageVertex(localPos, axis, voxels):
    # Calculate smoothed position for a corner of the quad
    # Identify the 4 corners of the quad face defined by localPos and axis
    # For each corner, average the midpoints of all 12 voxel edges sharing that corner and crossing the surface
    # This is a simplified placeholder for the complex averaging logic described in [S2, unofficial]
    # In practice, this requires iterating over the 12 edges of the voxel cube and checking boundary conditions
    pass

def AddQuad(localPos, axis, voxels):
    # Determine the 4 corners of the quad face
    # Compute smoothed positions for each corner
    # Return list of 4 vertices
    pass

def GenerateMesh(voxels, chunkPos):
    meshData = {
        'vertices': [],
        'triangles': [],
        'colors': [],
        'uvs': []
    }
    
    # Iterate through all voxels, including edge quads
    # Note: The loop range in [S2, unofficial] is -1 to CHUNK_SIZE + 1 to include edge quads
    for x in range(-1, CHUNK_SIZE + 1):
        for y in range(-1, CHUNK_SIZE + 1):
            for z in range(-1, CHUNK_SIZE + 1):
                localPos = [x, y, z]
                
                # Check if voxel is void
                if IsVoid(localPos, voxels):
                    continue
                
                # Iterate through positive axis directions
                for axis in voxelAxis:
                    neighborPos = [localPos[i] + axis[i] for i in range(3)]
                    
                    # Check if neighbor exists (is not void)
                    # Note: IsVoid returns True for void. We want to check if exactly one is void.
                    is_current_void = IsVoid(localPos, voxels)
                    is_neighbor_void = IsVoid(neighborPos, voxels)
                    
                    if is_current_void != is_neighbor_void:
                        # Boundary case: create surface quad
                        quadVertices = AddQuad(localPos, axis, voxels)
                        
                        # Split quad into two triangles
                        if not is_current_void: # Current voxel is solid
                            indices = voxelTriangles
                        else:
                            indices = voxelTrianglesReverse
                        
                        for idx in indices:
                            meshData['triangles'].append(len(meshData['vertices']) + idx)
                        
                        meshData['vertices'].extend(quadVertices)
                        
                        # Add colors and UVs (simplified)
                        # ...
    
    return meshData
```

**Parameters:**
*   **Thread group size:** $8 \times 8 \times 8$ threads per group [S1, unofficial].
*   **Smoothing toggle:** To disable smoothing and use raw voxel positions, replace the `AverageVertex` call with `vertices[i] = quadVertexPos` [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Check:** The `FindFeaturePoints` kernel skips voxels where $x+1 \ge Size.x$, $y+1 \ge Size.y$, or $z+1 \ge Size.z$ to avoid accessing out-of-bounds neighbor voxels [S1, unofficial].
*   **Boundary Handling:** The iteration loop extends from -1 to `CHUNK_SIZE + 1` to ensure quads on the chunk edges are generated, requiring neighbor checks to handle out-of-bounds or adjacent chunk data correctly via `IsInsideChunk` [S2, unofficial].
*   **Division by Zero:** In `InverseLerp`, if $a == b$, the function returns 0 to prevent division by zero [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Produces a smoother surface than standard voxel extraction [S2, unofficial]. Runs on the GPU for real-time performance [S1, unofficial].
*   **Limitations:** May have gaps in the terrain (as noted by the author) [S1, unofficial]. CSG Boolean operations are a work in progress [S1, unofficial].
*   **When to use:** Real-time deformable terrain where smoothing is desired [S1, unofficial].
*   **When not to use:** When exact geometric fidelity is required (vs. smoothed approximation) [S2, unofficial].

## Background
*   **Origin:** The method is an implementation of the Surface Nets surface extraction algorithm running on the GPU [S1, unofficial].
*   **History:** The specific implementation described is from an unofficial GitHub repository [S1, S2, unofficial].
*   **Usage:** Planned for use as a real-time deformable terrain with CSG Boolean operations and Chunking [S1, unofficial].

## Sources
[S1] Dmajster/surface_nets, https://github.com/Dmajster/surface_nets, fetched 2026-10-27 (unofficial)
[S2] MrAlvaroRamirez/Unity-SurfaceNet-VoxelTerrain, https://github.com/MrAlvaroRamirez/Unity-SurfaceNet-VoxelTerrain, fetched 2026-10-27 (unofficial)
