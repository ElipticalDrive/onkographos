# Dynamic Orthographic Shadow Map Generation

Also called: Shadow Depth Pass, Voxel Shadow Map

## Overview
Generates a per-frame depth texture from a directional light's perspective to compute shadow occlusion for voxel models [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scene entities with bounding boxes, directional light direction (from scene trie), current viewport dimensions [S1, unofficial].
*   **Output:** A 1024x1024 depth texture (`shadowDepthTexture`) attached to an OpenGL framebuffer object (`shadowFramebuffer`) [S1, unofficial].

**Data Structures:**
*   `shadowFramebuffer`: OpenGL framebuffer object ID bound to `GL_FRAMEBUFFER` [S1, unofficial].
*   `shadowDepthTexture`: 2D texture ID, internal format `GL_DEPTH_COMPONENT`, type `GL_FLOAT`, dimensions 1024x1024, attached to `GL_DEPTH_ATTACHMENT` [S1, unofficial].
*   `shadowMatrix`: 4x4 column-major float array [S1, unofficial].
*   `AABB`: Axis-Aligned Bounding Box defined by `min` and `max` vectors [S1, unofficial].

**Parameters:**
*   `shadowTextureWidth`, `shadowTextureHeight`: Resolution of the shadow depth map in pixels [S1, unofficial].
*   Orthographic projection bounds: `right=210`, `left=-210`, `top=210`, `bottom=-210`, `near=210`, `far=-210` [S1, unofficial].
*   Z-clamp range: Minimum and maximum Z coordinates constrained to `[0, 255]` before matrix construction [S1, unofficial].

**Steps:**
1.  Initialize global bounding box vectors `min` to $(\infty, \infty, \infty)$ and `max` to $(-\infty, -\infty, -\infty)$ [S1, unofficial].
2.  Retrieve the model's bounding box and screen-space bounds via `GetModelBounds` [S1, unofficial].
3.  If the object's screen-space bounds fall completely outside the viewport, skip it to avoid unnecessary shadow map updates [S1, unofficial].
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
    min_bound = [float('inf'), float('inf'), float('inf')]
    max_bound = [float('-inf'), float('-inf'), float('-inf')]
    object_count = 0

    # 2-4. Accumulate bounds
    for entity in entities:
        model_min, model_max = entity.get_bounds()
        screen_min, screen_max = get_model_bounds(entity, viewport_bounds)

        # 3. Cull outside viewport
        if not intersects_viewport(screen_min, screen_max, viewport_bounds):
            continue

        min_bound = [min(min_bound[i], model_min[i]) for i in range(3)]
        max_bound = [max(max_bound[i], model_max[i]) for i in range(3)]
        object_count += 1

    # 5. Handle single object degenerate case
    if object_count == 1:
        centroid = [(min_bound[i] + max_bound[i]) / 2.0 for i in range(3)]
        min_bound = centroid
        max_bound = centroid

    # 6. Clamp Z
    min_bound[2] = max(0.0, min(255.0, min_bound[2]))
    max_bound[2] = max(0.0, min(255.0, max_bound[2]))

    # 7-9. Compute Matrices
    centroid = [(min_bound[i] + max_bound[i]) / 2.0 for i in range(3)]
    target = [centroid[0] + light_direction[0], centroid[1] + light_direction[1], centroid[2] + light_direction[2]]
    view_matrix = look_at(centroid, target, [0, 1, 0])

    right, left, top, bottom, near, far = 210, -210, 210, -210, 210, -210
    proj_matrix = [
        [2.0/(right-left), 0, 0, -(right+left)/(right-left)],
        [0, 2.0/(top-bottom), 0, -(top+bottom)/(top-bottom)],
        [0, 0, -2.0/(far-near), -(far+near)/(far-near)],
        [0, 0, 0, 1]
    ]

    shadow_matrix = multiply_matrices(proj_matrix, view_matrix)

    # 11. Setup FBO
    # bind_framebuffer(GL_FRAMEBUFFER, shadow_framebuffer_id)
    # set_viewport(0, 0, 1024, 1024)
    # enable_depth_test()
    # clear_depth_buffer()

    # 12-13. Setup Shader
    # use_shader(4)
    # set_uniform_matrix4("shadowMatrix", shadow_matrix)
    # set_uniform3("lightDirection", light_direction)

    # 14. Render
    for entity in entities:
        render_entity(entity)

# Helper definitions
def get_model_bounds(entity, viewport):
    # Projects entity's world-space AABB to screen space [G]
    corners = []
    for x in [entity.min[0], entity.max[0]]:
        for y in [entity.min[1], entity.max[1]]:
            for z in [entity.min[2], entity.max[2]]:
                corners.append([x, y, z])
    screen_corners = [project_to_screen(c, viewport) for c in corners]
    return [min(c[i] for c in screen_corners) for i in range(2)], [max(c[i] for c in screen_corners) for i in range(2)]

def intersects_viewport(screen_min, screen_max, viewport):
    # Checks if screen-space AABB overlaps viewport [G]
    return screen_min[0] < viewport.max_x and screen_max[0] > viewport.min_x and \
           screen_min[1] < viewport.max_y and screen_max[1] > viewport.min_y

def look_at(eye, target, up):
    # Standard view matrix construction [G]
    z_axis = normalize(subtract(eye, target))
    x_axis = normalize(cross(up, z_axis))
    y_axis = cross(z_axis, x_axis)
    return [
        [x_axis[0], y_axis[0], z_axis[0], 0],
        [x_axis[1], y_axis[1], z_axis[1], 0],
        [x_axis[2], y_axis[2], z_axis[2], 0],
        [-dot(x_axis, eye), -dot(y_axis, eye), -dot(z_axis, eye), 1]
    ]

def multiply_matrices(a, b):
    # Standard 4x4 matrix multiplication [G]
    res = [[0]*4 for _ in range(4)]
    for i in range(4):
        for j in range(4):
            for k in range(4):
                res[i][j] += a[i][k] * b[k][j]
    return res

def project_to_screen(point, viewport):
    # Simplified projection to screen coordinates [G]
    return [point[0] * 100, point[1] * 100] # Placeholder logic for demo, actual depends on camera

def bind_framebuffer(target, id): pass
def set_viewport(x, y, w, h): pass
def enable_depth_test(): pass
def clear_depth_buffer(): pass
def use_shader(index): pass
def set_uniform_matrix4(name, mat): pass
def set_uniform3(name, vec): pass
def render_entity(entity): pass
```

**Edge Cases and Pitfalls:**
*   **Degenerate AABB:** If only one object is shadowed, `min` equals `max`, resulting in a zero-volume orthographic matrix. This is handled by collapsing bounds to the object's centroid [S1, unofficial].
*   **Viewport Corruption:** Objects outside the viewport can corrupt the shadow matrix bounds if not culled. Early culling based on screen-space AABB prevents this [S1, unofficial].

## Subtopics

### 2D Voxel Shadow Map
The base concept storing the highest occupied voxel Y-coordinate per (X, Z) column. Choose this when you need simple occlusion without ray marching. [2D Voxel Shadow Map](5769-2d-voxel-shadow-map.md)

## Operating characteristics
*   **Measured Performance:** Executed in the `ShadowsUpdate` system every game loop iteration [S1, unofficial].
*   **Strengths:** Fast per-frame update for dynamic scenes; simple depth comparison during lighting. [G]
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
Executed in the ShadowsUpdate system every game loop iteration [S1, unofficial].

## Sources
[S1] KellerMartins/PixelVoxels, https://github.com/KellerMartins/PixelVoxels, fetched 2026-10-27 (unofficial)
