# Pool proposer architecture

Also called: Pooling-based proposer

## Overview
The Pool proposer is a trainable network component used in NeRF-ID to generate fine-grained sampling positions from coarse features along a ray. It solves the problem of adaptive sampling by learning to propose new points where the scene geometry or appearance is complex, based on aggregated information from initial coarse samples. It is distinguished from the MLPMix proposer by its use of average pooling and sinusoidal positional encoding summation, though it exhibits higher variance and worse performance on specific complex scenes like Blender: Materials and Blender: Mic [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** $N_c = 64$ coarse features, where each feature is a vector of size $D$ (typically matching the MLP hidden dimension, e.g., 256).
*   **Output:** $N_f = 128$ proposed sampling positions (and associated features) along the ray.

**Data Structures:**
*   Coarse features: Tensor of shape $(N_c, D)$.
*   Bottleneck Layer: A single fully connected (FC) layer with input dimension 256 and output dimension 256.
*   Positional Encoding: Sinusoidal encodings applied to spatial positions.

**Steps:**
1.  **Feature Aggregation:** The $N_c$ coarse features are processed. The most expensive operations occur before average pooling. The features are passed through the bottleneck FC layer ($256 \times 256$) [S1, unofficial].
2.  **Positional Encoding:** Sinusoidal positional encodings are computed for the spatial positions of the coarse samples along the ray.
3.  **Summation:** The sinusoidal positional encodings are summed with the processed features.
4.  **Proposal Generation:** The resulting aggregated representation is used to generate $N_f = 128$ new sampling positions. The exact mechanism of mapping the aggregated feature to positions is not detailed beyond the summation and bottleneck application, but the architecture assumes this process yields the final proposals [S1, unofficial].

**Parameters:**
*   $N_c = 64$: Number of coarse samples.
*   $N_f = 128$: Number of proposed fine samples.
*   Hidden Dimension: 256.

**Edge Cases and Pitfalls:**
*   **Blind Proposer:** Ignoring input coarse features (always outputting the same proposals) performs badly, indicating the input features are necessary [S1, unofficial].
*   **No-position Proposer:** Ignoring the positions of coarse samples along the ray performs badly on Blender scenes because it is insensitive to feature ordering (e.g., producing identical proposals for colinear rays of opposite directionality). It remains competitive on LLFF-NeRF scenes, which are forward-facing [S1, unofficial].
*   **Variance:** The Pool proposer exhibits higher variance in performance compared to MLPMix [S1, unofficial].

## Subtopics

### MLPMix Proposer
The MLPMix proposer is an alternative to the Pool proposer. It is preferred over the Transformer proposer because the Transformer shows significantly worse performance on scenes like Blender: Ficus, Blender: Lego, and LLFF-NeRF: T-Rex, along with unstable training curves. Pool and MLPMix are generally on par, but Pool has higher variance and is significantly worse on Blender: Materials and Blender: Mic [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
The most expensive operations in Pool and MLPMix are those applied to each of the $N_c = 64$ coarse features before average pooling. The bottleneck is a single fully connected layer of size $256 \times 256$. Since the NeRF backbone already applies FC layers of this size on $8 \times (N_c + N_c + N_f) = 2048$ vectors, the additional application on $N_c = 64$ vectors is negligible [S1, unofficial]. The Transformer proposer is somewhat more expensive due to the lack of pooling operations, requiring heavy constraints to keep costs reasonable [S1, unofficial].

**Measured Performance:**
*   **Blender: Materials:** Pool is significantly worse than MLPMix [S1, unofficial].
*   **Blender: Mic:** Pool is significantly worse than MLPMix [S1, unofficial].
*   **General:** Pool and MLPMix tend to be the best among the three proposer architectures (Pool, MLPMix, Transformer), with findings being statistically significant [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Negligible additional computational cost compared to the backbone; generally competitive performance.
*   **Limitations:** Higher variance than MLPMix; poor performance on specific complex scenes (Blender: Materials, Blender: Mic) [S1, unofficial].

## Background
The Pool proposer is part of the NeRF-ID framework, which extends the original NeRF method. The architectures of the coarse and fine networks are identical to those in the original NeRF paper [S1, unofficial]. The proposer architectures were introduced to improve sampling efficiency and reconstruction quality by learning to propose better sampling positions. The work is based on the paper "NeRF in detail: Learning to sample for view synthesis" [S1, unofficial].

## Sources
[S1] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
