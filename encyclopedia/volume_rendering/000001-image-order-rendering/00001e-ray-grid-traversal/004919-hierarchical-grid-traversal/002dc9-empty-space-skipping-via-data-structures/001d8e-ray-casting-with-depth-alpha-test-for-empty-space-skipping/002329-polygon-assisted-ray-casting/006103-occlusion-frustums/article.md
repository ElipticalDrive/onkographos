# Occlusion Frustums

Also called: Occlusion volumes, Proxy geometry via first-hit image

## Overview
Occlusion Frustums is an empty space skipping optimization for volume raycasting that reduces costly volume texture fetches by generating proxy geometry based on visibility information from previous frames [S1]. Unlike Polygon Assisted Ray Casting, which requires complex, data-dependent preprocessing and simplification, this method constructs simple pyramidal frustums on-the-fly using a first-hit image, allowing instant adaptation to transfer function changes without invalidation [S1]. It differs from point-based space leaping by operating in object space, eliminating the need for reprojection during view changes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A "first-hit image" from the previous frame's rendering pass. This image contains the position of the first non-empty voxel per ray in its color channels and marks rays hitting only empty voxels in its alpha channel [S1].
*   **Output:** A set of triangle primitives (proxy geometry) representing occlusion frustums, which are rendered to a depth buffer to skip ray traversal in empty regions [S1].

**Data Structures:**
*   **First-Hit Image:** A 2D texture where each pixel $(u, v)$ corresponds to a ray [S1]. The RGB values encode the 3D position of the first hit voxel [S1]. The alpha channel marks rays that only hit empty voxels [S1].
*   **Occlusion Block:** A grouping of pixels in the first-hit image [S1].

**Parameters:**
*   **Block Size:** The dimension of the pixel grid grouping [S1]. Typical value: $4 \times 4$ [S1]. This balances accuracy against the complexity of the generated geometry [S1].

**Steps:**
1.  **Generate First-Hit Image:** During the previous frame's raycasting, render a first-hit image [S1]. For each ray, if a non-empty voxel is hit, store its 3D position in the pixel color [S1]. If the ray hits only empty space, mark the pixel alpha as empty [S1].
2.  **Identify Empty Blocks:** Divide the first-hit image into blocks of size $B \times B$ (e.g., $4 \times 4$) [S1]. A block is marked as empty if all constituent pixels indicate empty voxels [S1].
3.  **Construct Frustum Geometry:** For each non-empty block, construct a quadrilateral pyramidal occlusion frustum [S1]:
    *   **Top Base:** A quadrilateral located at the 3D position of the first-hit voxel corresponding to the block [S1].
    *   **Extrusion:** Extrude the top base along projectors extending from the Center of Projection (COP) through the vertices of the top base quad [S1].
    *   **Depth Extent:** Extend the frustum such that its bottom base lies outside the dataset's bounding box [S1].
    *   **Clipping:** Clip the frustum against the dataset's bounding box (unit cube $[0,1]^2$ in ray parameter space) to ensure it fits within the hardware-clamped color range $[0,1]$ [S1].
    *   **Enlargement:** Slightly enlarge the clipped frustum and move it towards the camera to ensure all potentially non-empty voxels are covered [S1].
4.  **Handle New Objects:** To prevent skipping objects that enter the view from outside the previous frame's view frustum, subtract the previous frame's view frustum from an initial cube proxy geometry (which encloses all voxels) and add this resulting geometry to the set of occlusion frustums [S1].
5.  **Render Proxy Geometry:** Render the constructed frustums using a fragment shader that assigns vertex positions as fragment colors [S1]. The z-buffer handles overlapping frustums [S1].
6.  **Raycasting Skip:** During the current frame's raycasting, use the rendered proxy geometry to determine if a ray passes through empty space [S1]. If a ray intersects the proxy geometry, it skips sampling in that region [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_occlusion_frustums(first_hit_image, block_size=4):
    """
    Generates proxy geometry from a first-hit image.
    first_hit_image: 2D array of (color, alpha) tuples.
    Returns: List of triangle primitives.
    """
    frustums = []
    height, width = first_hit_image.shape
    
    # Iterate over blocks
    for y in range(0, height, block_size):
        for x in range(0, width, block_size):
            # Check if block is empty
            is_empty = True
            hit_pos = None
            for dy in range(block_size):
                for dx in range(block_size):
                    if y + dy < height and x + dx < width:
                        color, alpha = first_hit_image[y + dy, x + dx]
                        if alpha != 0.0: # Alpha indicates emptiness
                            is_empty = False
                            hit_pos = color # Use first non-empty hit for block
                            break
                if not is_empty:
                    break
            
            if not is_empty:
                # Construct frustum for this block
                frustum = build_frustum(hit_pos, block_size, x, y, width, height)
                frustums.append(frustum)
                
    # Add geometry for new objects (outside previous view frustum)
    new_object_geometry = subtract_previous_view_frustum_from_initial_cube()
    frustums.extend(new_object_geometry)
    
    return frustums

