# Frustum-Occlusion Culling for Octree Traversal

Also called: Frustum-octant traversal, Voxel LOD selection, Voxel LOD selection via Frustum Culling, Voxel Rasterization via Frustum Culling and Instancing

## Overview
This method identifies a set of visible voxels by traversing an octree hierarchy and comparing each node against the camera's view frustum [S1, unofficial]. It serves as a visibility culling and Level-of-Detail (LOD) selection mechanism for voxel rasterization, ensuring that only octants intersecting the frustum and large enough to project onto at least one pixel are processed [S1, unofficial]. It differs from standard ray-marching octree traversal by operating in screen-space to cull entire subtrees based on projected area, rather than per-ray intersection tests [S1, unofficial].

## How it works
**Inputs:**
*   Camera view frustum (defined by 6 planes).
*   Octree structure (implicit or explicit), with a known Axis-Aligned Bounding Box (AABB) enclosing the root.
*   Screen resolution (width $W$, height $H$).

**Outputs:**
*   A buffer of visible voxels, where each entry contains the voxel's position and size (derived from the octree node's AABB).

**Data Structures:**
*   **Octree Node:** Implicitly defined by its depth $d$ and position within the global AABB. The spatial extent is determined by the root AABB and the subdivision level.
*   **Voxel Buffer:** A dynamic array storing tuples of $(\mathbf{p}, s)$, where $\mathbf{p}$ is the center of the voxel and $s$ is its half-extent.

**Canonical Voxel Template:**
The geometric primitive used for rasterization is a cube with extreme vertices $\mathbf{c}_{min} = (-1, -1, -1)$ and $\mathbf{c}_{max} = (+1, +1, +1)$ [S1, unofficial]. This template is scaled and translated to match the voxel's position and size during instancing.

**Algorithm Steps:**

1.  **Initialize Traversal:** Start at the root octant. The root's AABB is known.
2.  **Frustum Intersection Test:** For the current node, determine if it intersects the view frustum.
    *   To optimize, approximate the octant as a sphere. The sphere's diameter is the diagonal of the octant's AABB [S1, unofficial].
    *   Let the octant center be $\mathbf{c}$ and half-extent be $s$. The sphere radius is $r = s\sqrt{3}$.
    *   Test if this sphere intersects the frustum planes [S1, unofficial].
3.  **Projected Area Test:** If the node intersects the frustum, calculate its projected area on the screen.
    *   Project the octant's AABB corners to screen space.
    *   Calculate the bounding area $A_{proj}$ of the projected shape.
    *   Calculate the area of a single pixel $A_{pixel} = \frac{W \times H}{\text{total pixels}}$ (or simply $1$ in normalized device coordinates, but scaled by resolution for comparison) [S1, unofficial].
4.  **Termination Conditions:** Stop traversing the current node's children if:
    *   The node is a leaf (no more subdivisions).
    *   The projected area $A_{proj}$ is less than $A_{pixel}$ [S1, unofficial].
5.  **Buffering:** If traversal stops due to either condition above, append the voxel's position $\mathbf{p}$ (center of the AABB) and size $s$ (half-extent) to the Voxel Buffer [S1, unofficial].
6.  **Recursion:** If the node passes both tests and is not a leaf, recursively traverse its 8 children.
7.  **Rasterization:** Once the buffer is complete, use hardware instancing to render the voxels. Each entry in the buffer generates a cube using the canonical template [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def frustum_octree_traversal(node, frustum, screen_width, screen_height, voxel_buffer):
    """
    Recursively traverse the octree, culling based on frustum intersection 
    and projected screen area.
    
    Args:
        node: Object representing the current octree node. 
              Attributes: center (vec3), half_extent (float), is_leaf (bool), children (list of nodes or None).
        frustum: Object with method intersects_sphere(center, radius).
        screen_width: int
        screen_height: int
        voxel_buffer: List to append visible voxel data.
    """
    
    # 1. Frustum-Sphere Intersection Test
    # Approximate octant as sphere with diameter = diagonal of AABB
    # Diagonal length = 2 * half_extent * sqrt(3)
    # Radius = half_extent * sqrt(3)
    radius = node.half_extent * 1.73205080757 # sqrt(3)
    
    if not frustum.intersects_sphere(node.center, radius):
        return

    # 2. Projected Area Test
    # Project the node's AABB to screen space to get bounding area
    projected_area = project_aabb_to_screen(node.aabb, frustum.projection_matrix, screen_width, screen_height)
    pixel_area = (screen_width * screen_height) / (screen_width * screen_height) # Normalized 1.0 if NDC, or actual pixel count if needed. 
    # Note: The source implies comparing projected area to "area of a pixel". 
    # In NDC, a pixel is 1/(W*H). In screen coords, it is 1. 
    # We assume comparison is done in consistent screen-space units.
    # Let's assume screen_width/height are in pixels, so pixel_area = 1.0
    pixel_area = 1.0 
    
    # 3. Termination Conditions
    is_leaf = node.is_leaf
    too_small = projected_area < pixel_area
    
    if is_leaf or too_small:
        # Append position and size to buffer
        voxel_buffer.append({
            'position': node.center,
            'size': node.half_extent
        })
        return

    # 4. Recurse
    if node.children is not None:
        for child in node.children:
            frustum_octree_traversal(child, frustum, screen_width, screen_height, voxel_buffer)
```

**Edge Cases and Pitfalls:**
*   **Numeric Precision:** At high octree depths, numeric precision issues may arise during coordinate calculations [S1, unofficial].
*   **Empty Nodes:** The traversal must skip empty octants. The source notes that the octree can be stored with a single 8-bit mask per node to indicate occupancy, allowing efficient skipping [S1, unofficial].
*   **LOD Threshold:** The "projected area < pixel area" criterion acts as an automatic LOD selector. If the threshold is too aggressive, detail is lost; if too conservative, performance drops. The source does not specify a fixed multiplier, implying a direct comparison [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Proportional to the number of visible voxels at the selected LOD. The traversal cost is proportional to the number of nodes visited, which is bounded by the surface area of the visible volume in screen space.
*   **Memory Complexity:** Linear with respect to the number of visible voxels, as each visible voxel is stored in the buffer.
*   **Performance:** Profiled on an Intel Core i5-2540M and NVIDIA Quadro 3000M at 1920x1280 resolution [S1, unofficial].
    *   Voxel rendering times for the 'basketball' dataset (octree depth 10):
        *   Geometry Shader: 3.454 ms
        *   Uniform Buffer: 2.549 ms
        *   Hardware Texture Buffer: 2.773 ms
        *   Instancing Vertex Buffer: 5.599 ms
*   **Strengths:** Automatic LOD selection; efficient culling of distant/small voxels; compatible with hardware instancing.
*   **Limitations:** Requires a full traversal to build the buffer before rendering; performance depends heavily on the efficiency of the frustum-sphere intersection test; Geometry Shaders are slower for voxels than for quads [S1, unofficial].

## Background
This method was described in the context of hardware-accelerated geometry instancing for surfel and voxel rendering of scanned 4D media [S1, unofficial]. It was developed to address the need for efficient rendering of large-scale voxel datasets by combining hierarchical culling with modern GPU instancing techniques. The approach leverages the fact that hardware has a fast path for quadrilaterals but is less optimized for complex voxel cubes, making general-purpose instancing pipelines preferable to Geometry Shaders for this specific geometry [S1, unofficial].

## Sources
[S1] Hardware-accelerated geometry instancing for surfel and voxel rendering of scanned 4d media, http://133.5.19.150/papers/2010_2013/1-1-36.pdf, fetched 2026-10-31 (unofficial)
