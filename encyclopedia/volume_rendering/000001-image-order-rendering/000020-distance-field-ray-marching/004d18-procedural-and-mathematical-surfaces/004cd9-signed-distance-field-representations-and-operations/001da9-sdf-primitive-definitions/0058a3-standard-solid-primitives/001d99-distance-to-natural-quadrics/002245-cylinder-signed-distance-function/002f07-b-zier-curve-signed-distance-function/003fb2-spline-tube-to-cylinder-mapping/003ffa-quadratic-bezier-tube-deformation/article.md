# Quadratic Bezier Tube Deformation

Also called: Bezier Curve Deformation, Quadratic Bezier Tube

## Overview
Quadratic Bezier Tube Deformation maps points from an arbitrary spline tube defined by quadratic Bezier segments to a canonical vertical cylinder, enabling the use of pre-defined density fields [10061-spline-tube-to-cylinder-mapping.md]. It solves the inverse parameterization problem in volumetric rendering by providing a robust Signed Distance Function (SDF) to the curve, which yields the curvilinear abscissa required for texture coordinate calculation [S1, unofficial]. It is distinguished from cubic spline approaches by its computational simplicity in finding the closest point and curvilinear abscissa, though it imposes stricter constraints on segment concatenation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $P$ in world space; a ray origin and direction for bounding box culling; a list of quadratic Bezier segments defining the curve [S1, unofficial]. Each segment $i$ is defined by three control points $A_i, B_i, C_i$ [S1, unofficial].
*   **Output:** The deformed coordinates $(u, v, w)$ in the canonical cylinder space, where $u$ and $v$ are angular/radial coordinates and $w$ is the curvilinear abscissa (normalized height) [10061-spline-tube-to-cylinder-mapping.md][S1, unofficial].

**Data Structures:**
*   **Bezier Segment:** Defined by three 3D vectors $A, B, C$ [S1, unofficial].
    *   $A$: Start point ($t=0$).
    *   $B$: Control point (pulls the curve).
    *   $C$: End point ($t=1$).
*   **Segment List:** An ordered array of segments [S1, unofficial]. For concatenation, the start of segment $i+1$ must equal the end of segment $i$ ($A_{i+1} = C_i$) [S1, unofficial].

**Formulas:**
The quadratic Bezier curve for a segment is defined as:
$$ Q(t) = (1-t)^2 A + 2(1-t)t B + t^2 C, \quad t \in [0, 1] $$
The segment starts at $A$ when $t=0$, finishes at $C$ when $t=1$, and is pulled toward $B$ in between [S1, unofficial]. The tangent at $t=0$ is parallel to vector $\vec{AB} = B - A$, and the tangent at $t=1$ is parallel to vector $\vec{BC} = C - B$ [S1, unofficial].

To find the deformation, we must find the closest point on the curve to $P$ [G]. This is achieved by computing the SDF to the quadratic Bezier segment [S1, unofficial]. The SDF implementation (based on Quilez 2013) returns both the distance and the parameter $t$ of the closest point, which corresponds to the curvilinear abscissa [S1, unofficial].

**Steps:**
1.  **Initialize:** Set `min_dist` to infinity and `best_t` to 0 [G].
2.  **Iterate Segments:** For each segment $i$ in the curve:
    a.  **Optimization (Ray Casting):** Perform a ray-casting test against the bounding box of segment $i$ relative to the ray origin [S1, unofficial]. If the ray does not intersect the bounding box, skip this segment [S1, unofficial]. Cache this result per pixel to avoid re-evaluation during ray marching [S1, unofficial].
    b.  **SDF Evaluation:** Compute the SDF from point $P$ to the quadratic Bezier segment $i$ [G]. Let the result be $d_i$ and the closest point parameter be $t_i$ [G].
    c.  **Update Minimum:** If $d_i < \text{min\_dist}$, set $\text{min\_dist} = d_i$ and $\text{best\_t} = t_i$ [G].
