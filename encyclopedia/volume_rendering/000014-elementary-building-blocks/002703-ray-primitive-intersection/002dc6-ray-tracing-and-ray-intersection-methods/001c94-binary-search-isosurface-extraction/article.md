# Binary Search IsoSurface Extraction

Also called: Bisection Isosurface Extraction, Isosurface Raycasting with Bisection, Isovalue Stepping, RayEvaluationSteppingIso

## Overview
Binary Search IsoSurface Extraction approximates the position of an isosurface between two sample points by iteratively halving the interval based on scalar field values [S1, S3, unofficial]. It solves the problem of locating the precise intersection of a ray with a scalar volume's isovalue surface without requiring analytical derivatives or complex root-finding algorithms [S1, S2, unofficial]. Unlike [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md), which assumes linear variation within a voxel, this method uses discrete sampling and bisection, making it robust for non-linear or noisy data but potentially slower due to multiple texture lookups [S1, S3, unofficial]. It serves as the geometric foundation for [Isocontour Shading with Curvature](3819-isocontour-shading-with-curvature.md) and [Surface Location via Segment Endpoint Values](3756-surface-location-via-segment-endpoint-values.md) [S1, S2, unofficial].

## How it works
**Inputs:**
*   `left`: 3D vector representing the start of the search interval (typically the ray entry point or previous sample).
*   `right`: 3D vector representing the end of the search interval (typically the ray exit point or next sample).
*   `iso`: The scalar isovalue defining the surface. Can be a single scalar or a batched tensor [S1, S2, unofficial].
*   `texVol`: A 3D texture object providing access to the volume scalar field [S1, S2, unofficial].
*   `rec_depth` (or `binarySearchSteps`): The fixed number of bisection iterations [S1, S2, S3, unofficial].

**Outputs:**
*   `surf`: A 3D vector representing the estimated intersection point on the isosurface [S1, unofficial].