def build_frustum(hit_pos, block_size, block_x, block_y, img_width, img_height):
    """
    Constructs a clipped, enlarged occlusion frustum.
    hit_pos: 3D position of the first hit voxel.
    """
    # 1. Define top base quad at hit_pos
    top_quad = get_quad_vertices_at_position(hit_pos, block_size)
    
    # 2. Extrude along projectors from Center of Projection (COP)
    cop = get_center_of_projection()
    bottom_quad = []
    for vertex in top_quad:
        # Ray from COP through vertex
        direction = normalize(vertex - cop)
        # Extrude far enough to go outside bounding box
        extended_vertex = cop + direction * large_distance
        bottom_quad.append(extended_vertex)
        
    # 3. Clip against dataset bounding box
    # The frustum must fit in [0,1]^2 for ray parameter encoding
    clipped_frustum = clip_frustum_against_bbox(top_quad, bottom_quad, bbox)
    
    # 4. Enlarge and move towards camera
    enlarged_frustum = enlarge_and_shift_towards_camera(clipped_frustum)
    
    # 5. Convert to triangles
    triangles = quad_to_triangles(enlarged_frustum)
    return triangles

def subtract_previous_view_frustum_from_initial_cube():
    """
    Generates geometry for objects entering from outside the previous view.
    """
    # Initial cube proxy geometry encloses all voxels
    initial_cube = get_bounding_box_cube()
    previous_view_frustum = get_previous_frame_view_frustum()
    
    # Subtract previous view frustum from initial cube
    new_object_geo = boolean_subtract(initial_cube, previous_view_frustum)
    return new_object_geo

def get_quad_vertices_at_position(hit_pos, block_size):
    s = block_size * 0.5
    return [
        (hit_pos[0] - s, hit_pos[1] - s, hit_pos[2]),
        (hit_pos[0] + s, hit_pos[1] - s, hit_pos[2]),
        (hit_pos[0] + s, hit_pos[1] + s, hit_pos[2]),
        (hit_pos[0] - s, hit_pos[1] + s, hit_pos[2])
    ]

def get_center_of_projection():
    return (0.0, 0.0, 0.0)

def normalize(v):
    length = (v[0]**2 + v[1]**2 + v[2]**2)**0.5
    return (v[0]/length, v[1]/length, v[2]/length)

def clip_frustum_against_bbox(top_quad, bottom_quad, bbox):
    # Clips the frustum geometry to fit within the unit cube [0,1]^2 for ray parameters
    return [top_quad, bottom_quad]

def enlarge_and_shift_towards_camera(clipped_frustum):
    # Slightly enlarges the frustum and moves it towards the camera
    return clipped_frustum

def quad_to_triangles(frustum):
    # Converts quad geometry to 12 triangles
    return []

def get_bounding_box_cube():
    return "CubeGeometry"

def get_previous_frame_view_frustum():
    return "ViewFrustumGeometry"

def boolean_subtract(cube, frustum):
    # Subtracts frustum geometry from cube geometry
    return "DifferenceGeometry"

large_distance = 1000.0
bbox = (0.0, 0.0, 0.0, 1.0, 1.0, 1.0)
```

**Edge Cases and Pitfalls:**
*   **Undersampling:** If voxels are missed due to undersampling, parts of the volume may be incorrectly identified as empty and permanently removed from the proxy geometry [S1]. This is mitigated by constant refreshing of the geometry and using occlusion blocks (all voxels in a block must be missed) [S1].
*   **Hardware Clamping:** Geometry reaching outside the unit cube $[0,1]^2$ leads to wrong ray directions because hardware clamps colors to $[0,1]$ [S1]. The frustum must be closed from all sides and clipped against the bounding box [S1].
*   **New Objects:** Objects appearing outside the previous view frustum are skipped if not handled [S1]. This is addressed by adding the geometry resulting from subtracting the previous view frustum from the initial cube proxy [S1].
*   **Non-continuous View Changes:** Unlike point-based methods, this technique works for non-continuous view changes and introduces no image artifacts [S1].

## Operating characteristics
*   **Time Complexity:** The runtime of the optimization is mainly dependent on the resulting 2D image resolution, not the 3D data set resolution [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Tested on Intel Core 2 Duo E6300 CPU and NVIDIA GeForce 8800 GT GPU [S1].
    *   Sparse data sets (e.g., 'vertebra'): Speedup of 1.36x [S1].
    *   Dense data sets (e.g., 'vmhead (skin)'): Speedup of 0.77x (slowdown) due to overhead [S1].
    *   The optimization is most useful for high-resolution data sets and high-quality rendering where sampling costs are high [S1].
*   **Scaling:** The technique scales well even to the quite large viewport size of $1024^2$, for some data sets even producing greater speed-ups than for $512^2$ [S1].
*   **Strengths:** No preprocessing required; adapts instantly to transfer function changes; works for non-continuous view changes; no image artifacts [S1].
*   **Limitations:** Overhead for dense data sets; potential for undersampling errors (mitigated by refresh rate and block size) [S1].

## Background
*   **Origin:** Introduced by J. Mensmann, T. Ropinski, and K. Hinrichs in 2008 [S1].
*   **Paper:** "Accelerating Volume Raycasting using Occlusion Frustums" [S1].
*   **History:** Developed as a novel approach to empty space leaping to reduce costly volume texture fetches during ray traversal [S1]. It was implemented in OpenGL/GLSL and integrated into the Voreen volume rendering framework [S1].
*   **Comparison:** Unlike PARC [ASK92], it does not use a closely-fitted, data-dependent proxy geometry requiring simplification [S1]. Unlike point-based space leaping, no reprojection is necessary [S1].

## Sources
[S1] Accelerating Volume Raycasting using Occlusion Frustums, https://doi.org/10.2312/vg/vg-pbg08/147-154, fetched 2026-10-24
