# Decomposed Spatiotemporal Representation

Also called: NeRFPlayer Decomposition

## Overview
Decomposed Spatiotemporal Representation is a method for modeling dynamic scenes by categorizing volumetric areas into static, deforming, and new content, assigning each category a specialized representation field to optimize for temporal consistency and computational efficiency. It differs from the parent DyNeRF method, which uses a single monolithic MLP conditioned on time, by explicitly separating the scene into distinct temporal behaviors, allowing static regions to remain time-invariant and deforming regions to use canonical space mappings.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D spatial coordinate $p \in \mathbb{R}^3$ and a temporal coordinate $t \in \mathbb{R}$ (normalized time). Camera pose is required for the final view-conditioned network.
*   **Output:** Density $\sigma$ and view-dependent color $c$.

**Data Structures:**
The method employs four primary fields. The exact internal architecture (e.g., layer widths) is not specified in the sources, but their functional signatures are:
1.  **Decomposition Field $f(p, t)$:** Outputs a probability distribution over three categories.
    *   Output: $(P_{static}, P_{deform}, P_{new})$ where $\sum P_* = 1$.
    *   Implementation: Explicitly cached features $V_f$ and a small MLP decoder $D_f$ [S1].
2.  **Stationary Field $s(\cdot)$:** Models static areas.
    *   Input: Spatial coordinate $p$.
    *   Output: Feature vector $v_{static}$.
    *   Note: Assumed to be time-invariant [S1].
3.  **Deformation Field $d(p, t)$:** Models deforming areas.
    *   Input: Spatial coordinate $p$ and time $t$.
    *   Output: Displacement vector $\Delta p$.
    *   Implementation: A small MLP [S1].
4.  **Newness Field $n(p, t)$:** Models emerging content.
    *   Input: Spatial coordinate $p$ and time $t$.
    *   Output: Feature vector $v_{new}$.
    *   Implementation: Explicitly cached features and a small MLP [S1].

**Numbered Steps:**
1.  **Decomposition:** Query the decomposition field $f(p, t)$ to obtain probabilities $P_{static}, P_{deform}, P_{new}$.
2.  **Feature Extraction:**
    *   Query the stationary field $s(p)$ to get $v_{static}$.
    *   Query the deformation field $d(p, t)$ to get displacement $\Delta p$. Compute the deformed point $p' = p + \Delta p$. Query the canonical space (e.g., the stationary field $s$) at $p'$ to get $v_{deform}$. *Note: The sources state $d(p,t) \rightarrow (\Delta p)$ and "the deformed point $p + \Delta p$ is sent as the query point into a predefined canonical space (e.g., the static field s)" [S1]. This implies $v_{deform}$ is derived from the canonical query, not directly from $d$.*
    *   Query the newness field $n(p, t)$ to get $v_{new}$.
3.  **Thinning:** For each category $* \in \{static, deform, new\}$, if $P_* < \tau$ (default $\tau = 0.001$), set $v_* = \mathbf{0}$ and skip the field forwarding [S1].
4.  **Aggregation:** Compute the expected feature vector $v$ by summing the weighted feature vectors:
    $$ v = \sum_{* \in \{static, deform, new\}} P_* v_* $$
5.  **Rendering:** Pass the aggregated feature vector $v$ and the camera viewing direction to a lightweight view-conditioned network (MLP) to predict density $\sigma$ and color $c$ [S1].

**Formulas:**
*   **Expected Feature Vector:**
    $$ v = P_{static} v_{static} + P_{deform} v_{deform} + P_{new} v_{new} $$
*   **Regularization Loss:**
    To encourage sparsity and assume most points are static or deforming, the regularization loss is:
    $$ L_{reg} = \alpha P_{deform} + P_{new} $$
    where $P_* = \frac{1}{|R_p|} \sum_{p \in R_p} P_*(p)$ is the average probability of points in the batch $R_p$, and $\alpha$ weights the ratio of deforming to new [S1].
