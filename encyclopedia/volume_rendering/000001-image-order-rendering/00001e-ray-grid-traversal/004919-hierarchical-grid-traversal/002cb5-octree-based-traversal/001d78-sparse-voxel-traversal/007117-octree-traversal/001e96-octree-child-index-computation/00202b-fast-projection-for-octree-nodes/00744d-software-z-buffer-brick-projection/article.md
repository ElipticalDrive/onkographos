# Software Z-buffer Brick Projection

Also called: Projected z-value template, Template-based projection

## Overview
Software Z-buffer Brick Projection is a CPU-based method for determining ray-volume intersection regions in sparse voxel octrees by projecting a single representative brick per viewing direction to create a z-buffer footprint template [S1]. It solves the problem of integrating volume rendering into systems where off-screen OpenGL rendering (PBuffers) is unavailable or memory-constrained, such as high-resolution dual-display medical visualization setups [S1]. Unlike hardware-accelerated approaches, it avoids the overhead of `glReadPixels` and dedicated GPU memory buffers by exploiting the geometric regularity of octree bricks, achieving performance comparable to OpenGL implementations while remaining fully software-defined [S1].

## How it works
The method relies on the fact that all bricks at a specific octree level share the same relative geometry. Therefore, the projection footprint (z-buffer values) for any brick at a given level can be derived from a single "template" brick projected in the current viewing direction [S1].

**Inputs:**
- Octree structure with defined brick sizes.
- Current viewing direction (camera position and orientation).
- Octree level $L$ for which the footprint is being computed.

**Outputs:**
- A z-buffer template (array of z-values) representing the depth of the brick's front and back faces for each screen pixel covered by the projection.
- This template is used to determine which rays intersect the volume and where the entry points are located [S1].

**Data Structures:**
- **Z-buffer Template:** A 2D array of floating-point values, size $W \times H$ (screen resolution). Each element stores the z-component of the ray starting position (depth) where the brick surface intersects the ray [S1].
- **Brick Geometry:** Defined by its center, size, and orientation. Since bricks are axis-aligned or uniformly transformed, the template applies to all bricks of the same level via translation [S1].