3.  **Map to Cylinder:** Using the best $t$ found, map the point to the canonical cylinder [10061-spline-tube-to-cylinder-mapping.md]. The height $w$ is normalized based on the total length of the curve or the segment's contribution [S1, unofficial]. The radial and angular coordinates are derived from the vector from the closest point on the curve to $P$ [10061-spline-tube-to-cylinder-mapping.md].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def quadratic_bezier_sdf(P, A, B, C):
    """Computes SDF to a quadratic Bezier segment. Returns (distance, t). [G]"""
    def Q(t):
        u = 1.0 - t
        return ((u*u)*A[0] + 2.0*u*t*B[0] + t*t*C[0],
                (u*u)*A[1] + 2.0*u*t*B[1] + t*t*C[1],
                (u*u)*A[2] + 2.0*u*t*B[2] + t*t*C[2])
    def dQ(t):
        u = 1.0 - t
        return (2.0*u*(B[0]-A[0]) + 2.0*t*(C[0]-B[0]),
                2.0*u*(B[1]-A[1]) + 2.0*t*(C[1]-B[1]),
                2.0*u*(B[2]-A[2]) + 2.0*t*(C[2]-B[2]))
    t = 0.5
    for _ in range(10):
        diff = (P[0]-Q(t)[0], P[1]-Q(t)[1], P[2]-Q(t)[2])
        dot = diff[0]*dQ(t)[0] + diff[1]*dQ(t)[1] + diff[2]*dQ(t)[2]
        if abs(dot) < 1e-6: break
        ddot = dQ(t)[0]**2 + dQ(t)[1]**2 + dQ(t)[2]**2 + \
               diff[0]*2.0*(C[0]-2.0*B[0]+A[0]) + diff[1]*2.0*(C[1]-2.0*B[1]+A[1]) + diff[2]*2.0*(C[2]-2.0*B[2]+A[2])
        if abs(ddot) < 1e-12: break
        t -= dot / ddot
        t = max(0.0, min(1.0, t))
    dist = math.sqrt((P[0]-Q(t)[0])**2 + (P[1]-Q(t)[1])**2 + (P[2]-Q(t)[2])**2)
    return dist, t

def get_bounding_box(A, B, C):
    """Computes axis-aligned bounding box for quadratic Bezier segment. [G]"""
    denom_x = A[0] - 2.0*B[0] + C[0]
    t_ext_x = max(0.0, min(1.0, (A[0]-B[0])/denom_x)) if denom_x != 0 else 0.0
    denom_y = A[1] - 2.0*B[1] + C[1]
    t_ext_y = max(0.0, min(1.0, (A[1]-B[1])/denom_y)) if denom_y != 0 else 0.0
    denom_z = A[2] - 2.0*B[2] + C[2]
    t_ext_z = max(0.0, min(1.0, (A[2]-B[2])/denom_z)) if denom_z != 0 else 0.0
    
    mins = [min(A[0], B[0], C[0]), min(A[1], B[1], C[1]), min(A[2], B[2], C[2])]
    maxs = [max(A[0], B[0], C[0]), max(A[1], B[1], C[1]), max(A[2], B[2], C[2])]
    
    for t_ext in [t_ext_x, t_ext_y, t_ext_z]:
        if 0.0 < t_ext < 1.0:
            u = 1.0 - t_ext
            ext = (u*u)*A[0] + 2.0*u*t_ext*B[0] + t_ext*t_ext*C[0]
            ext_y = (u*u)*A[1] + 2.0*u*t_ext*B[1] + t_ext*t_ext*C[1]
            ext_z = (u*u)*A[2] + 2.0*u*t_ext*B[2] + t_ext*t_ext*C[2]
            mins[0] = min(mins[0], ext); mins[1] = min(mins[1], ext_y); mins[2] = min(mins[2], ext_z)
            maxs[0] = max(maxs[0], ext); maxs[1] = max(maxs[1], ext_y); maxs[2] = max(maxs[2], ext_z)
    return mins, maxs

def ray_intersects_bbox(ray_origin, ray_direction, mins, maxs):
    """Checks if ray intersects AABB using slab method. Returns True. [G]"""
    eps = 1e-8
    tmin = (mins[0] - ray_origin[0]) / max(abs(ray_direction[0]), eps)
    tmax = (maxs[0] - ray_origin[0]) / max(abs(ray_direction[0]), eps)
    if tmin > tmax: tmin, tmax = tmax, tmin
    tymin = (mins[1] - ray_origin[1]) / max(abs(ray_direction[1]), eps)
    tymax = (maxs[1] - ray_origin[1]) / max(abs(ray_direction[1]), eps)
    if tymin > tymax: tymin, tymax = tymax, tymin
    if (tmin > tymax) or (tymin > tmax): return False
    if tymin > tmin: tmin = tymin
    if tymax < tmax: tmax = tymax
    tzmin = (mins[2] - ray_origin[2]) / max(abs(ray_direction[2]), eps)
    tzmax = (maxs[2] - ray_origin[2]) / max(abs(ray_direction[2]), eps)
    if tzmin > tzmax: tzmin, tzmax = tzmax, tzmin
    if (tmin > tzmax) or (tzmin > tmax): return False
    if tzmin > tmin: tmin = tzmin
    return tmax >= 0.0

