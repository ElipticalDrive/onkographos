# Binary Search IsoSurface Extraction

Also called: Bisection Isosurface Extraction, Isovalue Stepping, RayEvaluationSteppingIso

## Overview
Binary Search IsoSurface Extraction approximates the position of an isosurface between two sample points by iteratively halving the interval based on scalar field values [S1, unofficial]. It solves the problem of locating the precise intersection of a ray with an implicit surface defined by a scalar volume, offering higher accuracy than coarse marching methods while remaining simpler than analytical root finding [S3, unofficial]. It distinguishes itself from [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md) by using iterative bisection rather than solving the trilinear equation analytically, and from [Hybrid Implicit Isosurface Ray Tracing](3832-hybrid-implicit-isosurface-ray-tracing.md) by not requiring spatial acceleration structures for the search phase [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**:
    *   `left`: 3D point where the scalar field value is less than or equal to the isovalue.
    *   `right`: 3D point where the scalar field value is greater than the isovalue.
    *   `iso`: The target isovalue (scalar or batched tensor) [S1, unofficial][S2, unofficial].
    *   `texVol`: The volume texture object containing the scalar field [S1, unofficial].
    *   `rec_depth`: The number of binary search iterations (typically 5) [S1, unofficial].
*   **Outputs**:
    *   `surf`: The 3D position of the estimated isosurface intersection [S1, unofficial].

**Data Structures**
The method relies on standard 3D vector types (`vec3`) and scalar types (`float`). In GPU implementations (CUDA), parameters are often packed into a struct for constant memory access [S2, unofficial]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct Parameters {
    scalar_t stepsize;
    int binarySearchSteps;
    // If isovalue is batched:
    ::kernel::Tensor1Read<scalar_t> isovalue;
    // If isovalue is scalar:
    // scalar_t isovalue; 
    scalar_t isocontourRange;
    cudaTextureObject_t isocontourTexture;
};
```

**Algorithm Steps**
1.  Initialize the interval boundaries `left` and `right` such that $V(\mathbf{left}) \le \text{iso}$ and $V(\mathbf{right}) > \text{iso}$, where $V$ is the sampled scalar value [S1, unofficial].
2.  Loop `iter` from $0$ to `rec_depth` - 1:
    a.  Compute the midpoint: $\mathbf{mid} = \frac{\mathbf{right} + \mathbf{left}}{2.0}$ [S1, unofficial].
    b.  Sample the scalar value at the midpoint: $v_{mid} = \text{texture}(\text{texVol}, \mathbf{mid}).x$ [S1, unofficial].
    c.  Compare $v_{mid}$ with `iso`:
        *   If $v_{mid} \le \text{iso}$, the surface lies in the right half; update $\mathbf{left} = \mathbf{mid}$ [S1, unofficial].
        *   Else, the surface lies in the left half; update $\mathbf{right} = \mathbf{mid}$ [S1, unofficial].
3.  Return the final surface position as the midpoint of the final interval: $\mathbf{surf} = \frac{\mathbf{right} + \mathbf{left}}{2.0}$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def binary_search_isosurface_extraction(left: vec3, right: vec3, iso: float, texVol: Texture, rec_depth: int = 5) -> vec3:
    """
    Finds the isosurface intersection between left and right using binary search.
    
    Args:
        left: Point where scalar value <= iso.
        right: Point where scalar value > iso.
        iso: Target isovalue.
        texVol: Volume texture for sampling.
        rec_depth: Number of bisection iterations.
        
    Returns:
        surf: Estimated intersection point.
    """
    for iter in range(rec_depth):
        # Calculate midpoint
        mid = (right + left) / 2.0
        
        # Sample scalar value at midpoint
        # Assuming texture access returns a vector, take the first component
        cur_iso = texVol.sample(mid).x
        
        # Bisection step
        if cur_iso <= iso:
            left = mid
        else:
            right = mid
            
    # Final surface position
    surf = (right + left) / 2.0
    return surf
```

**Parameters**
*   `rec_depth`: Fixed number of bisection iterations. Default/Fixed value is 5 [S1, unofficial].
*   `iso`: The scalar density value defining the surface. Can be a single double or a batched tensor. Default 0.5 [S2, unofficial].
*   `binarySearchSteps`: In some implementations, this controls the refinement steps. Range 0-20. Default 0 (implying fixed depth or separate logic) [S2, unofficial].

**Edge Cases and Pitfalls**
*   **Bracketing Requirement**: The method assumes `left` and `right` strictly bracket the isovalue ($V(\mathbf{left}) \le \text{iso} < V(\mathbf{right})$). If the initial interval does not bracket the surface, the algorithm will converge to a local extremum or boundary rather than the intersection [G].
*   **Precision**: The accuracy is limited by `rec_depth`. With 5 iterations, the interval size is reduced by a factor of $2^5 = 32$. For high-resolution volumes, this may still result in visible aliasing if the initial step size is large [S1, unofficial].
*   **Texture Sampling**: The method relies on `texture(texVol, mid)`. If `mid` falls outside the volume bounds, the result depends on the texture addressing mode (clamp, repeat, or border color) [G].

## Subtopics

### [Surface Location via Segment Endpoint Values](3756-surface-location-via-segment-endpoint-values.md)
This method locates isosurface intersections as a function of the scalar field values at the segment endpoints along a ray to eliminate coverage gaps that cause aliasing artifacts. It builds on [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) by using endpoint values to determine surface presence without intermediate sampling checks [G].

### [Isocontour Shading with Curvature](3819-isocontour-shading-with-curvature.md)
Isocontour Shading with Curvature is a shading method for isosurfaces that uses surface features including first curvature ($k_1$), second curvature ($k_2$), mean curvature ($(k_1+k_2)/2$), Gaussian curvature ($k_1 \cdot k_2$), or a curvature texture. It distinguishes itself from sibling methods like [Surface Location via Segment Endpoint Values](3756-surface-location-via-segment-endpoint-values.md) and [Multi-Stage Rasterization for Shaded Isosurfaces](5394-multi-stage-rasterization-for-shaded-isosurfaces.md) by explicitly incorporating curvature data and multi-level isocontour highlighting into the shading kernel. The method builds on [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) to refine surface positions before applying the curvature-aware shading pass.

### [Multi-Stage Rasterization for Shaded Isosurfaces](5394-multi-stage-rasterization-for-shaded-isosurfaces.md)
Multi-Stage Rasterization for Shaded Isosurfaces adapts the Fast Shaded Isosurfaces method for PC hardware using multi-stage rasterization and 2D textures to efficiently render shaded isosurfaces. It replaces iterative ray-marching with a two-pass register combiner pipeline that interpolates voxel gradients and computes dot-product lighting, extracting a specific isosurface boundary instead of integrating opacity.

### Isosurface Function
A ray function that computes whether a given isovalue $\sigma$ exists along the ray. It produces the same result as marching cubes but with higher accuracy [S3, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(k)$, where $k$ is the number of binary search iterations (typically 5). This is constant time per ray intersection, independent of volume resolution, but dependent on the initial interval size for convergence quality [S1, unofficial].
*   **Memory Complexity**: $O(1)$ auxiliary memory per thread. The volume data is accessed via texture memory [S1, unofficial].
*   **Performance**: Very fast for single-surface extraction. The fixed iteration count makes it predictable for GPU execution [S1, unofficial].
*   **Strengths**: Simple to implement; robust convergence if the function is monotonic within the interval; higher accuracy than linear interpolation [S3, unofficial].
*   **Limitations**: Requires a valid bracketing interval; may miss thin structures if the initial step size is too large; does not provide surface normals directly (requires gradient sampling) [G].
*   **When to use**: When high precision is needed for a single isosurface and the volume is well-sampled; when analytical solutions are too complex [S3, unofficial].
*   **When not to use**: When multiple isosurfaces need to be extracted simultaneously (use Marching Cubes or similar); when the scalar field is not monotonic within the search interval [G].

## Background
The method originates from the general class of ray evaluation stepping techniques in volume visualization [S2, unofficial]. It is described in educational contexts as a way to achieve higher accuracy than marching cubes by using software or hardware ray casting [S3, unofficial]. The specific implementation with `rec_depth = 5` is found in open-source ray tracing examples [S1, unofficial]. The CUDA-based implementation with curvature shading is part of the fV-SRN renderer [S2, unofficial].

## Sources
[S1] ShashwatC/Ray, https://github.com/ShashwatC/Ray, fetched 2026-10-26 (unofficial)
[S2] shamanDevel/fV-SRN, https://github.com/shamanDevel/fV-SRN, fetched 2026-10-26 (unofficial)
[S3] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
