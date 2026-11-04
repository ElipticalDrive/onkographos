# Binary Search IsoSurface Extraction

Also called: Bisection Isosurface Extraction, Isovalue Stepping, RayEvaluationSteppingIso

## Overview
Binary Search IsoSurface Extraction is a ray-tracing method that locates the intersection of a ray with an isosurface by iteratively halving the interval between two sample points whose scalar field values bracket the target isovalue [S1, unofficial]. Unlike Trilinear Isosurface Intersection, which solves for the exact intersection within a voxel using linear interpolation, or Hybrid Implicit Isosurface Ray Tracing, which may use analytical root finding, this method relies on discrete sampling and bisection, making it robust to non-linear field variations but potentially less precise per step [S2, unofficial]. It is often used in conjunction with curvature-based shading to enhance visual quality [S2, unofficial].

## How it works
**Inputs:**
- `left`: 3D position vector where the scalar field value is less than or equal to the isovalue.
- `right`: 3D position vector where the scalar field value is greater than the isovalue.
- `iso`: The target isovalue (scalar or batched tensor).
- `texVol`: Volume texture object for sampling scalar values.
- `rec_depth`: Number of binary search iterations (typically 5) [S1, unofficial].

**Outputs:**
- `surf`: 3D position vector approximating the isosurface intersection.

