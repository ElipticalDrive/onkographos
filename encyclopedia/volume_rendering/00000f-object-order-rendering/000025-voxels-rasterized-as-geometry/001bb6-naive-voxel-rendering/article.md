# Naive Voxel Rendering

Also called: Rasterization without LODs, VoxelShader::render

## Overview
Naive Voxel Rendering is an object-order volume rendering technique that rasterizes individual voxels as geometric primitives (typically cubes) without Level of Detail (LOD) reduction or spatial optimization [S1, S3, S4, unofficial]. It solves the problem of visualizing volumetric data by treating every occupied voxel as a distinct mesh element, projecting it directly onto the 2D image plane [S2, S4, unofficial]. This method differs from its siblings, such as [Binary Volume Rendering](1015-binary-volume-rendering.md) or [Sparse Voxel Rasterization](3032-sparse-voxel-rasterization.md), by explicitly generating geometry for every voxel rather than using ray-marching, thresholding, or hierarchical traversal, resulting in extreme simplicity at the cost of massive primitive counts [S1, S3, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A set of occupied voxel coordinates (e.g., `HashSet<IVec3>` or a dense array) and a camera configuration (view/projection matrices) [S3, S4, unofficial].
- **Output:** A 2D image where each voxel is rendered as a rasterized cube [S3, S4, unofficial].

**Data Structures:**
1.  **Voxel Grid/World:**
    -   Stores occupied voxel positions.
    -   Layout: A set of integer 3D coordinates (`IVec3`) or a dense array where existence is checked via a flag (e.g., `r > 0`) [S3, S4, unofficial].
    -   In sparse implementations, a `HashSet<IVec3>` is used to map integer coordinates to existence [S3, unofficial].
2.  **GPU Buffers (if instanced):**
    -   Position Buffer: `std::vector<glm::vec3>` containing world-space centers of voxels [S4, unofficial].
    -   Data Buffer: Optional, for color/normal attributes [S4, unofficial].

**Numbered Steps:**

1.  **Iterate over Voxel Coordinates:**
    Loop through all defined voxel positions. If using a sparse structure, iterate over the set keys [S3, unofficial]. If using a dense grid, iterate over the bounding box (e.g., $x, z \in [0, 32]$) [S3, unofficial].

2.  **Frustum Culling:**
    For each voxel, calculate its world position. Check if the voxel's Y coordinate (or full bounding box) exceeds the camera frustum borders. If outside, skip rendering for this voxel [S2, S4, unofficial].
    *Formula for World Position from Index (Dense Grid):*
    $$
    P_{world} = P_{min} + \frac{I_{voxel}}{N_{dim}} \times (P_{max} - P_{min})
    $$
    Where $I_{voxel}$ is the integer index, $N_{dim}$ is the dimension size, and $P_{min/max}$ are world-space bounds [S4, unofficial].

3.  **Back-face Culling (Optional):**
    If `cullInvisibleFaces` is enabled:
    a. Transform the voxel's normal to view space: $N_{view} = M_{view} \times N_{local}$ (treat normal as vector, $w=0$) [S2, unofficial].
    b. Calculate vector from voxel position to camera origin in view space: $V_{cam} = -P_{view}$ [S2, unofficial].
    c. Compute dot product: $D = N_{view} \cdot V_{cam}$ [S2, unofficial].
    d. If $D < 0$, the face is back-facing; return immediately [S2, unofficial].

4.  **Rasterization (Per-Voxel):**
    For each valid voxel, emit a cube primitive.
    a. **Vertex Generation:** A cube consists of 6 faces, 4 vertices each, 36 vertices total if not instanced [S3, unofficial].
    b. **Position Calculation:** For each vertex offset $(dx, dy, dz)$ relative to the voxel center:
       $$
       P_{view} = M_{view} \times (P_{world} + \frac{1}{2} \times V_{size} \times (dx, dy, dz))
       $$
       $$
       P_{clip} = M_{proj} \times P_{view}
       $$
       Set `gl_Position` to $P_{clip}$ [S2, S4, unofficial].
    c. **Attributes:** Set texture coordinates (e.g., $(0,0)$ to $(1,1)$) and color/normal [S2, S4, unofficial].
    d. **Draw Call:**
       -   *Naive:* Call `draw_cube` for each voxel, uploading 36 vertices per voxel per frame [S3, unofficial].
       -   *Instanced:* Upload all voxel positions to a VBO and issue a single `glDrawArraysInstanced` or `glDrawElementsInstanced` call [S4, unofficial].

5.  **Data Synchronization (Optional):**
    If the volume data is dynamic or stored on GPU:
    a. Pull volume texture data from GPU to CPU buffer [S4, unofficial].
    b. Update CPU-side voxel data structures [S4, unofficial].
    c. Reupload updated positions/data to GPU buffers using `glBufferData` with `GL_DYNAMIC_DRAW` [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_naive_voxels(voxel_positions, view_matrix, proj_matrix, voxel_size):
    """
    Renders voxels as individual cubes.
    voxel_positions: List of (x, y, z) integer coordinates.
    """
    for pos in voxel_positions:
        # 1. Frustum Culling (Simplified Y-check)
        if pos.y < camera_near or pos.y > camera_far:
            continue

        # 2. Calculate World Position
        # Assuming integer grid maps to world bounds
        world_pos = voxel_to_world(pos, voxel_size)

        # 3. Emit Cube Geometry
        # A cube has 6 faces. Each face is a quad (2 triangles, 4 vertices).
        # For simplicity, we define 4 corners of a face relative to center.
        offsets = [
            (-0.5, -0.5, -0.5), (0.5, -0.5, -0.5), (0.5, 0.5, -0.5), (-0.5, 0.5, -0.5), # Back
            (-0.5, -0.5, 0.5), (0.5, -0.5, 0.5), (0.5, 0.5, 0.5), (-0.5, 0.5, 0.5),     # Front
            # ... other faces omitted for brevity, total 36 vertices
        ]
        
        vertices = []
        for dx, dy, dz in offsets:
            local_pos = vec3(dx, dy, dz) * voxel_size
            view_pos = view_matrix * (world_pos + local_pos)
            clip_pos = proj_matrix * view_pos
            vertices.append(clip_pos)
        
        # 4. Rasterize
        # In a real engine, this would be a draw call.
        # Naive approach: Upload vertices to GPU and draw.
        upload_and_draw(vertices)
```

**Parameters:**
-   **Voxel Size:** World units per voxel. Determines the scale of the generated cubes [S4, unofficial].
-   **Frustum Bounds:** Camera near/far planes used for culling [S2, unofficial].

**Edge Cases and Pitfalls:**
-   **Performance:** Rendering scales linearly with voxel count. Increasing platform size causes massive performance drops because 36 vertices are uploaded and drawn per voxel every frame [S3, unofficial].
-   **Precision:** Using floating-point positions for grid alignment can cause gaps; integer-based coordinates (`IVec3`) are preferred for grid alignment [S3, unofficial].
-   **Degenerate Input:** Empty voxels (e.g., `r=0`) should be skipped or hidden by setting Y to a large value (e.g., 1,000,000) to avoid rendering artifacts [S4, unofficial].

## Subtopics

### [Per-voxel Projection (Object-order)](1816-per-voxel-projection-object-order.md)
Optimizes the object-order approach for Maximum Intensity Projection (MIP) by sorting voxels by value, allowing simple overwriting instead of blending. Choose when visualizing density/heat maps where occlusion is handled by value priority rather than geometry depth.

### [Instancing with Procedural/Shader Generation](2794-instancing-with-procedural-shader-generation.md)
Uses GPU instancing to render many voxels with a single draw call, reducing CPU-GPU transfer overhead. Choose when the voxel count is high but the geometry is uniform.

### [Sparse Voxel Rasterization](3032-sparse-voxel-rasterization.md)
Combines rasterization with sparse data structures (like SVOs) to skip empty space. Choose when the volume is sparse and memory bandwidth is a bottleneck.

## Operating characteristics
-   **Time Complexity:** $O(V)$, where $V$ is the number of voxels. Each voxel requires constant-time processing for culling and vertex generation [S3, unofficial].
-   **Memory Complexity:** $O(V)$ for storing voxel positions and geometry buffers [S3, S4, unofficial].
-   **Measured Performance:** Rendering individual cubes without LODs is "vastly inefficient" for small voxels due to the huge primitive count [S1, unofficial]. For a 32x32x1 platform, performance is acceptable, but increasing size causes severe bottlenecks due to vertex upload overhead (36 vertices per voxel) [S3, unofficial].
-   **Strengths:** Simple implementation; easy to debug; supports arbitrary voxel shapes if geometry is customized; no complex ray-marching logic required [S3, S4, unofficial].
-   **Limitations:** Extremely poor performance for large volumes; high CPU-GPU bandwidth usage; no inherent occlusion culling beyond frustum/back-face [S3, S4, unofficial].
-   **When to Use:** Prototyping, small-scale voxel art, or educational purposes where simplicity outweighs performance [S3, unofficial].
-   **When Not to Use:** Large-scale volumetric visualization, real-time applications with high voxel counts, or scenes requiring efficient occlusion handling [S1, S3, unofficial].

## Background
Naive Voxel Rendering originated as a basic technique in early voxel engines and educational tutorials, often serving as the first step in implementing voxel graphics [S3, unofficial]. It is frequently contrasted with more advanced methods like [Binary Volume Rendering](1015-binary-volume-rendering.md) or [Sparse Voxel Rasterization](3032-sparse-voxel-rasterization.md) due to its lack of optimization [S1, S3, unofficial]. The method is documented in community-driven resources such as "Make yourself a Voxel Engine THIS weekend" [S3, unofficial] and various GitHub repositories like VoxelRT [S1, unofficial] and Cloud-Renderer [S4, unofficial].

## Sources
[S1] VoxelRT, https://github.com/dubiousconst282/VoxelRT, fetched 2026-10-24 (unofficial)
[S2] svo_renderer, https://github.com/mmalesadev/svo_renderer, fetched 2026-10-26 (unofficial)
[S3] Make yourself a Voxel Engine THIS weekend, https://daymare.net/blogs/voxel-engine-in-a-weekend/, fetched 2026-10-26 (unofficial)
[S4] Cloud-Renderer, https://github.com/jaafersheriff/Cloud-Renderer, fetched 2026-10-27 (unofficial)
