# Binary Search IsoSurface Extraction

Also called: Bisection Isosurface Extraction, Isovalue Stepping, RayEvaluationSteppingIso

## Overview
Binary Search IsoSurface Extraction approximates the position of an isosurface between two sample points by iteratively halving the interval based on scalar field values [S1, unofficial]. It solves the problem of finding the precise intersection distance $t$ along a ray where the volume scalar value equals a target isovalue, offering higher accuracy than linear interpolation methods like [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md) while avoiding the computational overhead of dense ray marching [S3, unofficial]. Unlike [Hybrid Implicit Isosurface Ray Tracing](3832-hybrid-implicit-isosurface-ray-tracing.md), which may use analytical or hierarchical acceleration structures, this method relies purely on iterative sampling within a bounded interval [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Two 3D points `left` and `right` in world space that bracket the isosurface (i.e., the scalar value at `left` is $\le$ isovalue and at `right` is $>$ isovalue, or vice versa), the target `iso` value, and a volume texture `texVol` [S1, unofficial].
*   **Output:** A 3D point `surf` representing the estimated intersection location [S1, unofficial].

**Data Structures:**
The method operates on standard 3D vectors. In GPU implementations (e.g., CUDA), parameters are often packed into a struct for constant memory access [S2, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct Parameters {
    scalar_t stepsize;
    int binarySearchSteps;
    // If batched: ::kernel::Tensor1Read<scalar_t> isovalue;
    // If scalar: scalar_t isovalue;
    scalar_t isocontourRange;
    cudaTextureObject_t isocontourTexture; // Optional, for curvature shading
};
```

**Algorithm Steps:**
1.  Initialize the recursion depth limit `rec_depth`. The standard implementation fixes this at 5 [S1, unofficial]. Alternative implementations may parameterize this as `binarySearchSteps` with a range of 0–20 [S2, unofficial].
2.  Loop `iter` from 0 to `rec_depth - 1`:
    a. Compute the midpoint: `mid = (right + left) / 2.0` [S1, unofficial].
    b. Sample the scalar value at the midpoint: `curIso = texture(texVol, mid).x` [S1, unofficial].
    c. Compare `curIso` with the target `iso`:
        *   If `curIso <= iso`, the surface lies in the right half; update `left = mid` [S1, unofficial].
        *   Else (`curIso > iso`), the surface lies in the left half; update `right = mid` [S1, unofficial].
3.  After the loop, compute the final surface position as the midpoint of the final interval: `surf = (right + left) / 2.0` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def binary_search_isosurface(left: vec3, right: vec3, iso: float, texVol: Texture, rec_depth: int = 5) -> vec3:
    """
    Finds the isosurface intersection between left and right using binary search.
    
    Args:
        left: Start point of the interval.
        right: End point of the interval.
        iso: Target scalar value.
        texVol: Volume texture for sampling.
        rec_depth: Number of bisection iterations.
        
    Returns:
        surf: Estimated intersection point.
    """
    for iter in range(rec_depth):
        # Compute midpoint
        mid = (right + left) / 2.0
        
        # Sample volume value at midpoint
        # Assuming texture returns a vec4 and we take the first component (x)
        curIso = texVol.sample(mid).x
        
        # Bisection step
        if curIso <= iso:
            left = mid
        else:
            right = mid
            
    # Return final midpoint
    surf = (right + left) / 2.0
    return surf
```

**Parameters:**
*   `rec_depth`: Fixed number of bisection iterations. Default/Fixed value is 5 [S1, unofficial].
*   `iso`: The scalar density value defining the surface. Can be a single double or a batched tensor. Default 0.5 [S2, unofficial].
*   `binarySearchSteps`: Alternative parameter name for `rec_depth`. Range 0–20. Default 0 [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Bracketing:** The method assumes `left` and `right` strictly bracket the isovalue. If the scalar values at both endpoints are on the same side of the isovalue, the logic will converge to one endpoint without finding the surface [G].
*   **Precision:** With `rec_depth = 5`, the interval is reduced by a factor of $2^5 = 32$. For high-resolution volumes, this may not be sufficient for pixel-perfect accuracy, but it is sufficient for visual continuity in many ray-casting contexts [S1, unofficial].
*   **Texture Sampling:** The method relies on `texture(texVol, mid)`. If `mid` falls outside the volume bounds, the sampling result must be handled (e.g., clamping or returning a value outside the isovalue range) to prevent infinite loops or incorrect bisection [G].

## Subtopics

### [Surface Location via Segment Endpoint Values](3756-surface-location-via-segment-endpoint-values.md)
This variant locates isosurface intersections as a function of the scalar field values at the segment endpoints along a ray to eliminate coverage gaps that cause aliasing artifacts. It builds on [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) by using endpoint values to determine surface presence without intermediate sampling checks [G].

### [Isocontour Shading with Curvature](3819-isocontour-shading-with-curvature.md)
Isocontour Shading with Curvature is a shading method for isosurfaces that uses surface features including first curvature ($k_1$), second curvature ($k_2$), mean curvature ($(k_1+k_2)/2$), Gaussian curvature ($k_1 \cdot k_2$), or a curvature texture. It distinguishes itself from sibling methods like [Surface Location via Segment Endpoint Values](3756-surface-location-via-segment-endpoint-values.md) and [Multi-Stage Rasterization for Shaded Isosurfaces](5394-multi-stage-rasterization-for-shaded-isosurfaces.md) by explicitly incorporating curvature data and multi-level isocontour highlighting into the shading kernel. The method builds on [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) to refine surface positions before applying the curvature-aware shading pass.

### [Multi-Stage Rasterization for Shaded Isosurfaces](5394-multi-stage-rasterization-for-shaded-isosurfaces.md)
Multi-Stage Rasterization for Shaded Isosurfaces is a hardware-accelerated technique for rendering shaded isosurfaces from volumetric data on standard PC graphics hardware. It adapts the "Fast Shaded Isosurfaces" algorithm by utilizing multi-stage rasterization and 2D textures to compute surface normals and lighting interactions efficiently. Unlike general volume rendering methods that integrate opacity, this method focuses on extracting and shading a specific isosurface boundary. It is distinct from [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) by replacing iterative ray-marching with a two-pass rasterization pipeline that leverages the GPU's fixed-function register combiners for gradient interpolation and dot-product shading.

### Isosurface Function
A ray function that computes whether a given isovalue $\sigma$ exists along the ray. It produces the same result as marching cubes but with higher accuracy [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(k)$, where $k$ is the number of binary search iterations (typically 5). This is constant time per ray segment, significantly faster than dense ray marching which scales with the number of samples per unit length.
*   **Memory Complexity:** $O(1)$ auxiliary memory per ray, plus the cost of the volume texture access.
*   **Performance:** The method is highly parallelizable and efficient on GPUs due to the lack of branching divergence within a warp if the interval bracketing is consistent, though the `if/else` in the bisection step can cause minor divergence. The fixed depth of 5 iterations makes it predictable for hardware scheduling [S1, unofficial].
*   **Strengths:** Simple to implement; robust convergence if the function is monotonic within the interval; higher accuracy than linear interpolation [S3, unofficial].
*   **Limitations:** Requires a valid bracketing interval; cannot handle multiple intersections within a single segment (it finds only one); accuracy is limited by the number of iterations and the volume's spatial resolution [S1, unofficial].
*   **When to use:** When a quick, reasonably accurate isosurface intersection is needed for a ray segment known to cross the surface, particularly in real-time or interactive visualization contexts [S3, unofficial].
*   **When not to use:** When multiple intersections per segment are possible and must all be found, or when the scalar field is not monotonic within the search interval.

## Background
The method originates from basic numerical analysis techniques for root finding (bisection method) applied to volume rendering. It is cited in educational materials for scientific visualization, such as CS 519 at the University of Illinois Urbana-Champaign, where it is presented as a "ray function" that computes isosurface existence with higher accuracy than marching cubes [S3, unofficial]. It is implemented in open-source ray casting projects like `ShashwatC/Ray` and `shamanDevel/fV-SRN` [S1, unofficial][S2, unofficial].

## Sources
[S1] ShashwatC/Ray, https://github.com/ShashwatC/Ray, fetched 2026-10-26 (unofficial)
[S2] shamanDevel/fV-SRN, https://github.com/shamanDevel/fV-SRN, fetched 2026-10-26 (unofficial)
[S3] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
