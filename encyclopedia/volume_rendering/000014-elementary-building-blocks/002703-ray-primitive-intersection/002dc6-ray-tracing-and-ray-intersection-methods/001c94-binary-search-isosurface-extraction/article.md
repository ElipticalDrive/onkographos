# Binary Search IsoSurface Extraction

Also called: Bisection Isosurface Extraction, Isosurface Raycasting with Bisection, Isovalue Stepping, RayEvaluationSteppingIso

## Overview
Binary Search IsoSurface Extraction approximates the position of an isosurface between two sample points by iteratively halving the interval based on scalar field values [S1, unofficial]. It solves the problem of accurately locating the zero-crossing of a scalar field along a ray without requiring explicit polygon extraction [S1, unofficial]. Unlike [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md), which computes an analytical intersection using trilinear interpolation, this method uses a fixed number of iterative bisection steps to refine the hit position [S3, unofficial]. It is distinct from [Neubauer Iso-surface Ray Casting](7024-neubauer-iso-surface-ray-casting.md) in that it does not rely on specific acceleration structures for traversal but rather refines a detected interval [S3, unofficial].

## How it works
**Inputs:**
*   `left`: 3D point where the scalar field value is less than or equal to the isovalue.
*   `right`: 3D point where the scalar field value is greater than the isovalue.
*   `texVol`: Volume texture object providing scalar field samples.
*   `iso`: The target isovalue (scalar or batched tensor) [S2, unofficial].
*   `rec_depth`: The number of bisection iterations (typically 5) [S1, unofficial].

**Outputs:**
*   `surf`: The 3D position of the isosurface intersection, calculated as the midpoint of the final interval.

**Data Structures:**
*   `vec3`: A 3-component vector type (e.g., `float3` or `glm::vec3`) representing 3D coordinates.
*   `float`: Scalar floating-point type for volume values.
*   `cudaTextureObject_t`: Handle for accessing the volume texture on the GPU [S2, unofficial].

**Parameters:**
*   `rec_depth`: Fixed number of bisection iterations. Default value is 5 [S1, unofficial].
*   `binarySearchSteps`: Number of binary search steps to refine hit position. Range 0-20. Default 0 [S2, unofficial].
*   `isocontourRange`: Range of values $[-v, +v]$ for surface feature mapping. Range 0.01f-500.0f. Default 5.0f [S2, unofficial].
*   `selectedSurfaceFeature`: Enum for shading features (OFF, FIRST_PRINCIPAL_CURVATURE, etc.). Default OFF [S2, unofficial].

**Algorithm Steps:**
1.  Initialize the recursion depth limit `rec_depth` to 5 [S1, unofficial].
2.  Loop `iter` from 0 to `rec_depth - 1`:
    a.  Compute the midpoint: $mid = \frac{right + left}{2.0}$
    b.  Sample the volume scalar value at the midpoint: $curIso = texture(texVol, mid).x$
    c.  If $curIso \le iso$, set $left = mid$ (the surface is in the right half).
    d.  Else, set $right = mid$ (the surface is in the left half).
3.  Set the output surface point: $surf = \frac{right + left}{2.0}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def binary_search_isosurface_extraction(left, right, tex_vol, iso, rec_depth=5):
    """
    Approximates the position of an isosurface between two sample points
    by iteratively halving the interval based on scalar field values.
    
    Args:
        left (vec3): Start point where scalar <= iso.
        right (vec3): End point where scalar > iso.
        tex_vol (TextureObject): Volume texture for sampling.
        iso (float): Target isovalue.
        rec_depth (int): Number of bisection iterations. Default 5.
        
    Returns:
        vec3: The estimated surface intersection point.
    """
    for _ in range(rec_depth):
        # Compute midpoint
        mid = (right + left) / 2.0
        
        # Sample scalar value at midpoint
        # Note: texture() returns a vec4; we take the x component (scalar)
        cur_iso = tex_vol.sample(mid).x
        
        # Bisection step
        if cur_iso <= iso:
            left = mid
        else:
            right = mid
            
    # Return final midpoint
    surf = (right + left) / 2.0
    return surf
