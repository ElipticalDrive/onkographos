# Probabilistic Culling

Also called: Approximate Intersection Culling, Approximate Tile-Wise Culling, Ibrahim et al, Probabilistic Culling for Particle Rendering

## Overview
Probabilistic Culling is a rendering optimization that intentionally allows false-negatives in intersection tests to reduce duplicated fragments and rendering time, particularly in virtual reality (VR) and high-resolution contexts [S2]. It sets itself apart from conservative culling methods (like [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md)) by trading strict geometric accuracy for performance, relying on the fact that the resulting visual artifacts are often imperceptible to the human eye [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of view-dependent primitives (e.g., splats, particles, or tiles) and their bounding volumes.
*   **Output:** A subset of primitives passed to the rasterization stage, with some occluded primitives incorrectly classified as visible (false-negatives) [S2].

**Data Structures:**
The method operates on tile-wise or primitive-level bounding boxes. No specific complex data structure is required beyond the standard bounding volume hierarchy or tile grid used for the initial intersection test [S2].

**Steps:**
1.  **Tile/Primitive Bounding:** Compute tight bounding boxes for each tile or primitive group [S2].
2.  **Approximate Intersection Test:** Perform an intersection test between the view frustum (or camera ray) and the bounding box. Unlike conservative culling, this test is designed to be fast and may return "visible" even if the primitive is technically occluded or outside the view, effectively justifying the intersection test with false-negatives [S2].
3.  **Fragment Generation:** Pass the "visible" primitives to the rasterizer. The false-negatives result in additional fragments being generated and processed [S2].
4.  **Rendering:** Render the fragments. The visual impact of the extra fragments is negligible due to the nature of the primitives (e.g., soft splats) and the human visual system [S2].

**Formulas:**
The method does not introduce new mathematical formulas for compositing but modifies the culling condition. Let $V$ be the visibility flag. Conservative culling ensures $V = \text{true} \implies \text{primitive is visible}$. Probabilistic culling allows $V = \text{true} \implies \text{primitive is likely visible, but may be occluded}$ [S2].

**Parameters:**
*   **Tightness of Bounding Boxes:** Tighter bounding boxes reduce the number of false-negatives but increase culling overhead [S2].
*   **Framebuffer Resolution:** The benefit increases with resolution [S2].

**Edge Cases and Pitfalls:**
*   **Low Resolution:** At default framebuffer sizes (e.g., 1920x1080), the performance gain is minimal because the reduction in sorting/rasterization workload is offset by the slight increase in fragment creation duration [S2].
*   **Visual Quality:** While metrics suggest imperceptibility, extreme cases or specific scene contents might reveal artifacts [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def probabilistic_cull(primitives, camera):
    """
    Cull primitives using approximate intersection tests.
    """
    visible_primitives = []
    for primitive in primitives:
        # Compute tight bounding box for the primitive
        bbox = compute_tight_bbox(primitive)
        
        # Perform approximate intersection test
        # This test may return True even if the primitive is occluded
        is_visible = approximate_intersection_test(bbox, camera)
        
        if is_visible:
            visible_primitives.append(primitive)
            
    return visible_primitives
```

## Subtopics

### [Parameter Range Culling](5071-parameter-range-culling.md)
A runtime optimization that isolates particle subsets by culling particles whose simulation property values do not overlap a user-defined valid range. It sets itself apart by allowing interactive range changes, unlike static culling.

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** The culling step is $O(N)$ where $N$ is the number of primitives/tiles. The benefit comes from reducing the number of fragments processed in the rasterization stage [S2].
*   **Memory:** Minimal additional memory is required beyond the bounding boxes [S2].

**Measured Performance:**
*   **Berlin Scene (Desktop):** Fragments reduced from 6.3M to 5.9M; rendering time from 2.7ms to 2.6ms [S2].
*   **Berlin Scene (VR):** Fragments reduced from 12.8M to 9.8M; rendering time from 5.6ms to 5.2ms [S2].
*   **Visual Quality:** Image difference metrics show renderings are virtually indistinguishable (Berlin PSNR 41.83 dB, SSIM 0.9996, FLIP 0.0010, LPIPS 0.0021) [S2].

**Scaling:**
*   The impact increases with framebuffer size and is especially prominent in VR rendering, where the pixel count is significantly higher (e.g., 4.36 times more pixels in VR vs. desktop) [S2].

**Strengths and Limitations:**
*   **Strengths:** Significant performance gains in VR and high-resolution rendering; imperceptible visual quality loss [S2].
*   **Limitations:** Little impact at low resolutions; potential for slight increase in fragment creation duration [S2].

## Background
Probabilistic Culling, also known as Approximate Tile-Wise Culling, was proposed by Ibrahim et al. [IRR*22] for efficient particle rendering using task and mesh shader pipelines [S1, S2]. It is used in contexts where real-time performance is critical, such as VR, and where the visual cost of false-negatives is acceptable [S2].

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
[S2] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