**Steps:**
1. **Select Representative Brick:** Choose one arbitrary brick from the octree at the target level $L$. This brick serves as the geometric template for all other bricks at this level [S1].
2. **Project Template Brick:** Project the selected brick onto the screen plane for the current viewing direction. This involves calculating the screen-space bounding box and the depth (z-value) of the brick's front and back faces for each pixel within that bounding box [S1].
3. **Fill Z-buffer Template:** For each pixel in the projected area, compute the z-value corresponding to the intersection of the viewing ray with the brick's front face (entry point) and back face (exit point). Store the minimum z-value (closest to camera) in the z-buffer template to represent the visible surface [S1].
4. **Render Entry Bricks:** Iterate through all possible entry bricks (bricks that could be visible from the current view) in front-to-back order. Use the pre-computed z-buffer template to determine the final z-buffer footprint of the octree. This step effectively maps the template's depth values to the spatial positions of the actual bricks [S1].
5. **Determine Intersections:** Use the resulting z-buffer footprint to identify which rays intersect the volume. The z-buffer directly contains the z-components of the ray starting positions, allowing the raycaster to skip non-intersecting rays and begin integration at the correct entry point [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_z_buffer_template(brick, view_direction, screen_width, screen_height):
    """
    Projects a single brick to create a z-buffer footprint template.
    brick: Object with center, size, and transformation matrix.
    view_direction: Camera orientation and position.
    screen_width, screen_height: Resolution of the output template.
    Returns: 2D array of z-values (float), initialized to infinity.
    """
    z_buffer = [[float('inf')] * screen_width for _ in range(screen_height)]
    
    # 1. Project brick vertices to screen space to find bounding box
    # [G] Standard perspective projection: 
    # x_screen = (M * v).x / (M * v).w
    # y_screen = (M * v).y / (M * v).w
    # where M is the combined view-projection matrix.
    vertices = get_brick_vertices(brick)
    screen_vertices = [project_to_screen(v, view_direction) for v in vertices]
    
    # 2. Determine pixel coverage
    min_x, max_x = min(v.x for v in screen_vertices), max(v.x for v in screen_vertices)
    min_y, max_y = min(v.y for v in screen_vertices), max(v.y for v in screen_vertices)
    
    # 3. For each pixel in the bounding box, compute z-value
    for y in range(int(min_y), int(max_y) + 1):
        for x in range(int(min_x), int(max_x) + 1):
            if is_pixel_inside_brick_projection(x, y, screen_vertices, brick):
                # Compute ray intersection with brick front face
                ray_origin, ray_direction = get_ray_for_pixel(x, y, view_direction)
                t_entry, t_exit = intersect_ray_brick(ray_origin, ray_direction, brick)
                
                if t_entry is not None:
                    z_value = ray_origin.z + t_entry * ray_direction.z
                    # Store the closest z-value (front face)
                    if z_value < z_buffer[y][x]:
                        z_buffer[y][x] = z_value
                        
    return z_buffer

def render_octree_footprint(octree, z_buffer_template, view_direction):
    """
    Uses the template to determine the final z-buffer footprint for all bricks.
    """
    # [G] Iterate through all bricks at the target level
    for brick in octree.get_bricks_at_level():
        # Translate the template based on the brick's position relative to the representative brick
        # [G] The template contains depth offsets. Apply translation to map to world space.
        translated_z_buffer = translate_template(z_buffer_template, brick.center)
        
        # Update global z-buffer with the brick's contribution
        update_global_z_buffer(translated_z_buffer)
```

**Parameters:**
- **Screen Resolution:** The size of the z-buffer template. Typical values depend on the display, e.g., 1024x1024 [S1].
- **Octree Level:** The granularity of the bricks. The method is applied per level [S1].

**Edge Cases and Pitfalls:**
- **Memory Constraints:** On high-resolution dual-display setups, GPU memory may be fully occupied by the display buffers, leaving no space for OpenGL PBuffers (which require ~8 MB for a 1024x1024 image) [S1]. This method avoids that constraint.
- **Precision:** The z-buffer stores z-components of ray starting positions. Care must be taken with floating-point precision when translating the template to different brick locations [S1].
- **Granularity:** The method estimates intersections at the brick level. For finer detail, a cell invisibility cache is used to skip transparent cells down to 4x4x4 samples, as the octree may not go down to the cell level [S1].

## Subtopics

### [Sparse Octree Filtering with Node Map](2337-sparse-octree-filtering-with-node-map.md)
A sibling method that uses a node map for filtering, differing in its approach to managing sparse data density compared to the template-based projection of bricks.

## Operating characteristics
- **Time Complexity:** The costly projection operation is performed only once per viewing direction per octree level, making it as fast as the OpenGL implementation for the projection phase [S1]. Subsequent steps involve translating the template and updating the z-buffer, which is efficient due to the regular structure [S1].
- **Memory Complexity:** Requires $O(W \times H)$ memory for the z-buffer template, where $W$ and $H$ are screen dimensions. This is significantly less than the memory required for off-screen OpenGL buffers (e.g., 8 MB for 1024x1024) if multiple buffers were needed, and it does not consume GPU memory [S1].
- **Performance:** Achieves performance comparable to hardware-accelerated OpenGL rendering because it eliminates the overhead of `glReadPixels` and off-screen buffer management [S1].
- **Strengths:** Works on any hardware without requiring specific OpenGL extensions (PBuffers); avoids GPU memory contention; efficient for large volumes where many bricks share the same projection footprint.
- **Limitations:** Requires CPU computation for the initial projection; may be slower than fully hardware-accelerated pipelines if the CPU is a bottleneck; limited by the granularity of the octree (requires cell invisibility cache for fine details) [S1].
- **When to use:** Ideal for integrated visualization systems where GPU memory is constrained or unavailable for off-screen rendering, particularly in medical visualization with high-resolution displays [S1].

## Background
The method was developed as a software alternative to OpenGL-based rendering for volume raycasting of large data sets, specifically in the context of medical visualization [S1]. It addresses the limitations of PBuffers, which are not available on every graphics card and consume significant memory, conflicting with high-resolution dual-display setups [S1]. The approach builds on the idea of template-based projection, similar to work by Srinivasan et al. [S1]. It is part of a broader effort to create memory-efficient acceleration structures and techniques for CPU-based volume raycasting [S1].

## Sources
[S1] Memory efficient acceleration structures and techniques for CPU-based volume raycasting of large data, https://doi.org/10.1109/svvg.2004.8, fetched 2026-10-26
