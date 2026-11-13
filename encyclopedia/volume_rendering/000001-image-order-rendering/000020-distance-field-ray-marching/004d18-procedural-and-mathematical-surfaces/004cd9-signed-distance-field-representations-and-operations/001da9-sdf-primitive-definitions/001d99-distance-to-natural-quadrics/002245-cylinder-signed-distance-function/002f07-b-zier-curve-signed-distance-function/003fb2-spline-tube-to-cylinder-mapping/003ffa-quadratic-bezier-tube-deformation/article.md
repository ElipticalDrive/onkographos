# Quadratic Bezier Tube Deformation

Also called: Bezier Curve Deformation, Quadratic Bezier Tube

## Overview
Quadratic Bezier Tube Deformation is a technique for mapping points from an arbitrary spline tube defined by quadratic Bezier segments to a canonical vertical cylinder, enabling the use of pre-defined density fields. It solves the problem of inverse parameterization in volumetric rendering by providing a robust Signed Distance Function (SDF) to the curve, which yields the curvilinear abscissa required for texture coordinate calculation. It is distinguished from cubic spline approaches by its computational simplicity in finding the closest point and curvilinear abscissa, though it imposes stricter constraints on segment concatenation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $P$ in world space; a list of quadratic Bezier segments defining the curve. Each segment $i$ is defined by three control points $A_i, B_i, C_i$.
*   **Output:** The deformed coordinates $(u, v, w)$ in the canonical cylinder space, where $u$ and $v$ are angular/radial coordinates and $w$ is the curvilinear abscissa (normalized height).

**Data Structures:**
*   **Bezier Segment:** Defined by three 3D vectors $A, B, C$.
    *   $A$: Start point ($t=0$).
    *   $B$: Control point (pulls the curve).
    *   $C$: End point ($t=1$).
*   **Segment List:** An ordered array of segments. For concatenation, the start of segment $i+1$ must equal the end of segment $i$ ($A_{i+1} = C_i$).

**Formulas:**
The quadratic Bezier curve for a segment is defined as:
$$ Q(t) = (1-t)^2 A + 2(1-t)t B + t^2 C, \quad t \in [0, 1] $$
The tangent at $t=0$ is parallel to vector $\vec{AB} = B - A$. The tangent at $t=1$ is parallel to vector $\vec{BC} = C - B$.

To find the deformation, we must find the closest point on the curve to $P$. This is achieved by computing the SDF to the quadratic Bezier segment. The SDF implementation (based on Quilez 2013) returns both the distance and the parameter $t$ of the closest point, which corresponds to the curvilinear abscissa.

**Steps:**
1.  **Initialize:** Set `min_dist` to infinity and `best_t` to 0.
2.  **Iterate Segments:** For each segment $i$ in the curve:
    a.  **Optimization (Ray Casting):** Perform a ray-casting test against the bounding box of segment $i$ relative to the ray origin. If the ray does not intersect the bounding box, skip this segment. Cache this result per pixel to avoid re-evaluation during ray marching [S1, unofficial].
    b.  **SDF Evaluation:** Compute the SDF from point $P$ to the quadratic Bezier segment $i$. Let the result be $d_i$ and the closest point parameter be $t_i$.
    c.  **Update Minimum:** If $d_i < \text{min\_dist}$, set $\text{min\_dist} = d_i$ and $\text{best\_t} = t_i$.
3.  **Map to Cylinder:** Using the best $t$ found, map the point to the canonical cylinder. The height $w$ is normalized based on the total length of the curve or the segment's contribution. The radial and angular coordinates are derived from the vector from the closest point on the curve to $P$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quadratic_bezier_sdf_segment(P, A, B, C):
    """
    Computes SDF to a quadratic Bezier segment.
    Returns (distance, t) where t is the parameter of the closest point.
    Based on Quilez 2013 implementation logic.
    """
    # Implementation details for quadratic Bezier SDF are assumed available
    # as per the source reference [S1, unofficial].
    # This function solves for the closest point on the curve Q(t).
    return distance, t

def get_bounding_box(A, B, C):
    """
    Computes the axis-aligned bounding box for the quadratic Bezier segment.
    """
    # Calculate extrema of the quadratic function for each axis
    # This is a standard geometric computation for Bezier bounds.
    # Note: The source notes that a specific reference implementation 
    # by Quilez 2020 was found to be slower than simple ray-casting,
    # so this function is used primarily for the ray-casting check.
    pass

def ray_intersects_bounding_box(ray_origin, ray_direction, bbox):
    """
    Checks if a ray intersects an AABB.
    Returns True if intersection is possible.
    """
    # Standard slab method for ray-AABB intersection
    pass

