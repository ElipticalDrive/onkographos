# Probabilistic Culling

Also called: Approximate Intersection Culling, Approximate Tile-Wise Culling, Ibrahim et al, Probabilistic Culling for Particle Rendering

## Overview
Probabilistic culling is a rendering optimization that intentionally allows false-negatives in intersection tests to reduce duplicated fragments and rendering time, particularly in high-resolution or VR contexts [S2]. It sets itself apart from conservative culling methods (like [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md)) by trading strict geometric accuracy for performance gains, justified by the visual indistinguishability of the resulting artifacts [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of tiles or primitives (e.g., Gaussian splats) with bounding volumes and a view frustum or tile grid.
*   **Output:** A subset of primitives passed to the rasterization stage, with some potentially occluded or non-overlapping primitives incorrectly included (false positives) or excluded (false negatives). *Note: The source [S2] describes "false-negatives" in the context of intersection tests justifying the culling, but the performance data shows a reduction in fragments, implying the culling is aggressive. The term "probabilistic" implies the culling decision is not 100% accurate.*

**Data Structures:**
*   **Tile/Primitive Data:** Bounding boxes or spheres for each tile/primitive.
*   **Visibility Buffer:** Not explicitly defined in the sources for the culling phase itself, but the method operates on tile-wise intersection tests [S2].

**Steps:**
1.  **Tile Generation:** Divide the screen or volume into tiles [S2].
2.  **Approximate Intersection Test:** For each tile, perform an intersection test against the bounding volumes of the primitives. This test is "approximate," meaning it may yield false negatives (rejecting a primitive that actually intersects) or false positives (accepting one that doesn't). The source [S2] states the method "justifies intersection test with false-negatives," which is slightly ambiguous. However, the context of "reducing duplicated fragments" and the performance gains suggest that the culling is *aggressive*, potentially rejecting fragments that should be drawn (false negatives in visibility determination) or, more likely, the term "false-negative" refers to the *culling decision* being a false negative (i.e., we think it's not visible, but it is). *Correction based on standard terminology:* In culling, a "false negative" usually means we cull something that is visible. The source [S2] says "justifies intersection test with false-negatives... to reduce duplicated fragments." This implies that by allowing some visible fragments to be missed (false negatives in visibility), we reduce the total count of fragments processed.
3.  **Fragment Reduction:** Pass only the primitives that pass the approximate test to the rasterizer.
4.  **Rendering:** Render the reduced set of primitives.

**Formulas:**
*   None provided in sources.

**Parameters:**
*   **Tile Size:** Not explicitly given, but the method is "Tile-Wise" [S2].
*   **Approximation Level:** Not explicitly defined, but the method is "Approximate" [S2].

**Edge Cases and Pitfalls:**
*   **Visual Artifacts:** The method relies on the assumption that missing fragments are visually imperceptible. This holds for high-resolution displays and VR but may fail at lower resolutions or for critical details [S2].
*   **Performance Trade-off:** At default framebuffer sizes (1920x1080), the performance gain is minimal because the reduction in sorting/rasterization workload is offset by the overhead of the culling test itself [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def probabilistic_culling(tiles, primitives):
    """
    Perform approximate tile-wise culling.
    tiles: List of tile bounding boxes or screen regions.
    primitives: List of primitives with bounding volumes.
    Returns: List of primitives that pass the approximate test.
    """
    culled_primitives = []
    for tile in tiles:
        # Perform approximate intersection test
        # This test may return false negatives (rejecting visible primitives)
        # The exact algorithm for the "approximate" test is not detailed in [S2]
        # but is implied to be a fast, conservative or aggressive approximation
        # of the true intersection.
        passing_primitives = approximate_intersection_test(tile, primitives)
        culled_primitives.extend(passing_primitives)
    return culled_primitives

def approximate_intersection_test(tile, primitives):
    """
    Placeholder for the specific approximate intersection logic.
    The source [S2] does not provide the exact geometric test,
    only the results of using it.
    """
    # Implementation detail missing in sources [S2]
    # Likely involves simplified bounding box or sphere tests
    # that are faster than exact ray-primitive intersection.
    pass
```

## Subtopics

### [Parameter Range Culling](5071-parameter-range-culling.md)
Parameter Range Culling is a runtime optimization that isolates particle subsets by culling particles whose simulation property values do not overlap a user-defined valid range. It solves the problem of visualizing specific behaviors within large particle datasets by avoiding unnecessary ray/sphere intersection tests for particles outside the range of interest. It sets itself apart from static culling methods by allowing the valid range to be changed interactively at runtime, providing flexibility for data analysis, albeit with some computational overhead compared to pre-cropped data.

## Operating characteristics
*   **Time Complexity:** Not explicitly given, but the method reduces the number of fragments processed, leading to lower rendering times [S2].
*   **Memory Complexity:** Not explicitly given.
*   **Measured Performance:**
    *   **Berlin Scene (Desktop):** Fragments reduced from 6.3M to 5.9M; rendering time from 2.7ms to 2.6ms [S2].
    *   **Berlin Scene (VR):** Fragments reduced from 12.8M to 9.8M; rendering time from 5.6ms to 5.2ms [S2].
    *   **Visual Quality:** Image difference metrics (PSNR, SSIM, FLIP, LPIPS) show renderings are virtually indistinguishable from the ground truth [S2].
*   **Scaling:** The impact of approximate culling increases with framebuffer size and is especially prominent in VR rendering, where the pixel count is significantly higher [S2].
*   **Strengths:** Reduces fragment count and rendering time, especially in VR and high-resolution displays. Visually imperceptible artifacts [S2].
*   **Limitations:** Minimal performance gain at default desktop resolutions (1920x1080) due to overhead offsetting benefits [S2]. Potential for visual artifacts if the approximation is too aggressive or the scene is critical [S2].
*   **When to Use:** VR rendering, high-resolution displays, scenes where visual perfection is less critical than frame rate [S2].
*   **When Not to Use:** Low-resolution displays, scenes requiring precise geometric accuracy, or when the overhead of the culling test outweighs the benefits [S2].

## Background
*   **Origin:** The method is attributed to Ibrahim et al. [IRR*22] [S1, S2].
*   **History:** It was proposed as an efficient particle rendering optimization using task and mesh shader pipelines [S1].
*   **Usage:** Used in the "Splatshop" system for editing large Gaussian Splat models [S2].

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
[S2] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
