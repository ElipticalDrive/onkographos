# Sky Dome Initialization

Also called: Frozen Shell of Gaussians, Sky Dome

## Overview
Sky Dome Initialization is a technique for handling unbounded scenes in 3D Gaussian Splatting by modeling the distant background (sky) as a separate, frozen shell of Gaussians [S1, S2, unofficial]. It solves the problem of "cloud-like" artifacts that occur when foreground Gaussians attempt to model distant sky regions, which degrades novel view synthesis quality [S2, unofficial]. It sets itself apart from standard background models by explicitly disentangling the sky from the foreground geometry, often using a hemisphere to avoid occluding the ground plane [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of input images, camera poses, and optionally sky segmentation masks [S1, S2, unofficial].
*   **Outputs:** A rendered image where the sky is composited behind the foreground, and a set of frozen Gaussian parameters for the sky dome [S1, S2, unofficial].

**Data Structures:**
The Sky Dome consists of a fixed set of Gaussians.
*   **Sphere Variant:** Gaussians are distributed on a sphere surface [S1, S2, unofficial].
*   **Hemisphere Variant:** Gaussians are distributed on a hemisphere surface, leaving the bottom half empty (black background) [S1, unofficial].
*   **Mask Data:** If using semantic priors, sky masks are stored in a directory structure `<dataset-root>/sky/<FILENAME>.png`, where `FILENAME` matches the corresponding dataset image [S1, unofficial].

**Steps:**
1.  **Initialize Sky Gaussians:** Sample points from a sphere or hemisphere to initialize the sky Gaussian positions. A common method is sampling 100,000 points from a Fibonacci sphere [S2, unofficial].
2.  **Freeze Parameters:** The position, rotation, scale, and color of these sky Gaussians are frozen (not optimized) during training [S1, unofficial].
3.  **Render Sky Pass:** Render the sky dome independently to obtain the sky color $C_{sky}(p)$ and opacity $O(p)$ for each pixel [S2, unofficial].
    *   Accumulated transmittance is calculated as:
        $$ T_i = \prod_{k=1}^{i-1} (1 - \alpha_k) $$
        where $T_i$ is the transmittance at step $i$ [S2, unofficial].
    *   Foreground opacity is the sum of sample weights:
        $$ O(p) = \sum_i T_i \alpha_i $$
        [S2, unofficial].
4.  **Composite:** Combine the foreground and sky colors. The final pixel color $C(p)$ is:
    $$ C(p) = \sum_i T_i \alpha_i c_i + (1 - O(p)) \cdot C_{sky}(p) $$
    where $c_i$ is the color of the $i$-th foreground Gaussian [S2, unofficial].
5.  **Regularization (Optional):** If sky masks are available, apply a loss $L_{sky}$ to penalize foreground opacity in sky regions, encouraging $O(p)$ to be close to zero in those areas [S2, unofficial].
    $$ L_{sky} = \lambda_{sky} \sum_{p \in \text{sky}} O(p) $$
    where $\lambda_{sky}$ is a weight (e.g., 10) [S2, unofficial].

**Parameters:**
*   **Number of Sky Gaussians:** ~32,000 for the dome, which is negligible compared to millions of foreground Gaussians [S1, unofficial].
*   **$\lambda_{sky}$:** Set to 10 to remove cloud artifacts in the initial training steps (e.g., first 7,000 steps) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Sphere vs. Hemisphere:** Using a full sphere can pull geometry onto itself, making the ground less opaque. A hemisphere leaves the bottom half black, preventing interference with foreground geometry [S1, unofficial].
*   **Sparse Free Space:** Simply blending sky and foreground colors may not clear artifacts if sparse points exist in free space initially [S2, unofficial].
*   **Convergence Trade-offs:** Alternative approaches like sky depth loss or BCE semantic regularization may require longer convergence times and can negatively impact foreground modeling quality [S2, unofficial].

## Subtopics

### Gaussian Sky Modeling
A rendering model that incorporates a spherical environment map composed of Gaussians to model the sky separately from the foreground [S2, unofficial]. It addresses cloud-like artifacts in unbounded scenes by initializing 100,000 Gaussians via Fibonacci sphere sampling and using semantic priors to generate sky masks [S2, unofficial]. It is chosen when standard 3DGS produces significant sky artifacts in unbounded scenes [S2, unofficial].

## Operating characteristics
*   **Time and Memory Complexity:** The overhead is negligible (~32K Gaussians) compared to the millions of foreground Gaussians [S1, unofficial]. Rendering adds one additional pass for the sky dome [S1, unofficial].
*   **Performance:** Effectively disentangles sky from foreground, reducing floaters and improving novel view synthesis quality in unbounded scenes [S1, S2, unofficial].
*   **Strengths:** Simple to implement, low overhead, effective at removing cloud artifacts [S1, S2, unofficial].
*   **Limitations:** Requires careful choice between sphere and hemisphere based on scene content (ground visibility) [S1, unofficial]. Semantic regularization may slow convergence [S2, unofficial].
*   **When to Use:** Unbounded scenes with visible sky where standard 3DGS produces artifacts [S1, S2, unofficial].
*   **When Not to Use:** Bounded scenes where the sky is not visible or is fully occluded [S1, unofficial].

## Background
Originated as an improvement in Gaussian Splatting implementations to handle unbounded scenes [S1, S2, unofficial]. The "Sky Dome" concept was popularized in unofficial implementations like GaussianSplatting.jl [S1, unofficial]. The "Gaussian Sky Modeling" variant was introduced in GauStudio, a modular framework for 3D Gaussian Splatting [S2, unofficial]. It is used in 3D reconstruction and novel view synthesis for outdoor or large-scale scenes [S1, S2, unofficial].

## Sources
[S1] Better Gaussian Splatting in Julia, https://pxl-th.github.io/blog/better-gs-julia/, fetched 2026-10-26 (unofficial)
[S2] GauStudio: A Modular Framework for 3D Gaussian Splatting and Beyond, https://doi.org/10.48550/arXiv.2403.19632, fetched 2026-10-30 (unofficial)