**Data Structures:**
The method operates on 3D vectors (`vec3`) and scalar values (`float`). In GPU implementations (e.g., CUDA), parameters may be packed into a struct for constant memory access [S2, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct Parameters {
    scalar_t stepsize;
    int binarySearchSteps;
    scalar_t isovalue; // or Tensor1Read accessor if batched
    scalar_t isocontourRange;
    cudaTextureObject_t isocontourTexture;
};
```

**Algorithm Steps:**
1. Initialize `left` and `right` such that `value(left) <= iso` and `value(right) > iso`. This bracketing is typically established by a preceding ray-marching or traversal step.
2. Loop `iter` from 0 to `rec_depth - 1`:
   a. Compute midpoint: `mid = (left + right) / 2.0`.
   b. Sample the volume scalar value at `mid`: `curIso = texture(texVol, mid).x`.
   c. If `curIso <= iso`, the surface lies in the right half; update `left = mid`.
   d. Else (`curIso > iso`), the surface lies in the left half; update `right = mid`.
3. After the loop, compute the final surface position: `surf = (left + right) / 2.0` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def binary_search_isosurface(left: vec3, right: vec3, iso: float, texVol: Texture, rec_depth: int = 5) -> vec3:
    """
    Finds the isosurface intersection between left and right using binary search.
    
    Args:
        left: Start point where scalar <= iso.
        right: End point where scalar > iso.
        iso: Target isovalue.
        texVol: Volume texture for sampling.
        rec_depth: Number of bisection iterations.
        
    Returns:
        surf: Approximate intersection point.
    """
    for iter in range(rec_depth):
        mid = (left + right) / 2.0
        curIso = texVol.sample(mid).x  # Sample scalar value at mid
        if curIso <= iso:
            left = mid
        else:
            right = mid
    surf = (left + right) / 2.0
    return surf
```

**Parameters:**
- `rec_depth`: Fixed number of bisection iterations. Default is 5 [S1, unofficial].
- `iso`: The isovalue. Can be a single double or a batched tensor [S2, unofficial].
- `binarySearchSteps`: In some implementations, this controls the refinement steps. Range 0-20. Default 0 [S2, unofficial].
- `isocontourRange`: Range of values for surface feature mapping. Default 5.0f [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Bracketing Failure:** The method assumes `left` and `right` correctly bracket the isovalue. If the initial interval does not contain the surface (e.g., both points are on the same side), the binary search will converge to one of the endpoints or an incorrect location [S1, unofficial].
- **Precision:** The accuracy is limited by `rec_depth`. With 5 iterations, the interval size is reduced by a factor of $2^5 = 32$. For high-precision requirements, more iterations or hybrid methods may be needed [S1, unofficial].
- **Aliasing:** Coverage gaps between sampling points can cause aliasing artifacts. Using methods like Surface Location via Segment Endpoint Values can help mitigate this by leveraging monotonicity [S1, unofficial].

## Subtopics

### [Surface Location via Segment Endpoint Values](3756-surface-location-via-segment-endpoint-values.md)
This method locates isosurface intersections as a function of the scalar field values at the segment endpoints along a ray. It is used within binary search isosurface extraction to eliminate coverage gaps between sampling points that cause aliasing artifacts. Unlike methods that rely on intermediate sampling or fixed-step traversal, it leverages the monotonicity of the field between endpoints to determine surface presence without intermediate checks, provided the binary search interval is sufficiently small.

### [Isocontour Shading with Curvature](3819-isocontour-shading-with-curvature.md)
A shading method for isosurfaces that uses surface features including first curvature ($k_1$), second curvature ($k_2$), mean curvature ($(k_1+k_2)/2$), Gaussian curvature ($k_1*k_2$), or a curvature texture. This is often combined with binary search extraction to enhance visual detail.

### Multi-Stage Rasterization for Shaded Isosurfaces
An adaptation of the Fast Shaded Isosurfaces method for PC hardware using multi-stage rasterization and 2D-textures, enabling efficient rendering of shaded isosurfaces [S3, unofficial].
- **Definition:** Computes voxel gradients and writes them to RGB components of 2D-textures, with intensity in the alpha component.
- **Step:** Uses two general combiner stages. Combiner 0 interpolates intermediate slices. Combiner 1 computes the dot product $A \cdot B$, where $A$ is the interpolated gradient $\tilde{n}$ and $B$ is the light vector $\tilde{l}$ [S3, unofficial].
- **Layout:** Input A is RGB output of Combiner 0. Input B is constant color register 2 (light vector). Alpha-component is not modified by Combiner 1 [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(\log_2(1/\epsilon))$ per ray, where $\epsilon$ is the desired precision. With fixed `rec_depth`, it is $O(1)$ per ray, but the constant factor depends on the number of iterations [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary memory per ray, plus the volume texture storage [S1, unofficial].
- **Performance:** Fast for simple isosurfaces, but may require more samples than analytical methods for high-frequency details. GPU implementations can parallelize across rays efficiently [S2, unofficial].
- **Strengths:** Simple to implement, robust to non-linear fields, easy to integrate with existing ray-marching pipelines [S1, unofficial].
- **Limitations:** Less precise than analytical methods (e.g., Trilinear Isosurface Intersection) for the same computational cost. Sensitive to the initial bracketing interval [S1, unofficial].
- **When to Use:** When the scalar field is not easily invertible or when a simple, robust intersection method is preferred over complex analytical solutions [S1, unofficial].
- **When Not to Use:** When high precision is required and the field is smooth, or when the computational budget is extremely tight and analytical methods are available [S1, unofficial].

## Background
Binary Search IsoSurface Extraction is a fundamental technique in volume rendering, particularly in ray-tracing based approaches. It was popularized in early interactive volume rendering systems where simplicity and robustness were prioritized [S1, unofficial]. The method is often cited in the context of GPU-based volume rendering, where its parallelizable nature makes it suitable for modern graphics hardware [S2, unofficial]. It is also discussed in the context of shading techniques, where it is combined with curvature-based shading to enhance visual quality [S2, unofficial].

## Sources
[S1] ShashwatC/Ray, https://github.com/ShashwatC/Ray, fetched 2026-10-26 (unofficial)
[S2] shamanDevel/fV-SRN, https://github.com/shamanDevel/fV-SRN, fetched 2026-10-26 (unofficial)
[S3] Interactive Volume Rendering on Standard PC Graphics ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/VolVisforPCHardware.pdf, fetched 2026-10-28 (unofficial)