def deform_point(P, ray_origin, ray_direction, segments, pixel_cache):
    """
    Deforms point P to cylinder coordinates using quadratic Bezier segments.
    """
    min_dist = float('inf')
    best_t = 0.0
    best_segment_idx = -1
    
    # Optimization: Check bounding boxes
    # Cache results per pixel to avoid redundant ray-casting during marching
    for i, seg in enumerate(segments):
        A, B, C = seg
        
        # Check cache
        cache_key = (i, ray_origin, ray_direction) # Simplified key
        if cache_key in pixel_cache:
            if not pixel_cache[cache_key]:
                continue
        
        bbox = get_bounding_box(A, B, C)
        intersects = ray_intersects_bounding_box(ray_origin, ray_direction, bbox)
        
        # Cache the result
        pixel_cache[cache_key] = intersects
        
        if not intersects:
            continue
            
        # Compute SDF
        dist, t = quadratic_bezier_sdf_segment(P, A, B, C)
        
        if dist < min_dist:
            min_dist = dist
            best_t = t
            best_segment_idx = i
            
    # Map to cylinder space
    # The exact mapping of t to w depends on the specific spline tube definition
    # and normalization used in the parent method [10061-spline-tube-to-cylinder-mapping.md].
    # Typically, w is normalized by the total curve length or segment length.
    
    # Get the closest point on the curve
    # Q(t) = (1-t)^2 * A + 2*(1-t)*t * B + t^2 * C
    inv_t = 1.0 - best_t
    closest_point = (inv_t * inv_t) * A + (2.0 * inv_t * best_t) * B + (best_t * best_t) * C
    
    # Vector from curve to point
    vec = P - closest_point
    
    # Cylinder coordinates (u, v, w)
    # u, v are derived from vec (e.g., polar coordinates)
    # w is derived from best_t and segment index
    u = atan2(vec.y, vec.x) # Example projection
    v = length(vec.xz)      # Example radius
    w = best_t              # Normalized parameter
    
    return u, v, w
```

**Parameters:**
*   **Ray Casting Cache:** Enabled per pixel.
*   **Bounding Box Check:** Used for early rejection. Note: The source indicates that a complex bounding box calculation was removed in favor of simpler ray-casting because the former was slower on average [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Concatenation:** Quadratic Bezier segments are difficult to concatenate smoothly. Only one control point ($B$) is "free" per segment. $A$ is fixed by the previous segment's $C$. $B$ must be constrained to ensure derivative continuity. Modifying one segment affects all subsequent segments [S1, unofficial].
*   **Performance:** The SDF evaluation scales linearly with the number of segments. Without bounding box/ray-casting optimization, this is expensive. The ray-caching optimization is critical for performance in volumetric rendering [S1, unofficial].
*   **SDF Availability:** The method relies on the availability of an SDF for quadratic Bezier segments that returns the curvilinear abscissa. This is a key reason for choosing quadratic over cubic splines [S1, unofficial].

## Subtopics

### [Curve-based Volumetric Free-Form Deformation](14160-curve-based-volumetric-free-form-deformation.md)
Uses open uniform B-splines instead of Bezier curves. It constructs a lattice of curves in world space to avoid inverse parameterization issues, allowing flexible shape manipulation on the CPU. Choose this when smooth, global deformation of complex volumes (like fire) is needed and CPU-side evaluation is acceptable.

## Operating characteristics
*   **Time Complexity:** Linear $O(N)$ with respect to the number of Bezier segments, where $N$ is the number of segments. With ray-casting optimization, the constant factor is significantly reduced, often evaluating only a small subset of segments per ray step.
*   **Memory Complexity:** Low. Requires storing control points for the segments and a small cache per pixel for bounding box intersection results.
*   **Strengths:** Computationally simpler than cubic splines for finding the closest point and curvilinear abscissa. Robust SDF implementation available.
*   **Limitations:** Difficult to concatenate segments smoothly. Modifications to one segment propagate to all subsequent segments. Bounding box calculation overhead can negate benefits if not optimized (ray-casting is preferred).
*   **When to Use:** When a simple, robust deformation is needed for procedural volumes and the curve structure is relatively static or simple.
*   **When Not to Use:** When high-degree smoothness across many segments is required, or when the curve topology changes frequently.

## Background
The technique was developed for the real-time rendering of interstellar dust clouds and solar flares. Quadratic Bezier curves were chosen over cubic splines because solving for the closest point and curvilinear abscissa is computationally simpler, and a suitable SDF implementation returning the curvilinear abscissa was available (Quilez 2013). The method was implemented in a renderer using Sphere Marching for bounding box detection. A reference implementation for bounding box computation by Quilez (2020) was tested but found to be slower than simple ray-casting, leading to its removal [S1, unofficial].

## Sources
[S1] Sculpting procedural noise: real-time rendering of interstellar dust clouds and solar flares, https://openalex.org/W4293584834, fetched 2026-10-26 (unofficial)
