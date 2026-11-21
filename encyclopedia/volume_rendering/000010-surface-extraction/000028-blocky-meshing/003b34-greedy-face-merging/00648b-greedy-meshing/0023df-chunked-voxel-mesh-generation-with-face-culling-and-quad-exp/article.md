# Chunked Voxel Mesh Generation with Face Culling and Quad Expansion

Also called: Chunk-based Greedy Meshing, FrontGreedyMesh, Greedy Meshing with Chunked Processing, TopGreedyMesh

## Overview
Chunked Voxel Mesh Generation with Face Culling and Quad Expansion is a CPU-based algorithm that converts discrete 3D voxel data into optimized triangle meshes by merging adjacent coplanar faces of identical voxel types into larger quads [S1, S2, S4, S6, unofficial]. It solves the problem of excessive vertex and draw call counts in voxel worlds by reducing primitive density while preserving sharp, axis-aligned surfaces suitable for architectural geometry [S6, unofficial]. It differs from standard face culling (which only removes hidden faces) by actively merging contiguous blocks into maximal axis-aligned rectangles, and it differs from Marching Cubes by maintaining discrete voxel boundaries rather than smoothing surfaces [S2, S6, unofficial]. This method builds on [Greedy Meshing](3519-greedy-meshing.md) by processing data in fixed-size chunks and handling inter-chunk visibility.

## How it works
**Inputs and Outputs:**
*   **Input:** A 1D array of voxel data representing a chunk (e.g., `blockArray1D`), neighbor chunk data for boundary checks, and metadata (chunk size, block size, world offset) [S1, S5, S6, unofficial].
*   **Output:** A mesh structure containing vertices, indices, and optionally UVs/normals [S2, S6, unofficial].