**Data Structures:**
In GPU implementations (e.g., CUDA), parameters are often packed into a struct for constant memory access [S2, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct Parameters {
    scalar_t stepsize;
    int binarySearchSteps;
    // If batched: ::kernel::Tensor1Read<scalar_t> isovalue;
    // If scalar: scalar_t isovalue;
    scalar_t isocontourRange;
    cudaTextureObject_t isocontourTexture; // For curvature features
};
```

**Algorithm Steps:**
1.  Initialize the interval boundaries `left` and `right` [S1, unofficial].
2.  Loop `iter` from 0 to `rec_depth - 1` [S1, unofficial]:
    a. Compute the midpoint: `mid = (left + right) / 2.0` [S1, unofficial].
    b. Sample the scalar value at the midpoint: `curIso = texture(texVol, mid).x` [S1, unofficial].
    c. Compare `curIso` with `iso`:
       - If `curIso <= iso`, the surface is in the right half (or at mid); set `left = mid` [S1, unofficial].
       - Else (`curIso > iso`), the surface is in the left half; set `right = mid` [S1, unofficial].
3.  Return the final surface position as the midpoint of the final interval: `surf = (left + right) / 2.0` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def binary_search_isosurface(left: vec3, right: vec3, iso: float, texVol: Texture3D, rec_depth: int = 5) -> vec3:
    """
    Finds the isosurface intersection between left and right using binary search.
    
    Args:
        left: Start point of the interval.
        right: End point of the interval.
        iso: The target isovalue.
        texVol: 3D texture object for volume sampling.
        rec_depth: Number of bisection iterations.
        
    Returns:
        surf: Estimated intersection point.
    """
    for iter in range(rec_depth):
        mid = (left + right) / 2.0
        cur_iso = texVol.sample(mid).x  # Assuming .x accesses the scalar component
        
        if cur_iso <= iso:
            left = mid
        else:
            right = mid
            
    surf = (left + right) / 2.0
    return surf
```

**Parameters:**
*   `rec_depth`: Fixed number of bisection iterations. Default is 5 [S1, S3, unofficial]. This provides a precision of $2^{-5} = 1/32$ of the initial interval length.
*   `binarySearchSteps`: Alternative parameter name for the iteration count. Range 0-20. Default 0 (if 0, no refinement is performed) [S2, unofficial].
*   `iso`: Default 0.5 [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Bracketing:** The method assumes the isovalue is bracketed by `left` and `right` (i.e., one side is $\le iso$ and the other is $> iso$). If the volume does not cross the isovalue within the interval, the result is arbitrary [S1, unofficial].
*   **Precision:** The accuracy is limited by the number of iterations and the sampling resolution. [S3, unofficial] notes that adaptive sampling may be required for small features or silhouettes where standard sampling fails [S3, unofficial].
*   **Performance:** Each iteration requires a texture lookup. [S3, unofficial] reports that at a sampling distance of 1.0, performance is 24 fps, while at 5.0, it is 66 fps, indicating sensitivity to the number of steps and sampling density [S3, unofficial].

## Subtopics

### [Surface Location via Segment Endpoint Values](3756-surface-location-via-segment-endpoint-values.md)
This variant locates isosurface intersections as a function of the scalar field values at the segment endpoints along a ray to eliminate coverage gaps that cause aliasing artifacts. It builds on [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) by using endpoint values to determine surface presence without intermediate sampling checks [G].

### [Isocontour Shading with Curvature](3819-isocontour-shading-with-curvature.md)
Isocontour Shading with Curvature is a shading method for isosurfaces that uses surface features including first curvature ($k_1$), second curvature ($k_2$), mean curvature ($(k_1+k_2)/2$), Gaussian curvature ($k_1 \cdot k_2$), or a curvature texture. It distinguishes itself from sibling methods like [Surface Location via Segment Endpoint Values](3756-surface-location-via-segment-endpoint-values.md) and [Multi-Stage Rasterization for Shaded Isosurfaces](5394-multi-stage-rasterization-for-shaded-isosurfaces.md) by explicitly incorporating curvature data and multi-level isocontour highlighting into the shading kernel. The method builds on [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) to refine surface positions before applying the curvature-aware shading pass.

### [Multi-Stage Rasterization for Shaded Isosurfaces](5394-multi-stage-rasterization-for-shaded-isosurfaces.md)
Multi-Stage Rasterization for Shaded Isosurfaces adapts the Fast Shaded Isosurfaces method for PC hardware using multi-stage rasterization and 2D textures to efficiently render shaded isosurfaces. It replaces iterative ray-marching with a two-pass register combiner pipeline that interpolates voxel gradients and computes dot-product lighting, extracting a specific isosurface boundary instead of integrating opacity.

### Isosurface Function
A ray function that computes whether a given isovalue $\sigma$ exists along the ray. It produces the same result as marching cubes but with higher accuracy [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(k)$, where $k$ is the number of bisection steps (typically 5-20) [S1, S2, S3, unofficial]. Each step involves a constant-time texture lookup and arithmetic operations.
*   **Memory Complexity:** $O(1)$ auxiliary memory per ray, plus the volume texture storage [S1, S2, unofficial].
*   **Performance:** [S3, unofficial] reports 24 fps at sampling distance 1.0 and 66 fps at sampling distance 5.0 on the tested hardware [S3, unofficial]. The method has virtually no impact on performance when used for refinement [S3, unofficial].
*   **Strengths:** Simple to implement; robust for non-linear fields; does not require gradient computation [S1, S3, unofficial].
*   **Limitations:** Slower than analytical methods if high precision is required (many iterations); susceptible to aliasing if the interval is too large or sampling is sparse [S1, S3, unofficial].
*   **When to use:** When the volume data is discrete or noisy, and high precision is needed without computing gradients [S1, S3, unofficial].
*   **When not to use:** When real-time performance is critical and the field is smooth enough for linear interpolation [S1, unofficial].

## Background
The method originates from early volume ray-casting techniques that sought to accurately locate isosurfaces without the geometric complexity of marching cubes [S4, unofficial]. It was further refined in the context of GPU-based volume ray-casting with advanced illumination techniques, where it was used to refine intersection points detected by coarser methods [S3, unofficial]. The approach is widely used in scientific visualization for its simplicity and reliability [S1, S4, unofficial].

## Sources
[S1] ShashwatC/Ray, https://github.com/ShashwatC/Ray, fetched 2026-10-26 (unofficial)
[S2] shamanDevel/fV-SRN, https://github.com/shamanDevel/fV-SRN, fetched 2026-10-26 (unofficial)
[S3] GPU-Based Volume Ray-Casting with Advanced Illumination, https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2008/advanced_illumination.pdf, fetched 2026-10-29 (unofficial)
[S4] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
