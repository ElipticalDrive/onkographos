# Early Ray Termination via Depth Buffer

Also called: Depth-based Early Termination, Early Z-Test

## Overview
Early Ray Termination via Depth Buffer is an acceleration technique that skips fragment shader execution for rays that have already reached saturation (opacity $\approx 1$) by testing against a depth buffer before processing the volume data [S1, S2, unofficial]. It solves the problem of redundant computation in opaque regions by preventing the GPU from generating fragments for occluded or saturated positions [S2, unofficial]. It sets itself apart from sibling methods like [Empty-space skipping](1002-empty-space-skipping.md) (which skips empty geometry) and [Opacity-based Early Termination](7920-opacity-based-early-termination.md) (which terminates inside the shader) by using a hardware-level depth test to cull work *before* the expensive volume sampling occurs [S1, S2, unofficial].

## How it works
**Inputs:**
*   Volume data (voxels or bricks).
*   Current depth buffer (initialized from previous frame or clear).
*   Opacity threshold $\tau$ (typically $0.95 \dots 1.0$, varying by rendering mode) [S2, unofficial].

**Outputs:**
*   Final color buffer.
*   Updated depth buffer reflecting saturated rays.

**Data Structures:**
*   **Depth Buffer:** Standard 24-bit or 32-bit floating-point buffer storing the maximum depth $z_{max}$ encountered so far for each pixel.
*   **Brick Bounding Box:** A solid mesh representing the spatial extent of the current volume brick, used for the depth test.

**Algorithm Steps:**
The method requires traversing the volume in front-to-back order [S1, S2, unofficial].

1.  **Reconstruct Slice Stack:** Compute the current slice from all projections in front-to-back order [S2, unofficial].
2.  **Composite with Frame Buffer:** Composite the current slice with the existing frame buffer to update color and opacity [S2, unofficial].
3.  **Update Depth Buffer (Saturation Pass):**
    *   Examine every fragment's accumulated opacity $A$.
    *   If $A > \tau$, record the fragment's depth $z$ to update the depth buffer. Specifically, write the maximal depth value for saturated rays [S1, S2, unofficial].
    *   *Note:* This step ensures the depth buffer reflects where rays are already opaque [S2, unofficial].
4.  **Render Next Brick with Early Z-Test:**
    *   Before rendering the texture of the next brick, execute a fragment shader program while drawing a solid bounding box around the brick with back face culling switched on [S1, unofficial].
    *   This shader writes the maximal depth value in the depth buffer for saturated rays [S1, unofficial].
    *   When slicing the brick texture, perform an early Z-test. If the depth test fails (indicating the ray is already saturated/occluded by a previous brick), prevent any fragment operations from being executed [S1, unofficial].
5.  **Repeat:** Proceed to the next slice/brick.

**Formulas:**
The standard front-to-back compositing equations are:
$$ C_{i+1} = (1 - A_i) \cdot \alpha_i \cdot c_i + C_i $$
$$ A_{i+1} = (1 - A_i) \cdot \alpha_i + A_i $$
Where $C, A$ are the current ray color and opacity, $c, \alpha$ are the sample color and opacity from the transfer function, and $i$ is the sample index [S1, unofficial]. A ray is considered saturated when $A_i \approx 1$ [S1, unofficial].

**Parameters:**
*   **Opacity Threshold ($\tau$):** Varies between 0.0 and 1.0 depending on rendering mode (isosurface vs. full volume) [S2, unofficial]. Typical values are high (e.g., 0.99) for full volume to ensure only truly opaque rays are culled.

**Edge Cases and Pitfalls:**
*   **Overhead:** Early ray termination is performed only once per brick, not per octree node or sample. Performing it more frequently (e.g., changing fragment shaders per sample) would annihilate the benefits due to overhead [S1, unofficial].
*   **Low Computational Effort:** If the fragment shader has low computational effort, the overhead of the depth test and culling mechanism may offset the advantages [S2, unofficial]. This method benefits more from spreading methods with longer fragment shaders [S2, unofficial].
*   **Order Dependency:** The technique strictly requires front-to-back traversal to correctly identify saturation [S1, S2, unofficial].

## Subtopics

### [Depth Prepass with Equal-Depth Shading](15884-depth-prepass-with-equal-depth-shading.md)
Optimizes rendering of overlapping opaque primitives (spheres) by using a dedicated prepass and `depth_compare: Equal` to select the closest hit, bypassing GPU hidden-surface removal issues caused by discard-based impostors.

## Operating characteristics
*   **Time Complexity:** Reduces fragment shader invocations for saturated rays. The cost is proportional to the number of saturated pixels plus the overhead of the depth update pass.
*   **Memory Complexity:** Requires storage for the depth buffer and the brick bounding boxes.
*   **Performance:** Achieves good speed-ups for iso-surface rendering where rays are generally terminated early [S2, unofficial]. Effectiveness depends on the ratio of computational effort per fragment; it is most beneficial when fragment shaders are expensive [S2, unofficial].
*   **Strengths:** Eliminates reconstruction of voxels that do not contribute to rendering [S2, unofficial]. Reduces load on rasterization and fragment shader bottlenecks [S1, unofficial].
*   **Limitations:** Only effective if the volume traversal is front-to-back. Overhead can negate benefits if the shading is simple or if the test is performed too frequently (e.g., per sample) [S1, S2, unofficial].

## Background
The technique was introduced in the context of optimizing GPU volume rendering, specifically for brick-based rendering and distributed volume rendering (D2VR) [S1, S2, unofficial]. It relies on the under operator for front-to-back compositing [S1, unofficial]. The method was cited in works by Krueger et al. [KW03] and Ropinski et al. [RGW+03] [S1, unofficial].

## Sources
[S1] Optimizing GPU Volume Rendering, Danny Ruijters, https://dannyruijters.nl/docs/OptimizingGPUVolumeRendering_final.pdf, fetched 2026-10-26 (unofficial)
[S2] GPU-Accelerated D2VR, https://doi.org/10.2312/vg/vg06/023-030, fetched 2026-10-26 (unofficial)
