# Chunk-based Voxel Rendering Pipeline

Also called: Batched Voxel Rendering Pipeline

## Overview
The Chunk-based Voxel Rendering Pipeline is a high-throughput object-order rendering method that aggregates voxel faces into geometric chunks, transforms them to world space, and rasterizes them using a Painter's Algorithm [S2, unofficial]. It differs from naive voxel geometry by batching visible faces into chunks and applying distance-based sorting and horizon clipping to manage depth ordering and visual effects, rather than rendering individual voxels independently [S2, unofficial].

## How it works
The pipeline processes voxel data in chunks, generating geometry, transforming it, sorting by depth, and rasterizing.

### Inputs and Outputs
*   **Input**: A voxel grid (e.g., `Grid` or `map_data`) containing block IDs or occupancy flags.
*   **Output**: Rasterized pixels on a screen surface or frame buffer.

### Data Structures
The method uses specific layouts for face buffers and chunk data.

**Face Buffer Layout**
The raw face data is stored in a numpy array with specific column slices [S2, unofficial]:
*   `V_COLS` (indices 0–11): 12 vertex coordinates (3 vertices per face, 4 floats per vertex? No, 3 vertices * 4 floats = 12. Or 3 vertices * 3 coords + padding? The note says "vertex columns". Given 12 columns for 3 vertices, it implies 4 floats per vertex, likely x,y,z,1 or similar padding).
*   `C_COLS` (indices 12–14): Center coordinates (x, y, z).
*   `RGB_COLS` (indices 15–17): RGB color values.
*   Indices 19 and 20: Chunk offsets `WX` and `WY`.

**Chunk Structure**
A chunk aggregates geometry for a region of the voxel grid [S1, unofficial]:
*   `filled[512][512][512]`: Boolean grid indicating occupancy.
*   `rendervert`: Vector of floats for vertex positions.
*   `indices`: Vector of unsigned integers for triangle indices.
*   `count`: Number of blocks rendered.
*   `BIndex`: Bounding box indices `[min_i, max_i, min_j, max_j, min_k, max_k]`.

### Steps

1.  **Chunk Instantiation**: Create a chunk object with a specified voxel dimension, position, and display flag [S1, unofficial].
    ```python
    chunk = Chunk(voxel_dim, position=(0, 0, 0), display=True)
    ```

2.  **Grid Population**: Fill the chunk's internal `filled` grid from the source `Grid` using triple nested loops over the dimensions `dim` [S1, unofficial].

3.  **Geometry Generation**:
    *   Calculate the bounding box (`BIndex`) of occupied voxels within the chunk [S1, unofficial].
    *   Iterate through the bounding box. For each occupied voxel, determine visible faces using a mask (e.g., checking neighbors) [S1, unofficial].
    *   Generate vertex and index data for visible faces. In the C++ implementation, this is done via `RenderFace` and `block.Render()` [S1, unofficial].
    *   Aggregate all raw face data from visible sections into a single array `raw_stacked_faces` [S2, unofficial]. If empty, return.

4.  **World Space Transformation**:
    *   Extract vertex columns from `raw_stacked_faces` [S2, unofficial].
    *   Add chunk offsets `WX` (from index 19) to vertex columns 0, 3, 6, 9 (x-coordinates) [S2, unofficial].
    *   Add chunk offsets `WY` (from index 20) to vertex columns 1, 4, 7, 10 (y-coordinates) [S2, unofficial].
    *   The z-coordinate transformation is not explicitly detailed in the Python notes but is implied by the center calculation [S2, unofficial].

5.  **Projection and Clipping**:
    *   **Horizon Clipping**: Calculate world-space centers `wx`, `wy`, `wz` from face data [S2, unofficial].
        $$ dx = wx - cam\_pos[0] $$
        $$ dy = wy - cam\_pos[1] $$
        $$ dz = wz - cam\_pos[2] $$
        $$ dist\_sq = dx^2 + dy^2 + dz^2 $$
        Limit is $(render\_dist + 0.5)^2$. Mask faces where $dist\_sq \le limit\_sq$ [S2, unofficial].
    *   **Projection**: Project vertices using camera parameters to get projected points `pts` and a projection mask `proj_mask` [S2, unofficial].
    *   If no points pass projection (`not np.any(proj_mask)`), return [S2, unofficial].

6.  **Sorting**:
    *   Sort faces by distance from far to near using `np.argsort(dist_sq)[::-1]` [S2, unofficial].
    *   Apply the sort order to `pts`, `faces`, `dist_sq`, and `wz` [S2, unofficial].

7.  **Visual Effects**:
    *   Apply effects such as fog and height shading based on colors, z-centers, euclidean distance, and render distance [S2, unofficial].
    *   Parameters include `fog_density`, `use_h_shading`, `height_shading_factor`, `height_fog_density`, `height_fog_max_z`, and `height_fog_fade` [S2, unofficial].

