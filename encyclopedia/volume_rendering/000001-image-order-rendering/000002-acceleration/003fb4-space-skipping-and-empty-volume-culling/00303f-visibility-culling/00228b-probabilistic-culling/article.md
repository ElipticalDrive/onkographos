# Probabilistic Culling

Also called: Approximate Intersection Culling, Approximate Tile-Wise Culling, Ibrahim et al, Probabilistic Culling for Particle Rendering

## Overview
Probabilistic Culling is a rendering optimization that intentionally allows false-negatives in intersection tests to reduce duplicated fragments and rendering time, particularly in high-resolution or virtual reality contexts [S2]. It sets itself apart from conservative culling methods (like [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md)) by trading strict geometric accuracy for performance, relying on the fact that the resulting visual artifacts are imperceptible to the human eye [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of view-dependent primitives (e.g., Gaussian splats or particles) and a target framebuffer resolution.
*   **Output:** A rendered image with reduced fragment count and improved performance, potentially containing minor visual differences compared to a conservative rendering.

**Data Structures:**
The method operates on tiles or blocks of the framebuffer. For each tile, the algorithm determines visibility based on approximate intersection tests.
*   **Tile Bounding Box:** An approximate bounding volume for the primitives covering a tile.
*   **Intersection Test Result:** A boolean indicating if a primitive *might* intersect the tile. In probabilistic culling, this test is biased to return `true` more often than necessary, or specifically designed to allow false negatives (missing a hit) to skip rasterization. [G] *Note: The specific geometric approximation (e.g., simplified bounding box, lower-resolution depth buffer) is not detailed in the provided sources, but the effect is described as "approximate tile-wise culling".*

**Steps:**
1.  **Tile Partitioning:** Divide the framebuffer into tiles (e.g., 16x16 or 32x32 pixels).
2.  **Approximate Intersection Test:** For each primitive, perform a fast, approximate intersection test against the tile's bounding volume.
    *   *Key Characteristic:* This test is designed to be fast and may produce false negatives (i.e., it may fail to detect a valid intersection). [S2]
3.  **Culling Decision:** If the test returns `false` (no intersection), the primitive is culled for that tile. If it returns `true`, the primitive is processed (rasterized/splatted).
4.  **Rendering:** Render the surviving fragments.

**Formulas:**
The visual impact is measured using image difference metrics. Let $I_{exact}$ be the image rendered with conservative culling and $I_{approx}$ be the image rendered with probabilistic culling.
*   **PSNR (Peak Signal-to-Noise Ratio):**
    $$ \text{PSNR} = 10 \cdot \log_{10} \left( \frac{MAX_I^2}{\text{MSE}} \right) $$
    Where $MAX_I$ is the maximum possible pixel value (e.g., 255 for 8-bit) and MSE is the Mean Squared Error between $I_{exact}$ and $I_{approx}$.
*   **SSIM (Structural Similarity Index):** Measures perceived structural change.
*   **FLIP (Frequency-domain Image Difference):** Measures perceptual difference.
*   **LPIPS (Learned Perceptual Image Patch Similarity):** Measures perceptual difference using deep features.

**Parameters:**
*   **Tile Size:** Not specified in sources, but implied by "tile-wise".
*   **Approximation Level:** The degree of approximation in the intersection test is not explicitly parameterized in the sources, but it is the core variable enabling the trade-off.

**Edge Cases and Pitfalls:**
*   **Visual Artifacts:** While metrics suggest artifacts are imperceptible, extreme cases or specific scene content might reveal missing fragments. [S2]
*   **Performance Trade-off:** At standard resolutions (e.g., 1920x1080), the reduction in sorting/rasterization workload may be offset by the cost of the culling test itself or slight increases in fragment creation duration. [S2]
*   **VR Sensitivity:** The benefit is more prominent in VR due to the higher pixel count (e.g., 4.36x desktop pixels), where the reduction in fragment count is significant. [S2]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def probabilistic_culling_render(primitives, framebuffer, tile_size):
    # Initialize framebuffer
    fb = create_empty_framebuffer(framebuffer.width, framebuffer.height)
    
    # Partition framebuffer into tiles
    tiles = partition_into_tiles(framebuffer.width, framebuffer.height, tile_size)
    
    for tile in tiles:
        # Get tile bounding box
        tile_bbox = tile.bounding_box
        
        for primitive in primitives:
            # Perform approximate intersection test
            # This test may return False even if the primitive intersects the tile
            # (i.e., it allows false negatives)
            intersects = approximate_intersection_test(primitive, tile_bbox)
            
            if intersects:
                # Rasterize the primitive for this tile
                rasterize_primitive(primitive, tile, fb)
                
    return fb

def approximate_intersection_test(primitive, tile_bbox):
    # Implementation details of the approximation are not provided in sources.
    # The key is that this function is fast and biased to allow false negatives.
    # [G] A common implementation might use a simplified bounding box or 
    # a lower-resolution depth buffer check.
    return conservative_intersection_test(primitive, tile_bbox) # Placeholder for actual approximation logic
```

## Subtopics

### [Parameter Range Culling](5071-parameter-range-culling.md)
Parameter Range Culling is a runtime optimization that isolates particle subsets by culling particles whose simulation property values do not overlap a user-defined valid range. It solves the problem of visualizing specific behaviors within large particle datasets by avoiding unnecessary ray/sphere intersection tests for particles outside the range of interest. It sets itself apart from static culling methods by allowing the valid range to be changed interactively at runtime, providing flexibility for data analysis, albeit with some computational overhead compared to pre-cropped data.

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** The culling step adds overhead to the intersection test phase but reduces the rasterization and sorting workload. The net effect is a reduction in total rendering time, especially at high resolutions. [S2]
*   **Memory:** No significant additional memory requirements are noted, as it operates on existing tile structures.

**Measured Performance:**
*   **Desktop (1920x1080):** On the Berlin scene, approximate culling reduces fragments from 6.3M to 5.9M and rendering time from 2.7ms to 2.6ms. The impact is minimal because the reduction in sorting/rasterization is offset by the culling cost. [S2]
*   **VR:** On the Berlin scene, approximate culling reduces fragments from 12.8M to 9.8M and rendering time from 5.6ms to 5.2ms. The impact is more prominent due to the higher pixel count. [S2]
*   **Visual Quality:** Image difference metrics show renderings are virtually indistinguishable. For the Berlin scene: PSNR 41.83 dB, SSIM 0.9996, FLIP 0.0010, LPIPS 0.0021. [S2]

**Strengths and Limitations:**
*   **Strengths:** Significant performance gains in VR and high-resolution rendering; imperceptible visual quality loss. [S2]
*   **Limitations:** Minimal benefit at standard desktop resolutions; potential for minor visual artifacts in edge cases. [S2]

**When to Use:**
*   Use in VR or high-resolution rendering where performance is critical and minor visual artifacts are acceptable. [S2]
*   Do not use when strict geometric accuracy is required or at low resolutions where the overhead outweighs the benefits. [S2]

## Background
Probabilistic Culling, also known as Approximate Tile-Wise Culling, was proposed by Ibrahim et al. [IRR*22] for efficient particle rendering using task and mesh shader pipelines [S1]. It is designed to justify intersection tests with false-negatives to reduce duplicated fragments [S2]. The method is particularly relevant for real-time rendering of large particle datasets or Gaussian splats in VR. [S1, S2]

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
[S2] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
