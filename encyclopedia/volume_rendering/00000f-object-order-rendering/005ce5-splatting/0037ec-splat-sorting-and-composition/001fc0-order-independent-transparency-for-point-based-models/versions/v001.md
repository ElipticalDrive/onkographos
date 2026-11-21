# Order Independent Transparency for Point-Based Models

Also called: Depth-peeling for splats, Multi-pass splatting transparency

## Overview
This method renders transparent point-based models (splats) on GPUs in an order-independent manner by extending depth-peeling techniques to deferred-shaded multi-pass splatting [S1]. It solves the problem of expensive $O(N \log N)$ depth sorting required for correct alpha blending of translucent splats [S1]. Unlike standard depth-sorted rendering, it iteratively reconstructs a fixed number of transparent layers from closest to farthest, allowing for efficient GPU execution at the cost of potential color loss if the layer limit is exceeded [S1].

## How it works
The method operates in two main phases: opaque rendering and iterative transparent layer reconstruction.

**Inputs and Outputs:**
*   **Inputs:** Point cloud data (positions, colors, opacities, normals), camera parameters, and a bound $n$ for the number of transparent layers to reconstruct.
*   **Outputs:** A final composited image.

**Data Structures:**
*   **Depth Buffers:** A read-only depth texture storing the depth of the previously reconstructed layer (or opaque objects for the first pass).
*   **Attribute Buffers:** Textures storing accumulated attributes (color, normal) for the current layer.
*   **Layer Textures:** A set of $n$ textures storing the final shaded result for each reconstructed layer.

**Parameters:**
*   $n$: The maximum number of transparent layers to reconstruct. Typically bounded to 3 or 4 for most scenes to balance efficiency and memory usage [S1].

**Algorithm Steps:**

1.  **Opaque Rendering:**
    Render opaque objects using the standard three-pass splatting pipeline: visibility, attribute accumulation, and deferred shading [S1].

2.  **Iterative Transparent Layer Reconstruction (Loop $i$ from 1 to $n$):**
    For each layer $i$, perform the following sub-steps:

    *   **2.1. Depth Buffer Setup:**
        Bind the depth buffer from the previous iteration (or the opaque depth buffer if $i=1$) as a read-only depth texture [S1].

    *   **2.2. Buffer Clearing:**
        Clear the destination attribute and depth buffers for the current layer [S1].

    *   **2.3. Visibility Splatting (Pass 1):**
        Perform splatting with an additional depth comparison. Only fragments that are behind the depth of the previous layer (stored in the read-only texture) pass the test [S1].
        *   *Exception:* If $i = n$ (the last layer), skip this pass to reduce rendering time [S1].

    *   **2.4. Attribute Accumulation (Pass 2):**
        Perform splatting with two additional depth comparisons in the fragment shader:
        1.  Compare against the depth of the previous layer.
        2.  Compare against the depth of opaque objects.
        Only visible splats behind both depths pass these tests [S1].
        Accumulate the attributes (e.g., color, normal) of these visible splats into the current layer's attribute buffer.

    *   **2.5. Normalization and Deferred Shading (Pass 3):**
        Normalize the accumulated attributes and perform deferred shading. Store the result in a texture representing layer $i$ [S1].
        *   *Exception:* If $i = n$, switch to per-splat lighting instead of deferred shading. This is because blending surface attributes from different surfaces in the last layer makes no sense, and per-pixel lighting quality is equivalent due to the small size of distant splats [S1].

3.  **Final Compositing:**
    Combine all reconstructed layers with the final image by drawing them with alpha-blending in the inverse order of their construction (from farthest layer $n$ to closest layer 1) [S1].

4.  **Last Layer Modulation (Heuristic):**
    To mitigate artifacts when the viewpoint contains more than $n$ visible transparent layers, transparency coefficients are used to modulate the splatting weights in the last layer $n$. This ensures that more opaque surfaces have more importance [S1].

**Pitfalls and Edge Cases:**
*   **Layer Limit Exceeded:** If the viewpoint contains more than $n$ visible transparent layers, the color information of unreconstructed layers is lost [S1].
*   **Artifacts in Last Layer:** The approximation of using per-splat lighting for the last layer may introduce minor artifacts, though these are seldom perceptible because contributions of distant layers are attenuated by closer layers [S1].

## Subtopics

### [Order Independent Transparency Approaches](3611-order-independent-transparency-approaches.md)
This family encompasses methods for rendering transparent point-based models without relying on strict depth sorting. It addresses the non-commutative nature of translucent compositing by approximating the correct order of operations or using alternative accumulation strategies.

## Operating characteristics
*   **Time Complexity:** The method requires $n$ iterations for transparent layers, where each iteration involves multiple splatting passes. The cost is proportional to $n \times (\text{number of splats})$.
*   **Memory Complexity:** Requires storage for $n$ layer textures plus intermediate buffers for depth and attributes.
*   **Measured Performance:** On a GeForce 7800 GPU, rendering 27 transparent point-sampled spheres with 6 layers achieved 21 fps. With 3 layers, performance improved to 45 fps without the heuristic and 49 fps with the heuristic [S1].
*   **Strengths:** Avoids expensive depth sorting; leverages GPU parallelism; efficient for scenes with a limited number of transparent layers.
*   **Limitations:** Fixed layer limit $n$ can lead to visual artifacts if exceeded; deferred shading in the last layer is an approximation.

## Background
This method was introduced by [S1] as an extension of depth-peeling techniques to deferred-shaded multi-pass splatting for point-based rendering. It addresses the computational bottleneck of sorting large point clouds for transparency rendering on GPUs.

## Sources
[S1] Splat/Mesh Blending, Perspective Rasterization and Transparency for Point-Based Rendering, https://doi.org/10.2312/spbg/spbg06/049-057, fetched 2026-10-26
