# Continuous Attribute Interpolation

Also called: Bilinear Interpolation Attribute Sampling, Correct Interpolation Variant, Direct Interpolation Variant, Geometry and Attribute Sampling, MLS Attribute Sampling

## Overview
Continuous Attribute Interpolation is a method for obtaining continuous reconstruction of UV coordinates for attribute interpolation on splats, specifically designed to avoid visual artifacts at splat boundaries [S1, unofficial]. It solves the problem of discontinuous attribute sampling that arises when intersecting view rays with flat splat planes, which can cause large UV coordinate jumps for neighboring fragments [S1, unofficial]. It sets itself apart from naive direct interpolation by using a "correct interpolation" variant that projects intersections onto a plane defined by elevated neighboring points, ensuring smooth attribute transitions [S1, unofficial].

## How it works
The method operates on a model represented as a set of elevation and attribute maps, where each patch corresponds to a piece of the 3D model parameterized as a height field over a planar domain [S1, unofficial]. The pipeline operates in a single rendering pass, storing elevation maps in Vertex Buffer Objects (VBOs) and attribute maps in texture objects [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   View ray origin and direction.
    *   Splat geometry: A set of input points defining the patch, including their 3D positions and attributes (e.g., normals, texture coordinates) [G].
    *   Patch metadata: A $4 \times 4$ transformation matrix per patch [S1, unofficial].
    *   Attribute maps: Regular grids storing resampled attributes (normals, textures) [S1, unofficial].
*   **Outputs:**
    *   Corrected UV coordinates for the fragment [S1, unofficial].
    *   Interpolated attribute values (normals, textures) at the fragment location [S1, unofficial].

### Data Structures
*   **Elevation Map:** A regular grid stored in a VBO [S1, unofficial]. Each cell corresponds to a sample on the support plane [G].
*   **Attribute Map:** A regular grid (texture) storing resampled attributes [S1, unofficial]. Resolution is chosen independently per map [S1, unofficial].
*   **Patch Data:** For each splat, the system maintains the set of input points and the associated support plane [G].

### Steps

1.  **Patch Partitioning and Overlap Handling:**
    The 3D model is partitioned into regions, each corresponding to a surface patch [S1, unofficial]. To ensure continuous rendering and avoid gaps between patches, patches are made to slightly overlap [S1, unofficial]. An input point is included in a region if its $\epsilon$-neighborhood intersects the point set of that region, where $\epsilon > 0$ is a small practical value [S1, unofficial].

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
        - transform_matrix: 4x4 matrix (object space)
        - support_plane: Plane with normal (vec3), d (float), u_axis (vec3), v_axis (vec3)
        - elevation_map: 2D grid of vec3 positions
        - attribute_map: 2D grid of attribute values (vec3 or vec4)
        - resolution: vec2 (grid dimensions)
    """
    # 1. Transform ray to object space
    obj_ray_origin = transform_point(patch.transform_matrix, ray_origin)
    obj_ray_direction = transform_vector(patch.transform_matrix, ray_direction)

    # 2. Intersect with base plane to get initial UV estimate
    t_base, intersection_base = intersect_ray_plane(obj_ray_origin, obj_ray_direction, patch.support_plane)
    if t_base <= 0:
        return None

    # 3. Project to support plane for initial UV
    uv_direct = project_to_uv(intersection_base, patch.support_plane)

    # 4. Find grid points for the virtual triangle
    grid_points = get_surrounding_grid_points(uv_direct, patch.elevation_map, patch.resolution)
    if len(grid_points) < 3:
        return None

    # 5. Compute elevated plane from 3D positions
    p1, p2, p3 = grid_points
    v1 = p2 - p1
    v2 = p3 - p1
    plane_normal = normalize(cross(v1, v2))
    elevated_plane = Plane(normal=plane_normal, point=p1)

    # 6. Intersect ray with elevated plane
    t_elevated, intersection_elevated = intersect_ray_plane(obj_ray_origin, obj_ray_direction, elevated_plane)
    if t_elevated <= 0:
        return None

    # 7. Project to support plane for corrected UV
    uv_corrected = project_to_uv(intersection_elevated, patch.support_plane)

    # 8. Bilinear interpolation on attribute map
    attribute = bilinear_sample(patch.attribute_map, uv_corrected, patch.resolution)
    return attribute

def transform_point(matrix, point):
    w = dot(matrix[3], point)
    return vec3(
        dot(matrix[0], point) / w,
        dot(matrix[1], point) / w,
        dot(matrix[2], point) / w
    )

def transform_vector(matrix, vector):
    return vec3(
        dot(matrix[0], vector),
        dot(matrix[1], vector),
        dot(matrix[2], vector)
    )

def intersect_ray_plane(origin, direction, plane):
    denom = dot(plane.normal, direction)
    if abs(denom) < 1e-6:
        return 0, None
    t = dot(plane.normal, plane.point - origin) / denom
    return t, origin + t * direction

def project_to_uv(point, plane):
    # Orthogonal projection onto support plane
    t = dot(plane.normal, plane.point - point) / dot(plane.normal, plane.normal)
    projected = point + t * plane.normal
    # Extract UV coordinates using plane's local basis
    u = dot(projected, plane.u_axis)
    v = dot(projected, plane.v_axis)
    return vec2(u, v)

def get_surrounding_grid_points(uv, elevation_map, resolution):
    # Normalize UV to grid coordinates
    gx = floor(uv.x * resolution.x)
    gy = floor(uv.y * resolution.y)
    # Clamp to grid bounds
    gx = max(0, min(gx, resolution.x - 2))
    gy = max(0, min(gy, resolution.y - 2))
    # Retrieve 4 corners of the grid cell
    c00 = elevation_map[gx, gy]
    c10 = elevation_map[gx + 1, gy]
    c01 = elevation_map[gx, gy + 1]
    c11 = elevation_map[gx + 1, gy + 1]
    # Determine which triangle contains the UV point (standard diagonal split)
    if (uv.x * resolution.x - gx) + (uv.y * resolution.y - gy) < 1.0:
        return [c00, c10, c01]
    else:
        return [c10, c11, c01]

def bilinear_sample(texture, uv, resolution):
    # Normalize UV to texture coordinates [0, 1]
    u = uv.x / resolution.x
    v = uv.y / resolution.y
    # Compute sample coordinates
    x0 = floor(u * (resolution.x - 1))
    y0 = floor(v * (resolution.y - 1))
    x1 = min(x0 + 1, resolution.x - 1)
    y1 = min(y0 + 1, resolution.y - 1)
    # Interpolation weights
    fx = u * (resolution.x - 1) - x0
    fy = v * (resolution.y - 1) - y0
    # Fetch values
    v00 = texture[x0, y0]
    v10 = texture[x1, y0]
    v01 = texture[x0, y1]
    v11 = texture[x1, y1]
    # Bilinear interpolation
    top = lerp(v00, v10, fx)
    bottom = lerp(v01, v11, fx)
    return lerp(top, bottom, fy)

def lerp(a, b, t):
    return a + t * (b - a)

def normalize(v):
    return v / length(v)

def cross(a, b):
    return vec3(
        a.y * b.z - a.z * b.y,
        a.z * b.x - a.x * b.z,
        a.x * b.y - a.y * b.x
    )

def dot(a, b):
    return a.x * b.x + a.y * b.y + a.z * b.z

def length(v):
    return sqrt(dot(v, v))
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Eliminates visual artifacts caused by discontinuous UV mapping in close-up views [S1, unofficial]. Ensures continuous rendering by handling patch overlaps via $\epsilon$-neighborhoods [S1, unofficial].
*   **Limitations:** The "Correct" variant may discard fragments in rare cases where no intersection with the elevated plane exists [S1, unofficial]. The "Direct" variant is simpler but prone to artifacts [S1, unofficial].
*   **When to use:** When high-quality attribute interpolation (normals, textures) is required for splatted surfaces, especially in close-up views [S1, unofficial].
*   **When not to use:** Not given in the sources.

## Background
The method originates from the work on single-pass high-quality splatting, specifically addressing the challenges of attribute interpolation on partitioned surface patches [S1, unofficial]. It employs Moving Least Squares (MLS) surface reconstruction for resampling surface patches onto a regular parameter plane [S1, unofficial]. The approach was developed to handle the discontinuities that arise when rendering complex 3D models as splats, ensuring that attributes like normals and textures vary smoothly across the surface [S1, unofficial].

## Sources
[S1] Single-Pass High-Quality Splatting, http://nbn-resolving.de/urn:nbn:de:bsz:352-opus-22395, fetched 2026-10-26 (unofficial)
