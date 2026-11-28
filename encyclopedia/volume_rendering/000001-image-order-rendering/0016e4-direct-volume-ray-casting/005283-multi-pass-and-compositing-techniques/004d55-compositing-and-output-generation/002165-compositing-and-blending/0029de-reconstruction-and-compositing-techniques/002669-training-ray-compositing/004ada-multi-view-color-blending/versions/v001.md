# Multi-View Color Blending

Also called: Feature Blending, Pixel Alignment Feature Aggregation

## Overview
Multi-View Color Blending fuses original colors from $N$ reference views into query points using view-dependent weights derived from feature similarity and view direction [S1, S2]. It differs from standard alpha compositing by explicitly aggregating multi-view pixel features before density/color prediction, addressing view-dependent appearance and occlusion in generalizable neural radiance fields [S1, S2].

## How it works
**Inputs:**
- A set of $K$ reference views with images $I^k$ and feature maps $f^k \in \mathbb{R}^{H \times W \times C}$ [S2].
- A 3D query point $p \in \mathbb{R}^3$ [S2].
- Camera parameters for each reference view $k$ [S2].
- View direction $\vec{v}$ of point $p$ in the target camera view [S2].

**Outputs:**
- Aggregated pixel alignment feature $F_p$ [S2].
- Blending weights $w^k_q$ for each view $k$ [S2].
- Final color $c$ and density $\sigma$ at $p$ (via subsequent NeRF prediction) [S2].

**Data Structures:**
- $f^k$: Feature tensor for view $k$, shape $(H, W, C)$ [S2].
- $w^k_q$: Scalar blending weight for view $k$ [S2].
- $F_p$: Aggregated feature vector, shape $(C)$ [S2].
- $\vec{v}$: View direction vector [S2].
- $\theta^k_q$: Angle scalar between $\vec{v}$ and the ray from $p$ to projected point $q_k$ [S2].

**Steps:**
1. For each reference view $k \in \{1, \dots, K\}$:
   a. Project point $p$ into view $k$ to obtain 2D coordinates $q_k \in \mathbb{R}^2$ [S2].
   b. Fetch the feature vector $f^k_{q_k}$ from the feature map $f^k$ at location $q_k$ [S2].
   c. Compute the angle $\theta^k_q$ between the target view direction $\vec{v}$ and the vector from $p$ to $q_k$ [S2].
   d. Estimate blending weight $w^k_q$ using an MLP network $\mathbf{MLP}_{\mathcal{B}}$ with inputs $\vec{v}$, $\theta^k_q$, and $f^k_{q_k}$ [S2]:
      $$ w^k_q = \mathbf{MLP}_{\mathcal{B}} (\vec{v}, \theta^k_q, f^k_{q_k}) $$
2. Compute the aggregated pixel alignment feature $F_p$ as a weighted summation of image features:
   $$ F_p = \sum_{k=1}^{K} w^k_q f^k_{q_k} $$
3. Pass $F_p$, the transformed 3D location $p'$, and view direction $\vec{v}$ to the neural radiance field $\Phi$ to predict density $\sigma$ and color $c$ [S2].

**Parameters:**
- $K$: Number of reference views. Typical value: 24 (as used in HumanNeRF) [S2].
- $C$: Feature channel dimension. Not given in the sources.

**Edge Cases and Pitfalls:**
- Projection $q_k$ may fall outside image bounds; handling is not specified [G].
- MLP inputs must be normalized or scaled appropriately for stable training [G].
- Feature interpolation at $q_k$ requires care for sub-pixel accuracy [G].

## Subtopics

### Feature Blending
Aggregates pixel alignment features from multiple source views to provide input to the neural radiance field, using viewing directions and angles relative to source views [S2]. It is chosen when view-dependent appearance and sparse input generalization are critical, such as in dynamic human radiance fields [S2]. It differs from standard color blending by operating on intermediate feature maps rather than raw colors, allowing richer representation [S2]. See [HumanNeRF](https://doi.org/10.1109/cvpr52688.2022.00759) [S2].

## Operating characteristics
- **Time Complexity:** $O(K \cdot (P + M))$, where $K$ is the number of views, $P$ is projection cost, and $M$ is MLP evaluation cost per view [S2].
- **Memory Complexity:** $O(K \cdot C)$ to store feature vectors and weights per query point [S2].
- **Measured Performance:** HumanNeRF uses 24 views and achieves efficient generation from sparse inputs [S2].
- **Strengths:** Handles view-dependent effects and occlusions better than single-view methods [S1, S2].
- **Limitations:** Requires multiple reference views and additional MLP computation [S2].
- **When to Use:** Generalizable NeRF with sparse inputs, dynamic scenes [S2].
- **When Not to Use:** Dense multi-view settings with standard volume rendering may not need this complexity [G].

## Background
Originated in generalizable neural radiance fields for efficient synthesis from sparse inputs [S1, S2]. HumanNeRF introduced feature blending for dynamic human radiance fields from sparse inputs [S2]. NeRF-SDP used color blending for scene depth perception [S1].

## Sources
[S1] NeRF-SDP: Efficient Generalizable Neural Radiance Field with Scene Depth Perception, https://doi.org/10.1145/3595916.3626380, fetched 2026-10-30
[S2] HumanNeRF: Efficiently Generated Human Radiance Field from Sparse Inputs, https://doi.org/10.1109/cvpr52688.2022.00759, fetched 2026-10-26
