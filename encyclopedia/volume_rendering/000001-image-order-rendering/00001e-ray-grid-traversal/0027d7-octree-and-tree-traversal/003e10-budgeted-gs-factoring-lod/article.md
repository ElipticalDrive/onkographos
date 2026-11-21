# Budgeted-GS Factoring LOD

Also called: Budget-Centered Training (BCT) context, Factored LOD

## Overview
Budgeted-GS Factoring LOD is a hierarchical level-of-detail (LOD) method for real-time large-scale Gaussian Splatting that reduces rendering cost by aggregating primitives into equivalent planks based on a pixel-footprint budget [S1, unofficial]. It solves the problem of rendering millions of Gaussians in real-time by dynamically selecting between raw primitives, fine equivalent planks, or coarse aggregate planks depending on their distance and scale relative to the camera [S1, unofficial]. Unlike standard compression methods that operate in an over-budgeted regime with flat error curves, this method achieves a steeper error reduction (slope approaching -0.5) by using a "floor law" to select the optimal budget, allowing for significant size reduction (119× smaller than input) while maintaining visual fidelity [S1, unofficial].

## How it works
The method consists of a construction phase that builds a hierarchical octree of equivalent planks and a rendering phase that selects the appropriate LOD tier for each node [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- A set of $N$ trained 3D Gaussian primitives, each defined by mean $\mu_i$, rotation $R_i$, scale $\sigma_i$, opacity $\alpha_i$, and spherical harmonic (SH) color coefficients [S1, unofficial].
- Camera parameters: position $p$ and focal length $f$ (in pixels) [S1, unofficial].
- Budget parameter $c$ (scalar) [S1, unofficial].

**Outputs:**
- An octree structure where each node (leaf or internal) contains a list of equivalent planks, organized by scale bands [S1, unofficial].
- Rendered image using the selected tier for each visible primitive/plank [S1, unofficial].

### Data Structures
The scene is partitioned using a GPU octree [S1, unofficial].
- **Leaf Capacity:** Each leaf node holds at most 4096 primitives [S1, unofficial].
- **Primitive Representation (Plank):** Each primitive is converted to a "plank" defined by $(\mu, R, h=3\sigma, \alpha)$ [S1, unofficial].
- **Equivalent Plank:** An aggregate representation defined by $(c^*, R^*, h^*, w^*)$ and SH coefficients [S1, unofficial].
- **Scale Bands:** Within each leaf, planks are grouped into bands $\ell$ based on their maximum height $s = \max(h)$ [S1, unofficial].
- **Bucket:** Within a (leaf, band) group, planks are split into consecutive buckets of at most 256 planks [S1, unofficial].

### Steps

1.  **Primitive Conversion:**
    Convert each Gaussian primitive $i$ with scale $\sigma_i$ into a plank with height $h_i = 3\sigma_i$ [S1, unofficial]. The primitive is now represented as $(\mu_i, R_i, h_i, \alpha_i)$ [S1, unofficial].

2.  **Octree Construction:**
    Partition the scene bounding box into a GPU octree with a leaf capacity of 4096 primitives [S1, unofficial].

3.  **Scale Banding:**
    For each leaf node, determine the base scale $s_{base}$ as the 5% quantile of the height $s$ across the entire scene [S1, unofficial]. For each plank in the leaf with maximum height $s$, compute the scale band index:
    $$
    \ell = \lfloor \log_2 (s/s_{base}) \rfloor
    $$
    Group planks by their (leaf, band) pair [S1, unofficial].

4.  **Factoring (Moment Matching):**
    For each (leaf, band) group:
    a.  Order the planks by Morton code to ensure spatial locality [S1, unofficial].
    b.  Split the ordered planks into consecutive buckets, each containing at most 256 planks [S1, unofficial].
    c.  For each bucket, compute the equivalent plank parameters by conserving weighted volume $W$, centroid $c^*$, and inertia tensor $T^*$ [S1, unofficial].

    **Weighted Volume:**
    $$
    W = \sum_i \alpha_i |V_i|
    $$
    where $|V_i|$ is the volume of the $i$-th plank [S1, unofficial].

    **Centroid:**
    $$
    c^* = \frac{\sum_i \alpha_i |V_i| \mu_i}{W}
    $$

    **Inertia Tensor:**
    $$
    T^* = \sum_i \alpha_i |V_i| \left( (\mu_i - c^*)(\mu_i - c^*)^\top + \frac{1}{3} R_i \text{diag}(h_i)^2 R_i^\top \right)
    $$
    where $R_i$ is the rotation and $h_i$ is the height of the $i$-th plank [S1, unofficial].

    **Equivalent Plank Derivation:**
    Perform eigendecomposition on $T^*$ to derive the equivalent plank's rotation $R^*$ and height $h^*$ [S1, unofficial]. The weight $w^*$ is calculated as:
    $$
    w^* = W/|V^*| \in [0,1]
    $$
    where $|V^*|$ is the volume of the equivalent plank [S1, unofficial].

    **Color Handling:**
    Colors are handled as opacity-weighted first moments of the SH coefficients, computed separately for DC and higher bands [S1, unofficial].

5.  **Bottom-Up Aggregation:**
    Apply the same factoring treatment to internal nodes of the octree [S1, unofficial]. Every (node, band) pair owns one equivalent plank [S1, unofficial]. Raw planks are retained in Morton order within leaf nodes for the finest tier [S1, unofficial].

6.  **Rendering Tier Selection:**
    At render time, for a camera at position $p$ with focal length $f$, calculate the threshold $t$ for each (leaf, band) group:
    $$
    t = c \cdot \frac{d}{f}
    $$
    where $d$ is the distance from the camera to the centroid of the group [S1, unofficial].

    Determine the tier based on the mean scale $\bar{s}$ and equivalent-plank scale $\bar{h}$:
    -   **Raw Tier:** If $\bar{s} \ge t$, members are individually resolvable [S1, unofficial]. Render the original Gaussians [S1, unofficial].
    -   **Fine Tier:** If $\bar{s} < t \le \bar{h}$, the cluster is resolvable but members are not [S1, unofficial]. Render the fine equivalent planks [S1, unofficial].
    -   **Coarse Tier:** If $\bar{h} < t$, the content is sub-pixel [S1, unofficial]. Render the single aggregate plank of the (leaf, band) [S1, unofficial].

### Parameters
-   **Bucket Size:** 256 planks per bucket [S1, unofficial].
-   **Budget Parameter $c$:** Controls the aggressiveness of LOD [S1, unofficial].
    -   $c=1$: Exactly resolvable (reference tier) [S1, unofficial].
    -   $c>1$: Collapses sub-pixel content (scale truncation) [S1, unofficial].
    -   $c<1$: Super-resolving regime (keeps members below pixel scale) [S1, unofficial].
    -   Typical value for distant aerial cameras: $c \approx 0.3$ [S1, unofficial].

### Edge Cases and Pitfalls
-   **Precision:** The inertia tensor calculation involves summing weighted outer products, which may require double precision for large scenes to avoid numerical instability [G].
-   **Boundary Conditions:** The Morton code ordering ensures spatial locality, but extreme anisotropy in primitive scales may lead to inefficient bucketing if the scale banding is not fine-grained enough [G].
-   **Degenerate Input:** If a bucket contains fewer than 256 planks, the moment matching still applies to the available planks [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** Construction takes seconds on multi-million-primitive scenes on a single GPU [S1, unofficial]. Rendering is real-time, with tier selection being $O(1)$ per node [S1, unofficial].
-   **Memory Complexity:** The aggregate representation is 119× smaller than the input [S1, unofficial].
-   **Performance:** Achieves a steeper error curve (slope approaching -0.5) compared to published compression methods (slopes [-0.28, +0.09]) [S1, unofficial].
-   **Strengths:** Real-time performance on large-scale scenes, certified compression headroom, and significant size reduction [S1, unofficial].
-   **Limitations:** Requires a construction phase; effectiveness depends on the accuracy of the "floor law" estimation [S1, unofficial].

## Background
Budgeted-GS Factoring LOD was introduced in the paper "Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD" [S1, unofficial]. It builds on the principles of octree traversal and Gaussian splatting, extending them with a hierarchical factoring mechanism based on moment matching [S1, unofficial]. The method addresses the limitations of existing compression techniques by operating in a budgeted regime that aligns with the perceptual floor of the scene [S1, unofficial].

## Sources
[S1] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-29 (unofficial)