def deform_point(P, ray_origin, ray_direction, segments, pixel_cache):
    """Deforms point P to cylinder coordinates using quadratic Bezier segments."""
    min_dist = float('inf')
    best_t = 0.0
    best_seg = None
    cache_key = (ray_origin[0], ray_origin[1], ray_origin[2], ray_direction[0], ray_direction[1], ray_direction[2])
    
    for i, seg in enumerate(segments):
        A, B, C = seg
        if cache_key in pixel_cache:
            if not pixel_cache[cache_key].get(i, True):
                continue
        bbox_mins, bbox_maxs = get_bounding_box(A, B, C)
        intersects = ray_intersects_bbox(ray_origin, ray_direction, bbox_mins, bbox_maxs)
        if cache_key not in pixel_cache:
            pixel_cache[cache_key] = {}
        pixel_cache[cache_key][i] = intersects
        if not intersects:
            continue
        dist, t = quadratic_bezier_sdf(P, A, B, C)
        if dist < min_dist:
            min_dist = dist
            best_t = t
            best_seg = seg
            
    A, B, C = best_seg
    inv_t = 1.0 - best_t
    closest_point = ((inv_t*inv_t)*A[0] + 2.0*inv_t*best_t*B[0] + best_t*best_t*C[0],
                     (inv_t*inv_t)*A[1] + 2.0*inv_t*best_t*B[1] + best_t*best_t*C[1],
                     (inv_t*inv_t)*A[2] + 2.0*inv_t*best_t*B[2] + best_t*best_t*C[2])
    vec = (P[0]-closest_point[0], P[1]-closest_point[1], P[2]-closest_point[2])
    u = math.atan2(vec[1], vec[0])
    v = math.sqrt(vec[0]**2 + vec[2]**2)
    w = best_t
    return u, v, w
```

**Parameters:**
*   **Ray Casting Cache:** Enabled per pixel [S1, unofficial].
*   **Bounding Box Check:** Used for early rejection [S1, unofficial]. Note: The source indicates that a complex bounding box calculation was removed in favor of simpler ray-casting because the former was slower on average [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Concatenation:** Quadratic Bezier segments are difficult to concatenate smoothly [S1, unofficial]. Only one control point ($B$) is "free" per segment [S1, unofficial]. $A$ is fixed by the previous segment's $C$ [S1, unofficial]. $B$ must be constrained to ensure derivative continuity [S1, unofficial]. Modifying one segment affects all subsequent segments [S1, unofficial].
*   **Performance:** The SDF evaluation scales linearly with the number of segments [S1, unofficial]. Without bounding box/ray-casting optimization, this is expensive [S1, unofficial]. The ray-caching optimization is critical for performance in volumetric rendering [S1, unofficial].
*   **SDF Availability:** The method relies on the availability of an SDF for quadratic Bezier segments that returns the curvilinear abscissa [S1, unofficial]. This is a key reason for choosing quadratic over cubic splines [S1, unofficial].

## Subtopics

### [Curve-based Volumetric Free-Form Deformation](14160-curve-based-volumetric-free-form-deformation.md)
Uses open uniform B-splines instead of Bezier curves. It constructs a lattice of curves in world space to avoid inverse parameterization issues, allowing flexible shape manipulation on the CPU. Choose this when smooth, global deformation of complex volumes (like fire) is needed and CPU-side evaluation is acceptable.

## Operating characteristics
*   **Time Complexity:** Linear $O(N)$ with respect to the number of Bezier segments, where $N$ is the number of segments [S1, unofficial]. With ray-casting optimization, the constant factor is significantly reduced, often evaluating only a small subset of segments per ray step [S1, unofficial].
*   **Memory Complexity:** Low [S1, unofficial]. Requires storing control points for the segments and a small cache per pixel for bounding box intersection results [S1, unofficial].
*   **Strengths:** Computationally simpler than cubic splines for finding the closest point and curvilinear abscissa [S1, unofficial]. Robust SDF implementation available [S1, unofficial].
*   **Limitations:** Difficult to concatenate segments smoothly [S1, unofficial]. Modifications to one segment propagate to all subsequent segments [S1, unofficial]. Bounding box calculation overhead can negate benefits if not optimized (ray-casting is preferred) [S1, unofficial].
*   **When to Use:** When a simple, robust deformation is needed for procedural volumes and the curve structure is relatively static or simple [S1, unofficial].
*   **When Not to Use:** When high-degree smoothness across many segments is required, or when the curve topology changes frequently [S1, unofficial].

## Background
The technique was developed for the real-time rendering of interstellar dust clouds and solar flares [S1, unofficial]. Quadratic Bezier curves were chosen over cubic splines because solving for the closest point and curvilinear abscissa is computationally simpler, and a suitable SDF implementation returning the curvilinear abscissa was available (Quilez 2013) [S1, unofficial]. The method was implemented in a renderer using Sphere Marching for bounding box detection [S1, unofficial]. A reference implementation for bounding box computation by Quilez (2020) was tested but found to be slower than simple ray-casting, leading to its removal [S1, unofficial].

## Sources
[S1] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
