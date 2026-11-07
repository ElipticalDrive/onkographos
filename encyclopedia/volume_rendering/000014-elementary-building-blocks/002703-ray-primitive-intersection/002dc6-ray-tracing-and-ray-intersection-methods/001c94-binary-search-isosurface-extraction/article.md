# Binary Search IsoSurface Extraction

Also called: Bisection Isosurface Extraction, Isovalue Stepping, RayEvaluationSteppingIso

## Overview
Binary Search IsoSurface Extraction approximates the position of an isosurface between two sample points by iteratively halving the interval based on scalar field values [S1, unofficial]. It solves the problem of finding the precise intersection of a ray with a volume's isovalue surface without requiring explicit geometric primitives, serving as a high-accuracy alternative to marching cubes [S3, unofficial]. Unlike [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md) which uses linear interpolation, this method uses recursive bisection to converge on the root, and unlike [Hybrid Implicit Isosurface Ray Tracing](3832-hybrid-implicit-isosurface-ray-tracing.md) which may use analytical derivatives, it relies solely on sampled scalar values [S1, unofficial].

## How it works
**Inputs:**
- `left`: 3D vector (`vec3`) representing the starting point of the search interval, typically the entry point into the volume or the previous sample.
- `right`: 3D vector (`vec3`) representing the end point of the search interval, typically the exit point or the next sample.
- `iso`: The target isovalue (scalar density) defining the surface. Can be a single scalar or a batched tensor [S2, unofficial].
- `texVol`: A 3D texture object containing the volumetric scalar field [S1, unofficial].
- `rec_depth`: Integer specifying the number of binary search iterations. Fixed at 5 in the primary implementation [S1, unofficial].

**Outputs:**
- `surf`: 3D vector (`vec3`) representing the estimated intersection point on the isosurface [S1, unofficial].

**Data Structures:**
In GPU implementations (e.g., CUDA), parameters are often packed into a struct for constant memory access [S2, unofficial].
- **Scalar Case Struct:**
  - `stepsize`: `scalar_t` (e.g., float/double)
  - `binarySearchSteps`: `int` (Range 0-20, Default 0) [S2, unofficial]
  - `isovalue`: `scalar_t` (Default 0.5) [S2, unofficial]
  - `isocontourRange`: `scalar_t` (Range 0.01f-500.0f, Default 5.0f) [S2, unofficial]
  - `isocontourTexture`: `cudaTextureObject_t` (Optional, for curvature shading) [S2, unofficial]
- **Batched Case Struct:**
  - `stepsize`: `scalar_t`
  - `binarySearchSteps`: `int`
  - `isovalue`: `::kernel::Tensor1Read<scalar_t>` (Tensor of shape `(B,)`) [S2, unofficial]
  - `isocontourRange`: `scalar_t`
  - `isocontourTexture`: `cudaTextureObject_t`

**Algorithm Steps:**
1. Initialize the search interval with `left` and `right` boundaries.
2. Loop `iter` from 0 to `rec_depth - 1` (inclusive):
   a. Compute the midpoint: `mid = (right + left) / 2.0`.
   b. Sample the volume scalar value at `mid`: `curIso = texture(texVol, mid).x`.
   c. Compare `curIso` with `iso`:
      - If `curIso <= iso`, the surface lies in the right half (or at mid); update `left = mid`.
      - Else (`curIso > iso`), the surface lies in the left half; update `right = mid`.
3. After the loop, compute the final surface position as the midpoint of the final interval: `surf = (right + left) / 2.0` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def binary_search_isosurface_extraction(left: Vec3, right: Vec3, iso: float, tex_vol: Texture3D, rec_depth: int = 5) -> Vec3:
    """
    Finds the intersection of a ray segment [left, right] with the isosurface defined by 'iso'.
    
    Args:
        left: Start point of the interval.
        right: End point of the interval.
        iso: Target scalar value.
        tex_vol: 3D texture object for volume sampling.
        rec_depth: Number of bisection iterations.
        
    Returns:
        surf: Estimated intersection point.
    """
    # Initialize boundaries
    current_left = left
    current_right = right
    
    for iter in range(rec_depth):
        # 1. Compute midpoint
        mid = (current_right + current_left) / 2.0
        
        # 2. Sample scalar value at midpoint
        # Assuming texture lookup returns a vector where .x is the scalar value
        cur_iso = tex_vol.sample(mid).x
        
        # 3. Bisection logic
        if cur_iso <= iso:
            # Surface is in the right half (or at mid)
            current_left = mid
        else:
            # Surface is in the left half
            current_right = mid
            
    # 4. Return final midpoint
    surf = (current_right + current_left) / 2.0
    return surf
