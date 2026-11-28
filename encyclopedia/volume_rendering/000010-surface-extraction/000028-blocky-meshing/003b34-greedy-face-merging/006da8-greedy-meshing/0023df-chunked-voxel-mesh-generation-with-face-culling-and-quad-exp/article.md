# Chunked Voxel Mesh Generation with Face Culling and Quad Expansion

Also called: Chunk-based Greedy Meshing, FrontGreedyMesh, Greedy Meshing (Voxel Surface Extraction), Greedy Meshing with Chunked Processing, TopGreedyMesh

## Overview
Chunked Voxel Mesh Generation with Face Culling and Quad Expansion is a CPU-based algorithm that converts discrete 3D voxel data into optimized triangle meshes by merging adjacent coplanar faces of identical voxel types into larger quads [S1, S2, S4, S6, unofficial]. It solves the problem of excessive vertex and draw call counts in voxel worlds by reducing primitive density while preserving sharp, axis-aligned surfaces suitable for architectural geometry [S6, unofficial]. It differs from standard face culling (which only removes hidden faces) by actively merging contiguous blocks into maximal axis-aligned rectangles, and it differs from Marching Cubes by maintaining discrete voxel boundaries rather than smoothing surfaces [S2, S6, unofficial]. This method builds on [Greedy Meshing](18808-greedy-meshing.md) by processing data in fixed-size chunks and handling inter-chunk visibility [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 1D array of voxel data representing a chunk (e.g., `blockArray1D`), neighbor chunk data for boundary checks, and metadata (chunk size, block size, world offset) [S1, S5, S6, unofficial].
*   **Output:** A mesh structure containing vertices, indices, and optionally UVs/normals [S2, S6, unofficial].

**Data Structures:**
*   **Voxel Array:** A 1D array where each element represents a voxel. The index is calculated as $index = j + (i \times ChunkSize) + (k \times ChunkSize^2)$, where $i$ is X, $j$ is Y, and $k$ is Z [S2, unofficial].
*   **Visit Arrays:** Boolean arrays (e.g., `visitXN`, `visitXP`, `visitYN`, `visitYP`, `visitZN`, `visitZP`) of size $ChunkSize^3$ to track which faces have already been merged into a quad, preventing double-counting [S1, S4, unofficial].
*   **Vertex Packing (Optional):** Vertices can be packed into a single 32-bit integer to save memory. Layout: x (6 bits), y (6 bits), z (6 bits), voxel_id (8 bits), face_id (3 bits), ao_id (2 bits), flip_id (1 bit) [S4, unofficial]. Alternatively, explicit structs with position, normal, color, and material are used [S6, unofficial].
*   **Face Type Encoding:** Face normals can be encoded in the upper bits of vertex attributes (e.g., `yp=0<<27`, `yn=1<<27`, etc.) [S4, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Initialize empty vertex and index lists [S2, S6, unofficial].
    *   Initialize `visit` arrays to false [S1, S4, unofficial].
    *   If using heightmaps, initialize min/max altitude arrays to optimize Y-range iteration [S1, unofficial].

2.  **Voxel Iteration:**
    *   Iterate through all voxels in the chunk. The order varies by implementation:
        *   Z (k), X (i), Y (j) [S2, unofficial].
        *   Z, Y, X [S3, unofficial].
        *   X, Y, Z [S6, unofficial].
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
            *   For X-faces (±X), expand along the Z-axis (and then Y-axis) [S2, unofficial].
            *   For Y-faces (±Y), expand along the X-axis (and then Z-axis) [S5, unofficial].
            *   For Z-faces (±Z), expand along the X-axis (and then Y-axis) [S5, unofficial].
            *   *Note:* Some implementations only expand along one axis (e.g., Z for top faces) [S6, unofficial].
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
def generate_chunk_mesh(chunk_voxels, chunk_size, neighbor_chunks, block_size, world_offset):
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

    # Iterate through all voxels in Z, X, Y order [S2, unofficial]
    for k in range(chunk_size):
        for i in range(chunk_size):
            for j in range(chunk_size):
                idx = j + (i * chunk_size) + (k * chunk_size ** 2)
                voxel = chunk_voxels[idx]
                if voxel.index == 0:
                    continue

                # Check X- face
                if is_face_visible(i, j, k, 'xn', chunk_voxels, chunk_size, neighbor_chunks):
                    if not visit['xn'][idx]:
                        length_z, length_y = expand_face(i, j, k, 'xn', chunk_voxels, chunk_size, visit['xn'])
                        add_quad_to_mesh(vertices, indices, i, j, k, 'xn', length_z, length_y, voxel, block_size, world_offset)
                        mark_visited(i, j, k, 'xn', chunk_size, visit['xn'], length_z, length_y)

                # Check X+ face
                if is_face_visible(i, j, k, 'xp', chunk_voxels, chunk_size, neighbor_chunks):
                    if not visit['xp'][idx]:
                        length_z, length_y = expand_face(i, j, k, 'xp', chunk_voxels, chunk_size, visit['xp'])
                        add_quad_to_mesh(vertices, indices, i, j, k, 'xp', length_z, length_y, voxel, block_size, world_offset)
                        mark_visited(i, j, k, 'xp', chunk_size, visit['xp'], length_z, length_y)

                # Check Y- face
                if j > 0 and is_face_visible(i, j, k, 'yn', chunk_voxels, chunk_size, neighbor_chunks):
                    if not visit['yn'][idx]:
                        length_x, length_z = expand_face_y(i, j, k, 'yn', chunk_voxels, chunk_size, visit['yn'])
                        add_quad_to_mesh(vertices, indices, i, j, k, 'yn', length_x, length_z, voxel, block_size, world_offset)
                        mark_visited_y(i, j, k, 'yn', chunk_size, visit['yn'], length_x, length_z)

                # Check Y+ face
                if is_face_visible(i, j, k, 'yp', chunk_voxels, chunk_size, neighbor_chunks):
                    if not visit['yp'][idx]:
                        length_x, length_z = expand_face_y(i, j, k, 'yp', chunk_voxels, chunk_size, visit['yp'])
                        add_quad_to_mesh(vertices, indices, i, j, k, 'yp', length_x, length_z, voxel, block_size, world_offset)
                        mark_visited_y(i, j, k, 'yp', chunk_size, visit['yp'], length_x, length_z)

                # Check Z- face
                if k > 0 and is_face_visible(i, j, k, 'zn', chunk_voxels, chunk_size, neighbor_chunks):
                    if not visit['zn'][idx]:
                        length_x, length_y = expand_face_z(i, j, k, 'zn', chunk_voxels, chunk_size, visit['zn'])
                        add_quad_to_mesh(vertices, indices, i, j, k, 'zn', length_x, length_y, voxel, block_size, world_offset)
                        mark_visited_z(i, j, k, 'zn', chunk_size, visit['zn'], length_x, length_y)

                # Check Z+ face
                if is_face_visible(i, j, k, 'zp', chunk_voxels, chunk_size, neighbor_chunks):
                    if not visit['zp'][idx]:
                        length_x, length_y = expand_face_z(i, j, k, 'zp', chunk_voxels, chunk_size, visit['zp'])
                        add_quad_to_mesh(vertices, indices, i, j, k, 'zp', length_x, length_y, voxel, block_size, world_offset)
                        mark_visited_z(i, j, k, 'zp', chunk_size, visit['zp'], length_x, length_y)

    return vertices, indices

def is_face_visible(i, j, k, face, voxels, size, neighbors):
    if face == 'xn':
        if i == 0:
            if neighbors['xn'] is None: return True
            return neighbors['xn'].voxels[31 + j * size + k * size ** 2].index == 0
        return voxels[i - 1 + j * size + k * size ** 2].index == 0
    elif face == 'xp':
        if i == size - 1:
            if neighbors['xp'] is None: return True
            return neighbors['xp'].voxels[j * size + k * size ** 2].index == 0
        return voxels[i + 1 + j * size + k * size ** 2].index == 0
    elif face == 'yn':
        if j == 0:
            if neighbors['yn'] is None: return True
            return neighbors['yn'].voxels[i + 31 * size + k * size ** 2].index == 0
        return voxels[i + (j - 1) * size + k * size ** 2].index == 0
    elif face == 'yp':
        # Do not cull based on max Y boundary as players can move above the map [S4, unofficial]
        if neighbors['yp'] is None: return True
        return neighbors['yp'].voxels[i + j * size + k * size ** 2].index == 0
    elif face == 'zn':
        if k == 0:
            if neighbors['zn'] is None: return True
            return neighbors['zn'].voxels[i + j * size + 31 * size ** 2].index == 0
        return voxels[i + j * size + (k - 1) * size ** 2].index == 0
    elif face == 'zp':
        if k == size - 1:
            if neighbors['zp'] is None: return True
            return neighbors['zp'].voxels[i + j * size].index == 0
        return voxels[i + j * size + (k + 1) * size ** 2].index == 0
    return False

def expand_face(i, j, k, face, voxels, size, visit_array):
    # Expands along Z then Y for X-faces [S2, unofficial]
    length_z = 1
    while k + length_z < size:
        n_idx = i + j * size + (k + length_z) * size ** 2
        if voxels[n_idx].index != voxels[i + j * size + k * size ** 2].index or visit_array[n_idx]:
            break
        length_z += 1
    
    length_y = 1
    while j + length_y < size:
        for dz in range(length_z):
            n_idx = i + (j + length_y) * size + (k + dz) * size ** 2
            if voxels[n_idx].index != voxels[i + j * size + k * size ** 2].index or visit_array[n_idx]:
                return length_z, length_y
        length_y += 1
    return length_z, length_y

def expand_face_y(i, j, k, face, voxels, size, visit_array):
    # Expands along X then Z for Y-faces [S5, unofficial]
    length_x = 1
    while i + length_x < size:
        n_idx = (i + length_x) + j * size + k * size ** 2
        if voxels[n_idx].index != voxels[i + j * size + k * size ** 2].index or visit_array[n_idx]:
            break
        length_x += 1
        
    length_z = 1
    while k + length_z < size:
        for dx in range(length_x):
            n_idx = (i + dx) + j * size + (k + length_z) * size ** 2
            if voxels[n_idx].index != voxels[i + j * size + k * size ** 2].index or visit_array[n_idx]:
                return length_x, length_z
        length_z += 1
    return length_x, length_z

def expand_face_z(i, j, k, face, voxels, size, visit_array):
    # Expands along X then Y for Z-faces [S5, unofficial]
    length_x = 1
    while i + length_x < size:
        n_idx = (i + length_x) + j * size + k * size ** 2
        if voxels[n_idx].index != voxels[i + j * size + k * size ** 2].index or visit_array[n_idx]:
            break
        length_x += 1
        
    length_y = 1
    while j + length_y < size:
        for dx in range(length_x):
            n_idx = (i + dx) + (j + length_y) * size + k * size ** 2
            if voxels[n_idx].index != voxels[i + j * size + k * size ** 2].index or visit_array[n_idx]:
                return length_x, length_y
        length_y += 1
    return length_x, length_y

def mark_visited(i, j, k, face, size, visit_array, length_z, length_y):
    for dy in range(length_y):
        for dz in range(length_z):
            idx = i + (j + dy) * size + (k + dz) * size ** 2
            visit_array[idx] = True

def mark_visited_y(i, j, k, face, size, visit_array, length_x, length_z):
    for dx in range(length_x):
        for dz in range(length_z):
            idx = (i + dx) + j * size + (k + dz) * size ** 2
            visit_array[idx] = True

def mark_visited_z(i, j, k, face, size, visit_array, length_x, length_y):
    for dx in range(length_x):
        for dy in range(length_y):
            idx = (i + dx) + (j + dy) * size + k * size ** 2
            visit_array[idx] = True

def add_quad_to_mesh(vertices, indices, i, j, k, face, length_x, length_y, voxel, block_size, world_offset):
    # Calculate center and extents based on face normal
    x_center = (i + length_x / 2.0) * block_size
    y_center = (j + length_y / 2.0) * block_size
    z_center = k * block_size
    
    half_x = (length_x * block_size) / 2.0
    half_y = (length_y * block_size) / 2.0
    
    # Define 4 corners relative to face normal
    if face in ('xn', 'xp'):
        corners = [
            (x_center, y_center - half_y, z_center - half_x),
            (x_center, y_center - half_y, z_center + half_x),
            (x_center, y_center + half_y, z_center + half_x),
            (x_center, y_center + half_y, z_center - half_x)
        ]
    elif face in ('yn', 'yp'):
        corners = [
            (x_center - half_x, y_center, z_center - half_x),
            (x_center - half_x, y_center, z_center + half_x),
            (x_center + half_x, y_center, z_center + half_x),
            (x_center + half_x, y_center, z_center - half_x)
        ]
    else: # zn, zp
        corners = [
            (x_center - half_x, y_center - half_y, z_center),
            (x_center - half_x, y_center + half_y, z_center),
            (x_center + half_x, y_center + half_y, z_center),
            (x_center + half_x, y_center - half_y, z_center)
        ]
        
    # Add vertices
    base_idx = len(vertices)
    for c in corners:
        vertices.append((c[0] + world_offset[0], c[1] + world_offset[1], c[2] + world_offset[2]))
        
    # Add 2 triangles (6 indices) with correct winding [S2, S5, unofficial]
    indices.extend([base_idx, base_idx + 1, base_idx + 2, base_idx, base_idx + 2, base_idx + 3])
```

## Subtopics

### [Chunk-based Marching Cubes with Face Combination](4158-chunk-based-marching-cubes-with-face-combination.md)
Processes chunks to generate triangle meshes and combines adjacent coplanar faces into quads. Chosen when smooth surfaces are needed but draw calls must be reduced.

### [Greedy Meshing (Front Face)](16000-greedy-meshing-front-face.md)
Generates quads specifically for the front-facing (positive Z) surfaces of a voxel chunk. Chosen when only front-facing optimization is required or for specific rendering passes.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the chunk, as each voxel is visited a constant number of times. The merging step adds overhead but reduces the final mesh size [S2, unofficial].
*   **Memory Complexity:** $O(N)$ for the voxel data and $O(F)$ for the output mesh, where $F$ is the number of faces. The visit arrays add $O(N)$ memory [S1, unofficial].
*   **Performance:** On a CPU, a 64³ chunk (~50-100K triangles) can be generated in ~2ms [S6, unofficial].
*   **Strengths:** Significantly reduces vertex and triangle count for large solid volumes compared to naive face culling [S2, unofficial]. Produces sharp, axis-aligned surfaces ideal for architectural voxel art [S6, unofficial].
*   **Limitations:** Does not smooth surfaces, making it less suitable for organic shapes [S6, unofficial]. Requires careful handling of chunk boundaries and neighbor data [S4, unofficial]. The greedy approach may not produce the globally optimal mesh (e.g., it might miss merges that require a different iteration order) [S2, unofficial].
*   **When to Use:** Static voxel worlds, architectural designs, terrain generation where performance is critical and sharp edges are desired [S6, unofficial].
*   **When Not to Use:** Organic terrain requiring smooth surfaces (use Marching Cubes), dynamic scenes with frequent voxel changes where re-meshing overhead is prohibitive [S6, unofficial].

## Background
The algorithm originates from the need to optimize voxel rendering in games like Minecraft [S8, unofficial]. It extends 2D greedy meshing techniques to 3D by processing voxel chunks [S5, unofficial]. Key implementations include the Vercidium Engine [S1, S4, unofficial], Unity-based voxel engines [S2, S5, unofficial], and Python-based renderers [S3, S6, unofficial]. The method is widely used in voxel game development to balance visual fidelity with rendering performance [S8, unofficial].

## Sources
[S1] vercidium-patreon/meshing, https://github.com/vercidium-patreon/meshing, fetched 2026-10-26 (unofficial)
[S2] sm03854/Greedy-Voxel-Meshing, https://github.com/sm03854/Greedy-Voxel-Meshing, fetched 2026-10-26 (unofficial)
[S3] StanislavPetrovV/Minecraft, https://github.com/StanislavPetrovV/Minecraft, fetched 2026-10-26 (unofficial)
[S4] Vercidium/voxel-mesh-generation, https://github.com/Vercidium/voxel-mesh-generation, fetched 2026-10-26 (unofficial)
[S5] BloodyFish/UnityVoxelEngine, https://github.com/BloodyFish/UnityVoxelEngine, fetched 2026-10-27 (unofficial)
[S6] elyssov/prometheus-engine, https://github.com/elyssov/prometheus-engine, fetched 2026-10-28 (unofficial)
[S7] WildPixelGames/voxelis, https://github.com/WildPixelGames/voxelis, fetched 2026-10-26 (unofficial)
[S8] Rendering voxel data efficiently, https://gamedev.stackexchange.com/questions/186911/rendering-voxel-data-efficiently, fetched 2026-10-26 (unofficial)