*   **Total Loss:**
    $$ L = L_{rec} + \lambda L_{reg} $$
    where $L_{rec}$ is the reconstruction loss and $\lambda$ is a balancing hyper-parameter [S1].

**Parameters:**
*   $\tau$: Threshold for skipping fields. Default: 0.001 [S1].
*   $\alpha$: Weight for deformation probability in regularization. Value not given in sources.
*   $\lambda$: Weight for regularization loss. Value not given in sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The threshold $\tau$ prevents numerical noise from activating irrelevant fields. If $\tau$ is too high, valid features may be discarded; if too low, computational savings are lost [S1].
*   **Deformation Mapping:** The deformation field outputs a displacement $\Delta p$, not a direct feature. The feature $v_{deform}$ is obtained by querying the canonical space (static field) at the deformed location $p + \Delta p$. This requires the static field to be defined in a canonical space that accommodates the deformation [S1].

## Subtopics

### Streamable Hybrid Representation
A complementary component that uses explicit cached features for static parts and a tiny MLP for time-varying appearance to enable streaming. It reduces model size compared to fully implicit representations. [Link](7274-dynerf.md#streamable-hybrid-representation) (Note: The parent article DyNeRF does not explicitly detail this, but the source [S1] describes it as part of the overall framework. Since the parent article link is provided, we link to the parent for context, but the specific details of the streamable hybrid representation are part of the NeRFPlayer method described in [S1].) *Correction: The prompt asks for subtopics of the current method. The Decomposed Spatiotemporal Representation is a component of the NeRFPlayer framework. The "Streamable Hybrid Representation" is another component described in [S1]. It is a sibling technique within the same paper. I will list it as a subtopic if it fits the "child" definition. However, the tree structure provided is Volume Rendering > ... > DyNeRF > Decomposed Spatiotemporal Representation. There are no explicit child articles in the tree. I will leave this section empty as per instructions if no child articles exist.*

## Operating characteristics
*   **Time Complexity:** Rendering time is reduced for static regions because the stationary field $s(\cdot)$ is time-invariant and can be cached or queried without time-dependent computation. Deforming regions require computing $\Delta p$ and a canonical query. New regions require querying $n(p,t)$. The thresholding mechanism skips fields where $P_* < \tau$, further reducing computation [S1].
*   **Memory Complexity:** The method uses explicit cached features for the decomposition and newness fields, and a stationary field. This can significantly increase model size compared to a purely implicit representation if not managed carefully, but the use of "tiny MLPs" for deformation and view-conditioning helps mitigate this [S1].
*   **Strengths:** Maintains temporal consistency by modeling different areas with appropriate temporal regularizations. Saves computation by not modeling static areas with time-dependent fields. Handles rigid and non-rigid motion via the deformation field. Handles emerging content via the newness field [S1].
*   **Limitations:** Requires training a decomposition field to correctly categorize points. Performance depends on the accuracy of the decomposition probabilities. The assumption that most points are static or deforming may not hold for scenes with significant new content [S1].
*   **When to use:** Dynamic scenes with distinct static backgrounds, deforming objects, and emerging content. When temporal consistency and computational efficiency are priorities.
*   **When not to use:** Scenes with uniformly dynamic content where decomposition is difficult or inaccurate.

## Background
The Decomposed Spatiotemporal Representation was introduced in the NeRFPlayer paper by the authors of [S1]. It addresses the limitations of standard NeRF-based methods for dynamic scenes, which often struggle with temporal consistency and computational cost when modeling complex dynamics. The method builds on the observation that dynamic scenes contain areas with different temporal changing patterns (static, deforming, new) and models them separately [S1].

## Sources
[S1] NeRFPlayer: A Streamable Dynamic Scene Representation with Decomposed Neural Radiance Fields, https://doi.org/10.1109/tvcg.2023.3247082, fetched 2026-10-26
