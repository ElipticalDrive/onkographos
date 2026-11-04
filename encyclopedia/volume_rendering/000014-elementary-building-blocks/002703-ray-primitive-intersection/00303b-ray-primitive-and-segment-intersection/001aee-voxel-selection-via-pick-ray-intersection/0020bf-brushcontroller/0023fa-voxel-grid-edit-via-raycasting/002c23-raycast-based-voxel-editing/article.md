# Raycast-based Voxel Editing

Also called: Chunk Noise Modification, Marching Cubes Voxel Sculpting

## Overview
This method modifies voxel density values in a local radius around a raycast hit point and triggers chunk regeneration [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Camera screen coordinates (mouse position), current voxel chunk grid state, edit parameters (radius, edit value) [S1, unofficial].
- **Output:** Modified voxel density values in affected chunks, regenerated mesh geometry for those chunks [S1, unofficial].

**Data Structures:**
- **Chunk Grid:** A 3D array of `ChunkMatrix` objects indexed by global chunk coordinates `[S1, unofficial]`.
- **Hit Point:** A `Vector3` position in world space where the ray intersects a mesh `[S1, unofficial]`.
- **Chunk Indices:** A 3-tuple `(cx, cy, cz)` identifying the chunk containing the hit point, computed via `GetCurrentChunkFromPos` `[S1, unofficial]`.
- **Local Voxel Indices:** A 3-tuple `(vx, vy, vz)` identifying the specific voxel within the chunk, computed via `GetCurrentCubeInChunkFromPos` `[S1, unofficial]`.

**Steps:**
1. **Ray Casting:** Cast a ray from the camera through the mouse screen position with a maximum distance of 20 units [S1, unofficial].
2. **Hit Validation:** Check if the ray hits a transform tagged "Mesh" [S1, unofficial].
3. **Position Conversion:** Extract the world-space hit point and compute the global chunk indices using `GetCurrentChunkFromPos(hitPos)` [S1, unofficial].
4. **Local Index Calculation:** Compute the local voxel indices within the identified chunk using `GetCurrentCubeInChunkFromPos(hitPos)` [S1, unofficial].
5. **Direction Determination:** Determine the edit direction: `dir = 1` for left mouse button (add), `dir = -1` for right mouse button (subtract) [S1, unofficial].
6. **Voxel Modification:** Call `SetNoiseValues` on the hit chunk, passing the local voxel indices, `dir * editValue`, and radius to modify noise in the affected area [S1, unofficial].
7. **Mesh Regeneration:** Call `UpdateSurroundingChunks` with the hit chunk's global indices to regenerate affected meshes [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
CHUNK_SIZE = 16  # [G] Typical chunk dimension; adjust to actual grid layout

def handle_voxel_edit(screen_pos, camera, chunks, edit_value, radius, mouse_button):
    # Step 1: Cast ray
    ray_origin, ray_direction = compute_camera_ray(screen_pos, camera)
    max_distance = 20.0
    hit = cast_ray(ray_origin, ray_direction, max_distance)

    # Step 2: Hit validation
    if hit is None or hit.get("tag") != "Mesh":
        return

    hit_pos = hit["point"]

    # Step 3: Position to chunk indices
    cx = int(hit_pos[0]) // CHUNK_SIZE
    cy = int(hit_pos[1]) // CHUNK_SIZE
    cz = int(hit_pos[2]) // CHUNK_SIZE

    # Step 4: Position to local voxel indices
    vx = ((int(hit_pos[0]) % CHUNK_SIZE) + CHUNK_SIZE) % CHUNK_SIZE
    vy = ((int(hit_pos[1]) % CHUNK_SIZE) + CHUNK_SIZE) % CHUNK_SIZE
    vz = ((int(hit_pos[2]) % CHUNK_SIZE) + CHUNK_SIZE) % CHUNK_SIZE

    # Step 5: Direction
    dir = 1 if mouse_button == "left" else -1

    # Step 6: Modify voxels
    hit_chunk = chunks[cx, cy, cz]
    set_noise_values(hit_chunk, vx, vy, vz, dir * edit_value, radius)

    # Step 7: Regenerate meshes
    update_surrounding_chunks(chunks, cx, cy, cz)

def compute_camera_ray(screen_pos, camera):
    # [G] Standard perspective projection ray generation
    fov = camera.fov
    aspect = camera.aspect
    ndc_x = (2.0 * screen_pos[0] / camera.width) - 1.0
    ndc_y = (2.0 * screen_pos[1] / camera.height) - 1.0
    ray_dir = [ndc_x * aspect, ndc_y, -1.0]
    ray_dir = normalize(ray_dir)
    return camera.position, ray_dir

def cast_ray(origin, direction, max_dist):
    # [G] Standard ray-mesh intersection; returns hit dict or None
    # Implementation depends on scene graph structure
    return None

def set_noise_values(chunk, center_vx, center_vy, center_vz, value, radius):
    # [G] Standard spherical brush implementation not detailed in sources
    half_r = int(radius)
    for dx in range(-half_r, half_r + 1):
        for dy in range(-half_r, half_r + 1):
            for dz in range(-half_r, half_r + 1):
                x, y, z = center_vx + dx, center_vy + dy, center_vz + dz
                if not (0 <= x < CHUNK_SIZE and 0 <= y < CHUNK_SIZE and 0 <= z < CHUNK_SIZE):
                    continue
                dist_sq = dx*dx + dy*dy + dz*dz
                if dist_sq <= radius * radius:
                    chunk.noise_matrix[x, y, z] += value

def update_surrounding_chunks(chunks, cx, cy, cz):
    # [G] Regenerates mesh for the hit chunk and its 26 neighbors
    for dx in range(-1, 2):
        for dy in range(-1, 2):
            for dz in range(-1, 2):
                nx, ny, nz = cx + dx, cy + dy, cz + dz
                if 0 <= nx < len(chunks) and 0 <= ny < len(chunks[0]) and 0 <= nz < len(chunks[0][0]):
                    regenerate_mesh(chunks[nx, ny, nz])
```

**Parameters:**
- **Max Ray Distance:** 20 units [S1, unofficial].
- **Edit Value:** Not given in the sources. [G] Typical values depend on density scale.
- **Radius:** Not given in the sources. [G] Typical values are 1-5 voxels.

**Edge Cases:**
- **Ray Miss:** If the ray does not hit a "Mesh" tagged object, the function returns immediately without modifying data [S1, unofficial].
- **Chunk Boundaries:** `GetCurrentCubeInChunkFromPos` must correctly handle voxels on chunk boundaries using modulo arithmetic [S1, unofficial].
- **Precision:** Floating-point precision in `hit.point` and subsequent index calculations can lead to jitter if not handled carefully [G]. Rounding or epsilon checks may be needed.

## Operating characteristics
- **Time Complexity:** Raycasting is $O(1)$ per interaction. Voxel modification is $O(r^3)$ where $r$ is the brush radius. Mesh regeneration is $O(v)$ where $v$ is the number of voxels in the affected chunk and its immediate neighbors [G].
- **Memory Complexity:** $O(1)$ auxiliary memory for the edit operation, assuming in-place modification of the chunk's noise matrix [G].
- **Strengths:** Interactive, immediate feedback. Simple to implement [G].
- **Limitations:** Limited to local edits. Performance may degrade if many chunks are regenerated simultaneously [G].
- **When to Use:** Interactive voxel sculpting tools [G].
- **When Not to Use:** Large-scale, non-interactive terrain generation [G].

## Background
This method is described in an unofficial voxel sculpting implementation that modifies noise density values based on raycast hits and regenerates chunks [S1, unofficial]. The specific implementation details are from a GitHub repository by millerc3.

## Sources
[S1] millerc3/Marching-Cubes, https://github.com/millerc3/Marching-Cubes, fetched 2026-10-26 (unofficial)