**Data Structures:**
*   **Voxel Array:** A 1D array where each element represents a voxel. The index is calculated as $index = j + (i \times ChunkSize) + (k \times ChunkSize^2)$, where $i$ is X, $j$ is Y, and $k$ is Z [S1, S4, unofficial].
*   **Visit Arrays:** Boolean arrays (e.g., `visitXN`, `visitXP`, `visitYN`, `visitYP`, `visitZN`, `visitZP`) of size $ChunkSize^3$ to track which faces have already been merged into a quad, preventing double-counting [S1, S4, unofficial].
*   **Vertex Packing (Optional):** Vertices can be packed into a single 32-bit integer to save memory. Layout: x (6 bits), y (6 bits), z (6 bits), voxel_id (8 bits), face_id (3 bits), ao_id (2 bits), flip_id (1 bit) [S3, unofficial]. Alternatively, explicit structs with position, normal, color, and material are used [S6, unofficial].
*   **Face Type Encoding:** Face normals can be encoded in the upper bits of vertex attributes (e.g., `yp=0<<27`, `yn=1<<27`, etc.) [S4, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Initialize empty vertex and index lists [S2, S6, unofficial].
    *   Initialize `visit` arrays to false [S1, S4, unofficial].
    *   If using heightmaps, initialize min/max altitude arrays to optimize Y-range iteration [S1, unofficial].

2.  **Voxel Iteration:**
    *   Iterate through all voxels in the chunk. The order varies by implementation:
        *   Z (k), X (i), Y (j) [S1, unofficial].
        *   Z, Y, X [S2, unofficial].
        *   X, Y, Z [S3, S6, unofficial].
    *   For each voxel at $(i, j, k)$, check if it is solid (index > 0) [S1, S3, S6, unofficial].

3.  **Face Visibility Check:**
    *   For each of the 6 face directions (±X, ±Y, ±Z), determine if the face is visible. A face is visible if:
        *   The adjacent voxel is empty (index == 0) [S1, S3, S6, unofficial].
        *   **Boundary Handling:** If the voxel is at a chunk boundary, check the neighbor chunk.
            *   If the neighbor chunk is null or outside the world bounds, the face is visible [S4, S6, unofficial].
            *   If the neighbor chunk exists, check the corresponding voxel in the neighbor chunk. For example, for the X- face at chunk boundary $X=0$, check `neighborChunk.data[31 * ChunkSize + j + k * ChunkSize^2]` [S4, unofficial].
            *   **Exception:** For the Top (Y+) face, do not cull based on the maximum Y chunk boundary, as players may move above the map [S4, unofficial].

4.  **Face Culling and Merging (Quad Expansion):**
    *   If a face is visible and not yet visited (check `visit` array):
        *   Mark the current voxel's face as visited [S1, S4, unofficial].
        *   **Expand along the secondary axis:**
            *   For X-faces (±X), expand along the Z-axis (and then Y-axis) [S1, S4, unofficial].
            *   For Y-faces (±Y), expand along the X-axis (and then Z-axis) [S4, unofficial].
            *   For Z-faces (±Z), expand along the X-axis (and then Y-axis) [S4, unofficial].
            *   *Note:* Some implementations only expand along one axis (e.g., Z for top faces) [S5, unofficial].
        *   **Expansion Logic:**
            *   Iterate in the secondary direction (e.g., for X- face, iterate Z from $k$ to $ChunkSize$).
            *   Check if the neighbor voxel has the same type (and health/texture) [S4, unofficial].
            *   Check if the neighbor's face in the same direction is also visible and not visited [S1, S4, unofficial].
            *   If both conditions are met, mark the neighbor's face as visited and continue [S1, S4, unofficial].
            *   If not, stop expansion. The length of the merge is the number of consecutive matching voxels [S1, S4, S5, unofficial].
        *   **Generate Quad:**
            *   Create a quad spanning the merged area.
            *   Calculate vertices based on the start position, the merged width/length, and the face normal [S5, S6, unofficial].
            *   Calculate UVs and normals [S2, S6, unofficial].
            *   Add vertices and indices to the mesh. Indices are typically two triangles forming a quad: `[0, 1, 2, 0, 2, 3]` or `[2, 1, 0, 3, 2, 0]` depending on winding order [S2, S5, unofficial].

5.  **Finalization:**
    *   Convert lists to arrays [S2, unofficial].
    *   Recalculate normals and tangents if necessary [S2, unofficial].
    *   Optimize the mesh (e.g., vertex cache optimization) [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_chunk_mesh(chunk_voxels, chunk_size, neighbor_chunks):
    vertices = []
    indices = []
    # Initialize visit arrays for all 6 faces
    visit = {
        'xn': [False] * (chunk_size ** 3),
        'xp': [False] * (chunk_size ** 3),
        'yn': [False] * (chunk_size ** 3),
        'yp': [False] * (chunk_size ** 3),
        'zn': [False] * (chunk_size ** 3),
        'zp': [False] * (chunk_size ** 3)
    }

    # Iterate through all voxels
    for k in range(chunk_size):  # Z
        for i in range(chunk_size):  # X
            for j in range(chunk_size):  # Y
                voxel = chunk_voxels[i + j * chunk_size + k * chunk_size ** 2]
                if voxel.index == 0:
                    continue

                # Check X- face
                if is_face_visible(i, j, k, 'xn', chunk_voxels, chunk_size, neighbor_chunks):
                    if not visit['xn'][i + j * chunk_size + k * chunk_size ** 2]:
                        length_z, length_y = expand_face(i, j, k, 'xn', chunk_voxels, chunk_size, visit['xn'])
                        add_quad_to_mesh(vertices, indices, i, j, k, 'xn', length_z, length_y, voxel)
                        mark_visited(i, j, k, 'xn', chunk_voxels, chunk_size, visit['xn'], length_z, length_y)

                # Check other 5 faces similarly...
                # (Code omitted for brevity, follows same pattern)

    return vertices, indices

def is_face_visible(i, j, k, face, voxels, size, neighbors):
    # Determine neighbor coordinates
    if face == 'xn':
        if i == 0:
            if neighbors['xn'] is None: return True
            # Check neighbor chunk voxel
            return neighbors['xn'].get_voxel(31, j, k).index == 0
        else:
            return voxels[i - 1 + j * size + k * size ** 2].index == 0
    # ... handle other faces similarly
    return False

def expand_face(i, j, k, face, voxels, size, visit_array):
    # Example for X- face, expanding along Z then Y
    length_z = 1
    while k + length_z < size:
        neighbor_idx = i + j * size + (k + length_z) * size ** 2
        if voxels[neighbor_idx].index != voxels[i + j * size + k * size ** 2].index:
            break
        if visit_array[neighbor_idx]:
            break
        length_z += 1
    
    length_y = 1
    while j + length_y < size:
        # Check if the entire column in Z matches
        for dz in range(length_z):
            neighbor_idx = i + (j + length_y) * size + (k + dz) * size ** 2
            if voxels[neighbor_idx].index != voxels[i + j * size + k * size ** 2].index:
                return length_z, length_y
            if visit_array[neighbor_idx]:
                return length_z, length_y
        length_y += 1
        
    return length_z, length_y

def add_quad_to_mesh(vertices, indices, i, j, k, face, length_z, length_y, voxel):
    # Calculate world positions
    # ...
    # Add 4 vertices
    # Add 6 indices
    pass

def mark_visited(i, j, k, face, voxels, size, visit_array, length_z, length_y):
    # Mark all visited voxels in the merged area
    for dy in range(length_y):
        for dz in range(length_z):
            idx = i + (j + dy) * size + (k + dz) * size ** 2
            visit_array[idx] = True
```

## Subtopics

### [Chunk-based Marching Cubes with Face Combination](4158-chunk-based-marching-cubes-with-face-combination.md)
Processes chunks to generate triangle meshes and combines adjacent coplanar faces into quads. Chosen when smooth surfaces are needed but draw calls must be reduced.

### [Greedy Meshing (Front Face)](16000-greedy-meshing-front-face.md)
Generates quads specifically for the front-facing (positive Z) surfaces of a voxel chunk. Chosen when only front-facing optimization is required or for specific rendering passes.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the chunk, as each voxel is visited a constant number of times. The merging step adds overhead but reduces the final mesh size.
*   **Memory Complexity:** $O(N)$ for the voxel data and $O(F)$ for the output mesh, where $F$ is the number of faces. The visit arrays add $O(N)$ memory [S1, S4, unofficial].
*   **Performance:** On a CPU, a 64³ chunk (~50-100K triangles) can be generated in ~2ms [S6, unofficial].
*   **Strengths:** Significantly reduces vertex and triangle count for large solid volumes compared to naive face culling [S2, unofficial]. Produces sharp, axis-aligned surfaces ideal for architectural voxel art [S6, unofficial].
*   **Limitations:** Does not smooth surfaces, making it less suitable for organic shapes [S6, unofficial]. Requires careful handling of chunk boundaries and neighbor data [S4, unofficial]. The greedy approach may not produce the globally optimal mesh (e.g., it might miss merges that require a different iteration order) [S2, unofficial].
*   **When to Use:** Static voxel worlds, architectural designs, terrain generation where performance is critical and sharp edges are desired.
*   **When Not to Use:** Organic terrain requiring smooth surfaces (use Marching Cubes), dynamic scenes with frequent voxel changes where re-meshing overhead is prohibitive.

## Background
The algorithm originates from the need to optimize voxel rendering in games like Minecraft. It extends 2D greedy meshing techniques to 3D by processing voxel chunks. Key implementations include the Vercidium Engine [S1, S4, unofficial], Unity-based voxel engines [S2, S5, unofficial], and Python-based renderers [S3, S6, unofficial]. The method is widely used in voxel game development to balance visual fidelity with rendering performance.

## Sources
[S1] vercidium-patreon/meshing, https://github.com/vercidium-patreon/meshing, fetched 2026-10-26 (unofficial)
[S2] sm03854/Greedy-Voxel-Meshing, https://github.com/sm03854/Greedy-Voxel-Meshing, fetched 2026-10-26 (unofficial)
[S3] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
[S4] Vercidium/voxel-mesh-generation, https://github.com/Vercidium/voxel-mesh-generation, fetched 2026-10-26 (unofficial)
[S5] BloodyFish/UnityVoxelEngine, https://github.com/BloodyFish/UnityVoxelEngine, fetched 2026-10-27 (unofficial)
[S6] elyssov/prometheus-engine, https://github.com/elyssov/prometheus-engine, fetched 2026-10-28 (unofficial)