```

**Parameters:**
- `rec_depth`: Fixed at 5 iterations in the reference implementation [S1, unofficial]. This provides a precision of $2^{-5} = 1/32$ of the initial interval length.
- `isovalue`: Default 0.5 [S2, unofficial].
- `binarySearchSteps`: Optional refinement steps (0-20), default 0 [S2, unofficial].
- `isocontourRange`: Range for surface feature mapping (0.01f-500.0f), default 5.0f [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Bracketing Requirement:** The method assumes the isovalue is bracketed by the scalar values at `left` and `right` (i.e., one is $\le$ iso and the other is $>$ iso). If both endpoints are on the same side of the isovalue, the algorithm will converge to one of the endpoints or an incorrect location [S1, unofficial].
- **Precision:** The result is limited by the number of iterations. With `rec_depth=5`, the error is bounded by $L/32$ where $L$ is the initial interval length [S1, unofficial].
- **Texture Sampling:** Requires a valid 3D texture object. If `mid` falls outside the volume bounds, sampling behavior (clamp, repeat, or zero) affects correctness [S1, unofficial].

## Subtopics

### [Surface Location via Segment Endpoint Values](3756-surface-location-via-segment-endpoint-values.md)
This variant locates isosurface intersections as a function of the scalar field values at the segment endpoints along a ray to eliminate coverage gaps that cause aliasing artifacts. It builds on Binary Search IsoSurface Extraction by using endpoint values to determine surface presence without intermediate sampling checks.

### [Isocontour Shading with Curvature](3819-isocontour-shading-with-curvature.md)
Isocontour Shading with Curvature is a shading method for isosurfaces that uses surface features including first curvature ($k_1$), second curvature ($k_2$), mean curvature ($(k_1+k_2)/2$), Gaussian curvature ($k_1 \cdot k_2$), or a curvature texture. It distinguishes itself from sibling methods by explicitly incorporating curvature data and multi-level isocontour highlighting into the shading kernel. It uses parameters like `selectedSurfaceFeature` (OFF, FIRST_PRINCIPAL_CURVATURE, etc.) and `numIsocontours` (0-10, default 5).

### [Multi-Stage Rasterization for Shaded Isosurfaces](5394-multi-stage-rasterization-for-shaded-isosurfaces.md)
Multi-Stage Rasterization for Shaded Isosurfaces is a hardware-accelerated technique for rendering shaded isosurfaces from volumetric data on standard PC graphics hardware. It adapts the "Fast Shaded Isosurfaces" algorithm by utilizing multi-stage rasterization and 2D textures to compute surface normals and lighting interactions efficiently. Unlike general volume rendering methods that integrate opacity, this method focuses on extracting and shading a specific isosurface boundary. It is distinct from Binary Search IsoSurface Extraction by replacing iterative ray-marching with a two-pass rasterization pipeline that leverages the GPU's fixed-function register combiners for gradient interpolation and dot-product shading.

### Isosurface Function
A ray function that computes whether a given isovalue $\sigma$ exists along the ray. It produces the same result as marching cubes but with higher accuracy [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(k)$ where $k$ is the number of binary search iterations (fixed at 5 in the primary implementation) [S1, unofficial]. Each iteration requires one texture lookup and basic arithmetic.
- **Memory Complexity:** $O(1)$ auxiliary memory per ray, plus the volume texture storage [S1, unofficial].
- **Performance:** The fixed iteration count makes it predictable and suitable for parallel execution on GPUs [S1, unofficial]. It is slower than linear interpolation methods like Trilinear Isosurface Intersection but more robust to non-linear scalar fields [S1, unofficial].
- **Strengths:** High accuracy for smooth scalar fields; simple implementation; no derivative calculation required [S1, unofficial].
- **Limitations:** Requires the isovalue to be bracketed by the interval endpoints; precision is limited by iteration count; may miss thin structures if the initial interval is too large [S1, unofficial].
- **When to Use:** When high precision is needed for smooth isosurfaces and the volume data is accessible via texture sampling [S1, unofficial].
- **When Not to Use:** When the scalar field is highly discontinuous or when the bracketing condition cannot be guaranteed [S1, unofficial].

## Background
Binary Search IsoSurface Extraction originates from ray evaluation stepping methods in volume rendering [S2, unofficial]. It is described in the context of scientific visualization lectures as a method to compute isosurfaces with higher accuracy than marching cubes [S3, unofficial]. The specific implementation with `rec_depth=5` is found in the "Ray" project by ShashwatC [S1, unofficial]. The method is also integrated into the fV-SRN renderer for GPU-based volume rendering with optional curvature shading [S2, unofficial].

## Sources
[S1] ShashwatC/Ray, https://github.com/ShashwatC/Ray, fetched 2026-10-26 (unofficial)
[S2] shamanDevel/fV-SRN, https://github.com/shamanDevel/fV-SRN, fetched 2026-10-26 (unofficial)
[S3] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
