# Appearance Matching

Also called: Block-NeRF Appearance Consistency, NeRF Analogy Training

## Overview
Appearance matching is a post-training optimization procedure used in multi-block neural radiance field frameworks to ensure visual consistency across block boundaries. It solves the problem of appearance discontinuities that arise when independent Block-NeRFs are trained with randomly initialized appearance latent codes, which yield different colors for the same semantic code in adjacent blocks [S1]. It sets itself apart from standard training by freezing network weights and optimizing only latent codes or specific appearance parameters, whereas siblings like [HybridNeRF Model Architecture](4842-hybridnerf-model-architecture.md) or [VoxNeRF](8865-voxnerf.md) integrate appearance handling into the primary forward pass without separate consistency post-processing.

## How it works
The method operates in two distinct contexts: global consistency for Block-NeRFs [S1] and example-based transfer for NeRF Analogies [S2].

### Block-NeRF Appearance Matching
**Inputs:**
*   Two adjacent Block-NeRFs: a *Target* Block-NeRF (with a fixed, desired appearance) and an *Adjacent* Block-NeRF (with inconsistent appearance).
*   A 3D matching location $p$ in world coordinates.
*   The appearance latent code $c_{target}$ of the Target Block-NeRF.
*   The appearance latent code $c_{adj}$ of the Adjacent Block-NeRF.

**Data Structures:**
*   **Block-NeRF Network:** A neural network $f_\theta$ parameterized by $\theta$. Inputs: position $x$, viewing direction $v$, and appearance code $c$. Outputs: radiance $r$ and density $\sigma$.
*   **Visibility Predictor:** A function $V(x)$ estimating the probability that a point $x$ is visible and within the bounds of a specific Block-NeRF.

**Steps:**
1.  **Select Matching Location:** Identify a 3D point $p$ located between the Target and Adjacent blocks. The selection criterion is that the visibility prediction $V_{target}(p)$ and $V_{adj}(p)$ must both be high for their respective blocks [S1].
2.  **Freeze Weights:** Freeze the network weights $\theta_{target}$ and $\theta_{adj}$ of both Block-NeRFs. Only the appearance latent code $c_{adj}$ is treated as an optimization variable [S1].
3.  **Render Target Appearance:** Compute the target radiance $r_{target}$ using the fixed code:
    $$ r_{target} = f_{\theta_{target}}(p, v, c_{target}) $$
    where $v$ is the viewing direction at $p$.
4.  **Render Adjacent Appearance:** Compute the current radiance of the adjacent block:
    $$ r_{adj} = f_{\theta_{adj}}(p, v, c_{adj}) $$
5.  **Optimize:** Minimize the difference between $r_{adj}$ and $r_{target}$ by updating $c_{adj}$. The loss function is typically the L1 or L2 norm of the radiance difference:
    $$ \mathcal{L} = \| r_{adj} - r_{target} \|_1 $$
    Gradient descent updates $c_{adj}$ until the appearance in the Adjacent block matches the Target block at location $p$ [S1].

### NeRF Analogy Training
**Inputs:**
*   Target 3D positions $P_{target}$.
*   Target surface normals $N_{target}$.
*   Target viewing directions $V_{target}$.
*   Source appearance mapping function $\phi$.
*   Source appearances $A_{source}$ and source viewing directions $V_{source}$ derived from semantic correspondences.

**Data Structures:**
*   **Target NeRF Network:** A network $f_\theta$ taking inputs $(x, n, v)$, where $n$ is the surface normal. Outputs radiance $r$.
*   **Mapping Function $\phi$:** Maps source appearance and viewing direction to the target domain.

**Steps:**
1.  **Query Target Network:** Input the target position $p_i$, normal $n_i$, and viewing direction $v_i$ into the target NeRF network to get predicted radiance $r_{pred}$:
    $$ r_{pred} = f_\theta(p_i, n_i, v_i) $$
    Note: The density is assumed to be a Dirac delta distribution (surface), so volume rendering is skipped; the network outputs radiance directly at the surface point [S2].
2.  **Compute Source Appearance:** Apply the mapping function to the source appearance and viewing direction to get the target appearance $r_{source}$:
    $$ r_{source} = \phi(A_{source}, V_{source}) $$
3.  **Calculate Loss:** Compute the L1 loss between the predicted radiance and the mapped source appearance, averaged over all target indices $i$:
    $$ \mathcal{L} = \mathbb{E}_{i} \left[ \| f_\theta(p_i, n_i, v_i) - \phi(A_{source}, V_{source}) \|_1 \right] $$
4.  **Optimize:** Update network parameters $\theta$ to minimize $\mathcal{L}$ [S2].

**Parameters:**
*   **Surface Normals:** Added to the network input for NeRF Analogy to provide a high-frequency signal for recovering high-frequency color changes [S2].

**Edge Cases and Pitfalls:**
*   **Random Initialization:** Without appearance matching, randomly initialized appearance codes lead to different appearances in different blocks, causing visible inconsistencies during compositing [S1].
*   **Visibility:** The matching location must have high visibility in both blocks; otherwise, the optimization may fail or produce artifacts due to occlusion or out-of-bounds queries [S1].
*   **Dirac Delta Assumption:** NeRF Analogy assumes the target density is a Dirac delta distribution. If the target is a volumetric field rather than a surface, the direct radiance query may not be valid without volume rendering integration [S2].

## Subtopics

### NeRF Analogy Training
A method to train a target NeRF's appearance parameters such that the target's radiance matches the source's appearance under the source viewing direction, using semantic correspondences. It differs from Block-NeRF matching by using semantic mappings and surface normals rather than latent code optimization between adjacent blocks. It is chosen when transferring visual attributes (e.g., season, style) from a source image to a target NeRF. It assumes the target density is a Dirac delta distribution (surface), so no volume rendering is required [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the number of optimization steps and the number of matching points. For Block-NeRF, it is $O(N_{steps} \cdot N_{points} \cdot D_{network})$, where $D_{network}$ is the cost of a forward pass. Since weights are frozen, gradients do not flow through the entire network depth for the base features, only through the appearance embedding layer or final output layer depending on implementation [S1].
*   **Memory Complexity:** Low. Only the appearance latent codes or network parameters need to be stored in the optimizer's state. No additional large data structures are required [S1, S2].
*   **Strengths:** Ensures seamless compositing of large scenes split into blocks; enables style transfer without retraining the entire geometry [S1, S2].
*   **Limitations:** Requires high visibility at matching locations; NeRF Analogy requires accurate semantic correspondences and surface normals [S1, S2].
*   **When to use:** When compositing multiple Block-NeRFs or when transferring appearance attributes from a source image to a target NeRF [S1, S2].
*   **When not to use:** When real-time rendering is required without pre-computation, or when the scene is a single monolithic Block-NeRF [S1].

## Background
Block-NeRF appearance matching was introduced in "Block-NeRF: Scalable Large Scene Neural View Synthesis" (CVPR 2022) to address scalability issues in large scenes by splitting them into blocks, which inadvertently caused appearance inconsistencies [S1]. NeRF Analogy Training was introduced in "NeRF Analogies: Example-Based Visual Attribute Transfer for NeRFs" (CVPR 2024) to enable example-based visual attribute transfer using semantic correspondences [S2].

## Sources
[S1] Block-NeRF: Scalable Large Scene Neural View Synthesis, https://doi.org/10.1109/cvpr52688.2022.00807, fetched 2026-10-26
[S2] NeRF Analogies: Example-Based Visual Attribute Transfer for NeRFs, https://doi.org/10.1109/cvpr52733.2024.00444, fetched 2026-10-26
