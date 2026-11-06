# Dynamic Orthographic Shadow Map Generation

Also called: Shadow Depth Pass, Voxel Shadow Map

## Overview
Generates a per-frame depth texture from a directional light's perspective to compute shadow occlusion for voxel models [S1, unofficial]. It differs from ray-marched voxel shadows by pre-computing the highest occupied voxel Y-coordinate per (X, Z) column into a 2D texture, allowing lighting calculations to use simple depth comparisons without traversal [S1, unofficial]. This method builds on the [2D Voxel Shadow Map](5769-2d-voxel-shadow-map.md) concept by dynamically updating the depth buffer every frame to reflect moving objects or changing geometry.

## How it works
**Inputs and Outputs:**
*   **Input:** Scene entities with bounding boxes, directional light direction (from scene trie), current viewport dimensions.
*   **Output:** A 1024x1024 depth texture (`shadowDepthTexture`) attached to an OpenGL framebuffer object (`shadowFramebuffer`).

**Data Structures:**
*   `shadowFramebuffer`: OpenGL FBO ID bound to `GL_FRAMEBUFFER`.
*   `shadowDepthTexture`: 2D texture ID, internal format `GL_DEPTH_COMPONENT`, type `GL_FLOAT`, dimensions 1024x1024, attached to `GL_DEPTH_ATTACHMENT`.
*   `shadowMatrix`: `Matrix4x4` (4x4 column-major float array).
*   `AABB`: Axis-Aligned Bounding Box defined by `min` (Vector3) and `max` (Vector3).

**Parameters:**
*   `shadowTextureWidth`, `shadowTextureHeight`: 1024 pixels.
*   Orthographic Projection Bounds: `right=210`, `left=-210`, `top=210`, `bottom=-210`, `near=210`, `far=-210`.
*   Z-clamp Range: `[0, 255]`.

**Steps:**
1.  Initialize global bounding box vectors `min` to $(\infty, \infty, \infty)$ and `max` to $(-\infty, -\infty, -\infty)$ [S1, unofficial].
2.  Retrieve the model's bounding box and screen-space bounds via `GetModelBounds` [S1, unofficial].
3.  If the object's screen-space bounds fall completely outside the viewport, skip it to avoid unnecessary updates [S1, unofficial].
4.  Expand the global AABB to encapsulate the current model's bounds:
    $$ min = \min(min, model\_min) $$
    $$ max = \max(max, model\_max) $$
5.  If only one object is shadowed (i.e., `min == max`), collapse the bounds to that object's centroid to prevent degenerate zero-volume matrices [S1, unofficial].
6.  Clamp the Z-axis bounds of the AABB to the range $[0, 255]$ before matrix construction [S1, unofficial].
7.  Initialize the shadow matrix to identity:
    $$ I = \begin{bmatrix} 1 & 0 & 0 & 0 \\ 0 & 1 & 0 & 0 \\ 0 & 0 & 1 & 0 \\ 0 & 0 & 0 & 1 \end{bmatrix} $$
8.  Compute the view matrix centered at the AABB centroid $C = \frac{min + max}{2}$ [S1, unofficial].
9.  Compute the orthographic projection matrix $P$ using the fixed bounds:
    $$ P = \begin{bmatrix} \frac{2}{right-left} & 0 & 0 & -\frac{right+left}{right-left} \\ 0 & \frac{2}{top-bottom} & 0 & -\frac{top+bottom}{top-bottom} \\ 0 & 0 & -\frac{2}{far-near} & -\frac{far+near}{far-near} \\ 0 & 0 & 0 & 1 \end{bmatrix} $$
    With values: $right=210, left=-210, top=210, bottom=-210, near=210, far=-210$ [S1, unofficial].