```

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on a fixed number of iterations. With `rec_depth=5`, the interval is reduced by a factor of $2^5 = 32$. This may not be sufficient for high-precision requirements or very small features [S3, unofficial].
*   **Boundary Conditions:** The method assumes `left` and `right` correctly bracket the isovalue. If the bracketing condition is violated (e.g., both points are on the same side of the isosurface), the algorithm will converge to a point that is not the intersection [G].
*   **Performance:** Refining already detected intersections has virtually no impact on performance but improves accuracy [S3, unofficial].

## Subtopics

### [Surface Location via Segment Endpoint Values](3756-surface-location-via-segment-endpoint-values.md)
This method locates isosurface intersections as a function of the scalar field values at the segment endpoints along a ray to eliminate coverage gaps that cause aliasing artifacts. It builds on [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) by using endpoint values to determine surface presence without intermediate sampling checks [G].

### [Isocontour Shading with Curvature](3819-isocontour-shading-with-curvature.md)
Isocontour Shading with Curvature is a shading method for isosurfaces that uses surface features including first curvature ($k_1$), second curvature ($k_2$), mean curvature ($(k_1+k_2)/2$), Gaussian curvature ($k_1 \cdot k_2$), or a curvature texture. It distinguishes itself from sibling methods like [Surface Location via Segment Endpoint Values](3756-surface-location-via-segment-endpoint-values.md) and [Multi-Stage Rasterization for Shaded Isosurfaces](5394-multi-stage-rasterization-for-shaded-isosurfaces.md) by explicitly incorporating curvature data and multi-level isocontour highlighting into the shading kernel. The method builds on [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) to refine surface positions before applying the curvature-aware shading pass.

### [Multi-Stage Rasterization for Shaded Isosurfaces](5394-multi-stage-rasterization-for-shaded-isosurfaces.md)
Multi-Stage Rasterization for Shaded Isosurfaces adapts the Fast Shaded Isosurfaces method for PC hardware using multi-stage rasterization and 2D textures to efficiently render shaded isosurfaces. It replaces iterative ray-marching with a two-pass register combiner pipeline that interpolates voxel gradients and computes dot-product lighting, extracting a specific isosurface boundary instead of integrating opacity.

### [IsoSurface Ray Marching](11315-isosurface-ray-marching.md)
IsoSurface Ray Marching is a ray casting technique that renders an isosurface by sampling a 3D volume scalar field and detecting zero crossings relative to a threshold value. It avoids explicit polygon extraction by refining intersection positions via bisection and shading with Phong lighting. Unlike its sibling [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md), which approximates position by iteratively halving an interval, this method integrates the marching process directly into the ray traversal loop with a fixed step size, refining only upon crossing detection.

### Isosurface Function
A ray function that computes whether a given isovalue $\sigma$ exists along the ray. Produces the same result as marching cubes but with higher accuracy [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(k)$ where $k$ is the number of bisection steps (fixed constant, typically 5).
*   **Memory Complexity:** $O(1)$ auxiliary space.
*   **Measured Performance:** At a sampling distance of 1.0, performance is 24 fps; at a sampling distance of 5.0, performance is 66 fps [S3, unofficial].
*   **Strengths:** Virtually no impact on performance for refinement; improves accuracy for small features and silhouette regions [S3, unofficial].
*   **Limitations:** Fixed number of steps may not adapt to feature size; requires initial bracketing of the isosurface.
*   **When to Use:** When high accuracy is needed for already detected intersections or when analytical intersection is too complex [S3, unofficial].
*   **When Not to Use:** When adaptive sampling is required for varying feature sizes across the volume [S3, unofficial].

## Background
Binary Search IsoSurface Extraction is a technique used in GPU-based volume raycasting to refine isosurface intersections. It was discussed in the context of advanced illumination techniques for volume rendering [S3, unofficial]. The method is often used in conjunction with curvature-based shading to enhance visual quality [S2, unofficial]. It provides a higher accuracy alternative to marching cubes for ray-casted isosurfaces [S4, unofficial].

## Sources
[S1] ShashwatC/Ray, https://github.com/ShashwatC/Ray, fetched 2026-10-26 (unofficial)
[S2] shamanDevel/fV-SRN, https://github.com/shamanDevel/fV-SRN, fetched 2026-10-26 (unofficial)
[S3] GPU-Based Volume Ray-Casting with Advanced Illumination, https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2008/advanced_illumination.pdf, fetched 2026-10-29 (unofficial)
[S4] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
