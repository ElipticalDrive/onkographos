# Probabilistic Culling

Also called: Approximate Intersection Culling, Approximate Tile-Wise Culling, Ibrahim et al, Probabilistic Culling for Particle Rendering

## Overview
Probabilistic culling is a visibility optimization that intentionally allows false-negatives in intersection tests to reduce duplicated fragments and rendering time, particularly in high-resolution or VR contexts [S2]. It sets itself apart from conservative culling methods by trading strict geometric accuracy for performance gains, relying on the fact that missing a small number of fragments often results in visually imperceptible differences [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of rendering primitives (e.g., splats, particles, or tiles) and a target framebuffer resolution.
*   **Output:** A subset of primitives passed to the rasterization stage, with a reduced count of generated fragments.

**Data Structures:**
Not given in the sources. The method operates on tile-wise or primitive-level intersection data.

**Steps:**
1.  Perform an approximate intersection test between the view frustum (or tile bounds) and the primitive's bounding volume.
2.  If the test returns true, the primitive is processed. If the test returns false, the primitive is discarded, even if it might geometrically intersect the view [S2].
3.  The approximation is designed such that the probability of a false-negative is low enough that the resulting image error is imperceptible [S2].

**Formulas:**
Not given in the sources. The method relies on heuristic or statistical approximations for intersection rather than exact mathematical bounds.

**Parameters:**
*   **Approximation Level:** Not explicitly defined, but the method is described as "approximate tile-wise culling" [S2].
*   **Framebuffer Size:** The impact of the method scales with resolution. It shows little impact at 1920x1080 but significant impact in VR [S2].

**Edge Cases and Pitfalls:**
*   **Low Resolution:** At default framebuffer sizes (e.g., 1920x1080), the reduction in sorting/rasterization workload is offset by a slight increase in fragment creation duration, making the culling less effective [S2].
*   **Visual Quality:** While metrics suggest imperceptibility, the method inherently reduces visual fidelity by omitting fragments [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def probabilistic_cull(primitive, tile_bounds):
    """
    Performs an approximate intersection test.
    Returns True if the primitive is likely visible, False otherwise.
    Note: This may return False even if the primitive intersects (false-negative).
    """
    # The specific approximation algorithm is not detailed in the sources.
    # It is described as "approximate tile-wise culling" [S2].
    # Implementation would involve a fast, potentially conservative or 
    # statistical check against tile_bounds.
    
    # Placeholder for the specific approximation logic [G]
    is_approximately_visible = check_approximate_intersection(primitive, tile_bounds)
    
    return is_approximately_visible
```

## Subtopics

### [Parameter Range Culling](5071-parameter-range-culling.md)
Parameter Range Culling (PRC) is a runtime visibility optimization that isolates particle subsets by culling particles whose scalar parameter values (e.g., density, velocity) do not overlap a user-defined valid range. It sets itself apart from static culling by performing these checks dynamically during traversal, allowing for interactive exploration of simulation data without pre-processing.

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources. The method reduces the number of fragments processed, thereby lowering rasterization and sorting costs.

**Measured Performance:**
*   **Image Quality:** Image difference metrics show renderings with correct and approximate intersection tests are virtually indistinguishable. For the Berlin scene: PSNR 41.83 dB, SSIM 0.9996, FLIP 0.0010, LPIPS 0.0021 [S2].
*   **Fragment Reduction:**
    *   **Berlin (Desktop):** Fragments reduced from 6.3M (tight) to 5.9M. Rendering time from 2.7ms to 2.6ms [S2].
    *   **Berlin (VR):** Fragments reduced from 12.8M to 9.8M. Rendering time from 5.6ms to 5.2ms [S2].
    *   **Campus (VR):** Fragments reduced from 28.5M to 25.7M. Rendering time from 16.5ms to 14.1ms [S2].
*   **Scaling:** The impact of approximate culling increases with framebuffer size and is especially prominent in VR rendering, where the pixel count is significantly higher (e.g., 4.36x more pixels in VR vs desktop) [S2].

**Strengths and Limitations:**
*   **Strengths:** Significant reduction in duplicated fragments and rendering time in high-resolution and VR scenarios [S2].
*   **Limitations:** Little impact at default desktop resolutions (1920x1080) due to overhead in fragment creation balancing out rasterization savings [S2].

**When to Use:**
Use in VR or high-resolution rendering where fragment count is a bottleneck and minor visual artifacts are acceptable [S2].

**When Not to Use:**
Avoid in low-resolution desktop rendering where the overhead may negate benefits, or in applications requiring strict geometric accuracy [S2].

## Background
**Origin:**
The method is attributed to Ibrahim et al. [IRR*22] and is cited in the context of efficient particle rendering using task and mesh shader pipelines [S1, S2].

**History and Development:**
The technique is discussed in the context of "Splatshop," an efficient editing tool for large Gaussian Splat models, where it is referred to as "Approximate Tile-Wise Culling" [S2]. It is also referenced in work on efficient direct isosurface rasterization, though that work distinguishes itself by using conservative rather than probabilistic culling [S1].

**Usage:**
Used in real-time rendering of large particle systems and Gaussian splats, particularly in VR [S1, S2].

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
[S2] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
