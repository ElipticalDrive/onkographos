# Probabilistic Culling

Also called: Approximate Intersection Culling, Approximate Tile-Wise Culling, Ibrahim et al, Probabilistic Culling for Particle Rendering

## Overview
Probabilistic culling is a visibility optimization technique that intentionally allows false-negatives in intersection tests to reduce duplicated fragments and rendering time, particularly in high-resolution or virtual reality (VR) contexts [S2]. It distinguishes itself from conservative culling methods by trading strict geometric accuracy for significant performance gains, relying on the fact that the resulting visual artifacts are often imperceptible to the human eye [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of primitives (e.g., splats, particles) with bounding volumes and a tile-based rasterization grid.
*   **Output:** A subset of primitives deemed visible, allowing the renderer to skip intersection tests for others.

**Data Structures:**
Not given in the sources. The method operates on tile-based structures but specific memory layouts are not detailed.

**Steps:**
1.  **Tile Binning:** Divide the framebuffer into tiles.
2.  **Approximate Intersection Test:** For each tile, perform an intersection test between the tile's bounding box and the primitive's bounding volume. Unlike conservative culling, this test is "approximate," meaning it may return a false negative (claiming a primitive is outside the tile when it is actually inside) [S2].
3.  **Culling Decision:** If the test returns false (outside), the primitive is culled for that tile. If true (inside), the primitive is processed [S2].
4.  **Rendering:** Render only the primitives that passed the test.

**Formulas:**
Not given in the sources.

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Visual Artifacts:** False negatives result in missing fragments. While metrics show these are often imperceptible, they can be visible in specific scenes or at lower resolutions [S2].
*   **Resolution Dependence:** The benefit of probabilistic culling increases with framebuffer size. At default desktop resolutions (e.g., 1920x1080), the reduction in sorting/rasterization workload may be offset by increased fragment creation duration, yielding little net gain [S2]. In VR, where pixel counts are significantly higher, the performance impact is prominent [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for approximate tile-wise culling
# Note: Specific intersection logic is not provided in sources [S2]

def probabilistic_cull(primitives, tiles):
    visible_primitives = []
    
    for tile in tiles:
        for primitive in primitives:
            # Perform approximate intersection test
            # This test may return False even if the primitive overlaps the tile
            is_intersecting = approximate_intersection_test(tile.bbox, primitive.bbox)
            
            if is_intersecting:
                visible_primitives.append((tile, primitive))
                
    return visible_primitives

def approximate_intersection_test(tile_bbox, primitive_bbox):
    # Implementation detail not provided in sources [S2]
    # Conservative culling would use strict overlap checks.
    # Probabilistic culling uses an approximation that allows false negatives.
    return True # Placeholder for actual logic
```

## Subtopics

### [Parameter Range Culling](5071-parameter-range-culling.md)
Parameter Range Culling (PRC) is a runtime visibility optimization that isolates particle subsets by culling particles whose scalar parameter values (e.g., density, velocity) do not overlap a user-defined valid range. It sets itself apart from static culling by performing these checks dynamically during traversal, allowing for interactive exploration of simulation data without pre-processing.

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
*   **Visual Quality:** Image difference metrics between correct and approximate intersection tests show renderings are virtually indistinguishable. For the Berlin scene: PSNR 41.83 dB, SSIM 0.9996, FLIP 0.0010, LPIPS 0.0021 [S2].
*   **Fragment Reduction:** On the Berlin scene (desktop), approximate culling reduces fragments from 6.3M to 5.9M. In VR, it reduces fragments from 12.8M to 9.8M [S2].
*   **Rendering Time:** On the Berlin scene (desktop), rendering time drops from 2.7ms to 2.6ms. In VR, it drops from 5.6ms to 5.2ms [S2].
*   **Scaling:** The impact of approximate culling increases with framebuffer size. It shows little impact at 1920x1080 but is prominent in VR, where 4.36 times as many pixels are rendered as in desktop mode [S2].

**Strengths and Limitations:**
*   **Strengths:** Significant reduction in duplicated fragments and rendering time in high-resolution/VR contexts; imperceptible visual quality loss in most cases [S2].
*   **Limitations:** Minimal benefit at standard desktop resolutions; potential for visible missing fragments in specific scenarios [S2].

**When to Use:**
Use in VR rendering or high-resolution desktop rendering where performance is critical and minor visual artifacts are acceptable [S2].

**When Not to Use:**
Avoid in applications requiring strict geometric accuracy or at low resolutions where the overhead of the culling logic may not be offset by performance gains [S2].

## Background
Probabilistic culling, also referred to as approximate tile-wise culling, was proposed by Ibrahim et al. [IRR*22] for efficient particle rendering using task and mesh shader pipelines [S1]. It is utilized in systems like Splatshop to optimize the rendering of large Gaussian Splat models [S2].

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
[S2] Splatshop: Efficiently Editing Large Gaussian Splat Models, https://doi.org/10.1111/cgf.70214, fetched 2026-10-31