10. Multiply the view matrix by the projection matrix to form the final shadow matrix [S1, unofficial].
11. Bind `shadowFramebuffer`, set viewport to 1024x1024, enable depth testing, and clear the depth buffer [S1, unofficial].
12. Activate the shadow rendering shader (index 4), fetch the sun direction from the scene trie, and cache uniform locations [S1, unofficial].
13. Upload the computed shadow matrix to the shader [S1, unofficial].
14. Iterate through entities again and render them to the depth texture [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_shadow_map(entities, light_direction, viewport_bounds):
    # 1. Initialize bounds
    min_bound = Vector3(INFINITY, INFINITY, INFINITY)
    max_bound = Vector3(-INFINITY, -INFINITY, -INFINITY)
    object_count = 0

    # 2-4. Accumulate bounds
    for entity in entities:
        model_min, model_max = entity.get_bounds()
        screen_min, screen_max = get_model_bounds(entity, viewport_bounds)
        
        # 3. Cull outside viewport
        if not intersects_viewport(screen_min, screen_max, viewport_bounds):
            continue
            
        min_bound = min(min_bound, model_min)
        max_bound = max(max_bound, model_max)
        object_count += 1

    # 5. Handle single object degenerate case
    if object_count == 1:
        centroid = (min_bound + max_bound) / 2.0
        min_bound = centroid
        max_bound = centroid

    # 6. Clamp Z
    min_bound.z = clamp(min_bound.z, 0, 255)
    max_bound.z = clamp(max_bound.z, 0, 255)

    # 7-9. Compute Matrices
    identity = Matrix4x4.identity()
    centroid = (min_bound + max_bound) / 2.0
    view_matrix = look_at(centroid, centroid + light_direction, Vector3(0, 1, 0))
    
    right, left, top, bottom, near, far = 210, -210, 210, -210, 210, -210
    proj_matrix = Matrix4x4(
        m=[
            [2.0/(right-left), 0, 0, -(right+left)/(right-left)],
            [0, 2.0/(top-bottom), 0, -(top+bottom)/(top-bottom)],
            [0, 0, -2.0/(far-near), -(far+near)/(far-near)],
            [0, 0, 0, 1]
        ]
    )
    
    shadow_matrix = proj_matrix * view_matrix

    # 11. Setup FBO
    bind_framebuffer(GL_FRAMEBUFFER, shadow_framebuffer_id)
    set_viewport(0, 0, 1024, 1024)
    enable_depth_test()
    clear_depth_buffer()

    # 12-13. Setup Shader
    use_shader(4)
    set_uniform_matrix4("shadowMatrix", shadow_matrix)
    set_uniform3("lightDirection", light_direction)

    # 14. Render
    for entity in entities:
        render_entity(entity)
```

**Edge Cases and Pitfalls:**
*   **Degenerate AABB:** If only one object is shadowed, `min` equals `max`, resulting in a zero-volume orthographic matrix. This is handled by collapsing bounds to the object's centroid [S1, unofficial].
*   **Viewport Corruption:** Objects outside the viewport can corrupt the shadow matrix bounds if not culled. Early culling based on screen-space AABB prevents this [S1, unofficial].
*   **Precision:** The depth texture uses `GL_FLOAT`. Ensure the orthographic near/far planes are set correctly to maintain depth precision across the clamped Z range [S1, unofficial].

## Subtopics

### 2D Voxel Shadow Map
The base concept storing the highest occupied voxel Y-coordinate per (X, Z) column. Choose this when you need simple occlusion without ray marching. [2D Voxel Shadow Map](5769-2d-voxel-shadow-map.md)

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of visible entities, as each entity is processed twice (once for bounds, once for rendering).
*   **Memory Complexity:** $O(1)$ for the shadow map itself (fixed 1024x1024 float depth buffer).
*   **Measured Performance:** Executed in the `ShadowsUpdate` system every game loop iteration [S1, unofficial].
*   **Strengths:** Fast per-frame update for dynamic scenes; simple depth comparison during lighting.
*   **Limitations:** Resolution-dependent (1024x1024); requires rebinding and clearing FBO every frame; directional light only.
*   **When to use:** Dynamic voxel worlds with moving objects.
*   **When not to use:** Static scenes where a pre-computed shadow map suffices; perspective lighting.

## Background
Originated in the PixelVoxels project by KellerMartins [S1, unofficial]. It is used in voxel-based games to provide real-time shadowing for dynamic entities without the computational cost of ray marching through the voxel grid.

## Sources
[S1] KellerMartins/PixelVoxels, https://github.com/KellerMartins/PixelVoxels, fetched 2026-10-27 (unofficial)
