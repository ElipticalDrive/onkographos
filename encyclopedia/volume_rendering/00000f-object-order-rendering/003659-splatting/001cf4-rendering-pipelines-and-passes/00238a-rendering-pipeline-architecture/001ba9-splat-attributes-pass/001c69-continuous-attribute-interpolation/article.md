# Continuous Attribute Interpolation

Also called: Bilinear Interpolation Attribute Sampling, Correct Interpolation Variant, Direct Interpolation Variant, Geometry and Attribute Sampling, MLS Attribute Sampling

## Overview
Continuous Attribute Interpolation is a method for obtaining continuous reconstruction of UV coordinates for attribute interpolation on splats, specifically designed to avoid visual artifacts at splat boundaries [S1, unofficial]. It solves the problem of discontinuous attribute sampling that arises when intersecting view rays with flat splat planes, which can cause large UV coordinate jumps for neighboring fragments [S1, unofficial]. It sets itself apart from naive direct interpolation by using a "correct interpolation" variant that projects intersections onto a plane defined by elevated neighboring points, ensuring smooth attribute transitions [S1, unofficial].

## How it works
The method operates on a model represented as a set of elevation and attribute maps, where each patch corresponds to a piece of the 3D model parameterized as a height field over a planar domain [S1, unofficial]. The pipeline operates in a single rendering pass, storing elevation maps in Vertex Buffer Objects (VBOs) and attribute maps in texture objects [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   View ray origin and direction.
    *   Splat geometry: A set of input points defining the patch, including their 3D positions and attributes (e.g., normals, texture coordinates).
    *   Patch metadata: A $4 \times 4$ transformation matrix per patch [S1, unofficial].
    *   Attribute maps: Regular grids storing resampled attributes (normals, textures) [S1, unofficial].
*   **Outputs:**
    *   Corrected UV coordinates for the fragment.
    *   Interpolated attribute values (normals, textures) at the fragment location.

### Data Structures
*   **Elevation Map:** A regular grid stored in a VBO. Each cell corresponds to a sample on the support plane.
*   **Attribute Map:** A regular grid (texture) storing resampled attributes. Resolution is chosen independently per map [S1, unofficial].
*   **Patch Data:** For each splat, the system maintains the set of input points and the associated support plane.

### Steps

1.  **Patch Partitioning and Overlap Handling:**
    The 3D model is partitioned into regions, each corresponding to a surface patch. To ensure continuous rendering and avoid gaps between patches, patches are made to slightly overlap [S1, unofficial]. An input point is included in a region if its $\epsilon$-neighborhood intersects the point set of that region, where $\epsilon > 0$ is a small practical value [S1, unofficial].

2.  **Attribute Resampling (Pre-processing):**
    Attributes (normals, textures) are resampled from the original 3D model onto the regular grids of the attribute maps [S1, unofficial].
    *   **MLS Method:** Normal vectors at arbitrary height field locations are computed as normal vectors of the associated Moving Least Squares (MLS) surface [S1, unofficial].
    *   **Bilinear/Delaunay Method:** If normal vectors are given per input point, they are resampled on a regular grid on the support plane using bilinear interpolation within a Delaunay triangulation of the projected input points on the support plane [S1, unofficial].
    *   Texture attributes are handled identically to normal vectors [S1, unofficial].

3.  **Ray-Splat Intersection:**
    Intersect the view ray with the oriented splat in object space to find the intersection point [S1, unofficial].

4.  **UV Coordinate Determination:**
    Two variants exist for determining the UV coordinates on the attribute map:

    *   **Direct Interpolation Variant:**
        Read off attribute parameters directly on the base plane of the splat. Interpolation from the regular grid uses bilinear or higher-order filtering [S1, unofficial].
        *   *Pitfall:* This may lead to large distances of UV coordinates for neighboring fragments, resulting in visual artifacts in close-up views [S1, unofficial].

    *   **Correct Interpolation Variant (Recommended):**
        This variant ensures continuous reconstruction of UV coordinates [S1, unofficial].
        1.  Compute the UV coordinates using the Direct Interpolation Variant as an initial estimate [S1, unofficial].
        2.  Identify the grid points corresponding to the virtual triangle containing the estimated UV point on the support plane [S1, unofficial].
        3.  Determine the plane given by the elevated neighboring points (the 3D positions of the vertices of the virtual triangle) [S1, unofficial].
        4.  Intersect the view ray with this elevated plane [S1, unofficial].
        5.  Project the intersection point onto the support plane to determine the corrected UV coordinates [S1, unofficial].
        6.  Perform final interpolation over the four adjacent samples on the attribute map using bilinear interpolation [S1, unofficial].
        *   *Edge Case:* In rare cases where there is no intersection between the view ray and the plane defined by elevated neighboring points, discard the fragment to prevent visual artifacts [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def continuous_attribute_interpolation(ray_origin, ray_direction, patch):
    """
    ray_origin: vec3
    ray_direction: vec3
    patch: Object containing:
        - support_plane: Plane equation (normal, d)
        - elevation_map: Grid of 3D points (or height values)
        - attribute_map: Texture/Grid of attributes
        - transformation_matrix: 4x4 matrix
        - input_points: List of 3D points defining the patch
        - epsilon: float for overlap handling
    """
    
    # 1. Transform ray to object space if necessary
    # (Assuming patch data is in object space)
    obj_ray_origin = transform_point(patch.transformation_matrix, ray_origin)
    obj_ray_direction = transform_vector(patch.transformation_matrix, ray_direction)
    
    # 2. Intersect ray with the splat's base plane (support plane)
    # This gives the initial UV estimate
    t_base, intersection_base = intersect_ray_plane(obj_ray_origin, obj_ray_direction, patch.support_plane)
    
    if t_base <= 0:
        return None # Behind camera
    
    # 3. Determine UV coordinates on the support plane
    # Project intersection_base onto the support plane to get UV
    uv_direct = project_to_uv(intersection_base, patch.support_plane)
    
    # 4. Identify the virtual triangle containing the UV point
    # Find the 3 grid points in the elevation map that form the triangle containing uv_direct
    grid_points = get_surrounding_grid_points(uv_direct, patch.elevation_map)
    
    if len(grid_points) < 3:
        return None # Degenerate case
    
    # 5. Construct the plane from elevated neighboring points
    # The "elevated" points are the actual 3D positions from the elevation map
    p1_3d = grid_points[0].position_3d
    p2_3d = grid_points[1].position_3d
    p3_3d = grid_points[2].position_3d
    
    # Compute normal of the plane defined by these 3D points
    v1 = p2_3d - p1_3d
    v2 = p3_3d - p1_3d
    plane_normal = normalize(cross(v1, v2))
    
    # Define the plane equation: n . (x - p1) = 0
    elevated_plane = Plane(normal=plane_normal, point=p1_3d)
    
    # 6. Intersect ray with the elevated plane
    t_elevated, intersection_elevated = intersect_ray_plane(obj_ray_origin, obj_ray_direction, elevated_plane)
    
    if t_elevated <= 0:
        return None # No intersection or behind camera
    
    # 7. Project the intersection with the elevated plane onto the support plane
    # This gives the corrected UV coordinates
    uv_corrected = project_to_uv(intersection_elevated, patch.support_plane)
    
    # 8. Bilinear interpolation on the attribute map
    # Sample the attribute map (e.g., normal or texture) at uv_corrected
    attribute = bilinear_sample(patch.attribute_map, uv_corrected)
    
    return attribute

def project_to_uv(point_3d, support_plane):
    """
    Projects a 3D point onto the support plane and returns 2D UV coordinates.
    Assumes a mapping from the support plane's local coordinate system to UV.
    """
    # Project point onto support plane
    # d = support_plane.d
    # n = support_plane.normal
    # t = (d - dot(n, point_3d)) / dot(n, n)
    # projected_point = point_3d + t * n
    
    # Convert projected_point to UV based on the grid's basis vectors
    # This depends on the specific grid layout, but generally involves dot products
    # with the grid's U and V axes.
    pass

def get_surrounding_grid_points(uv, elevation_map):
    """
    Returns the 3 grid points forming the triangle containing the UV coordinate.
    """
    # Find the cell in the grid containing uv
    # Determine which of the two triangles in the cell contains uv
    # Return the 3 vertices of that triangle
    pass
```

## Operating characteristics
*   **Time Complexity:** The per-fragment cost is dominated by the ray-plane intersection and the bilinear texture lookup. The "Correct" variant requires additional geometric calculations (identifying the triangle, computing the elevated plane) but remains constant time per fragment.
*   **Memory Complexity:** Requires storage for elevation maps (VBOs) and attribute maps (textures). The resolution of these maps can be chosen independently [S1, unofficial].
*   **Strengths:** Eliminates visual artifacts caused by discontinuous UV mapping in close-up views [S1, unofficial]. Ensures continuous rendering by handling patch overlaps via $\epsilon$-neighborhoods [S1, unofficial].
*   **Limitations:** The "Correct" variant may discard fragments in rare cases where no intersection with the elevated plane exists [S1, unofficial]. The "Direct" variant is simpler but prone to artifacts [S1, unofficial].
*   **When to use:** When high-quality attribute interpolation (normals, textures) is required for splatted surfaces, especially in close-up views.
*   **When not to use:** If performance is critical and visual artifacts are acceptable, the "Direct" variant or simpler methods may be preferred. If the geometry is flat and attributes are constant, simpler interpolation may suffice.

## Background
The method originates from the work on single-pass high-quality splatting, specifically addressing the challenges of attribute interpolation on partitioned surface patches [S1, unofficial]. It builds on Moving Least Squares (MLS) surface reconstruction techniques [S1, unofficial]. The approach was developed to handle the discontinuities that arise when rendering complex 3D models as splats, ensuring that attributes like normals and textures vary smoothly across the surface [S1, unofficial].

## Sources
[S1] Single-Pass High-Quality Splatting, http://nbn-resolving.de/urn:nbn:de:bsz:352-opus-22395, fetched 2026-10-26 (unofficial)