8.  **Rasterization**:
    *   **CPU-based (Python/Pygame)**: For each face, draw a polygon using the projected points and colors [S2, unofficial].
        ```python
        pygame.draw.polygon(surface, colors[i], pts[i])
        ```
    *   **GPU-based (C++/OpenGL)**:
        *   Allocate memory for vertices and indices [S1, unofficial].
        *   Create Vertex Buffer Object (VBO) and Index Buffer Object (IBO) [S1, unofficial].
        *   Define vertex layout (e.g., 4 floats per vertex) [S1, unofficial].
        *   Draw the mesh using `glDrawElements` [S1, unofficial].

### Pseudocode (Python-like)

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_chunk(chunk, grid, camera, render_dist):
    # 1. Populate grid
    chunk.filled = grid[chunk.bounds]
    
    # 2. Generate Geometry
    raw_faces = []
    for i, j, k in chunk.bounding_box():
        if not chunk.filled[i, j, k]:
            continue
        faces = generate_visible_faces(i, j, k, chunk.filled)
        for face in faces:
            # face contains 3 vertices, center, color
            raw_faces.append(face)
            
    if not raw_faces:
        return

    # 3. Transform to World Space
    # Assuming raw_faces is a structured array or list of dicts
    # Add chunk offsets to x and y coordinates
    for face in raw_faces:
        face.vertices[:, 0] += chunk.pos.x
        face.vertices[:, 1] += chunk.pos.y
        
    # 4. Horizon Clipping
    centers = np.array([f.center for f in raw_faces])
    dx = centers[:, 0] - camera.pos.x
    dy = centers[:, 1] - camera.pos.y
    dz = centers[:, 2] - camera.pos.z
    dist_sq = dx**2 + dy**2 + dz**2
    
    limit_sq = (render_dist + 0.5)**2
    mask = dist_sq <= limit_sq
    filtered_faces = [f for f, m in zip(raw_faces, mask) if m]
    filtered_dist_sq = dist_sq[mask]
    
    if not filtered_faces:
        return

    # 5. Project
    projected_pts = []
    proj_mask = []
    for face in filtered_faces:
        pts = project_vertices(face.vertices, camera)
        if pts is not None:
            projected_pts.append(pts)
            proj_mask.append(True)
        else:
            proj_mask.append(False)
            
    if not any(proj_mask):
        return
        
    # 6. Sort Far to Near
    # Re-apply mask to dist_sq
    valid_dist_sq = filtered_dist_sq[[m for m in proj_mask]]
    sort_indices = np.argsort(valid_dist_sq)[::-1]
    
    sorted_faces = [filtered_faces[i] for i in sort_indices]
    sorted_pts = [projected_pts[i] for i in sort_indices]
    
    # 7. Visual Effects
    colors = apply_visual_effects(sorted_faces, camera, render_dist)
    
    # 8. Rasterize
    for i, face in enumerate(sorted_faces):
        draw_polygon(surface, colors[i], sorted_pts[i])
```

### Parameters
*   `render_dist`: Distance limit for horizon clipping. Typical value: Not given in the sources.
*   `fog_density`: Density for fog effect. Typical value: Not given in the sources.
*   `height_shading_factor`: Factor for height-based shading. Typical value: Not given in the sources.

### Edge Cases and Pitfalls
*   **Empty Chunks**: If no voxels are visible or pass projection, the pipeline returns early [S2, unofficial].
*   **Precision**: Floating-point precision in world space transformation and distance calculations may affect sorting accuracy [S2, unofficial].
*   **Boundary Conditions**: The bounding box calculation must correctly handle chunks at the edge of the voxel grid [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Dominated by geometry generation ($O(N)$ where $N$ is the number of voxels in the chunk) and sorting ($O(M \log M)$ where $M$ is the number of visible faces).
*   **Memory Complexity**: Proportional to the number of visible faces in the chunk.
*   **Performance**: High throughput due to batching and culling [S2, unofficial].
*   **Strengths**: Efficient culling, support for visual effects, and depth sorting for transparency [S2, unofficial].
*   **Limitations**: Sorting overhead for large numbers of faces; potential z-fighting if not handled [S2, unofficial].
*   **When to Use**: Scenes with large volumes of transparent or semi-transparent voxels requiring correct depth ordering [S2, unofficial].
*   **When Not to Use**: Real-time applications with extremely high voxel counts where sorting becomes a bottleneck; opaque scenes might benefit from other methods like ray marching or ray tracing.

## Background
The Chunk-based Voxel Rendering Pipeline is implemented in the `3D-Mesh-Voxelizer` project [S1, unofficial] and the `pyvorengi-sdk-demo` [S2, unofficial]. It represents a practical approach to rendering voxel data by leveraging standard graphics pipeline stages (transform, clip, sort, rasterize) [S2, unofficial].

## Sources
[S1] 3D-Mesh-Voxelizer - GitHub, https://github.com/Talkative-Banana/3D-Mesh-Voxelizer, fetched 2026-10-26 (unofficial)
[S2] herbal1st/pyvorengi-sdk-demo, https://github.com/herbal1st/pyvorengi-sdk-demo, fetched 2026-10-27 (unofficial)
