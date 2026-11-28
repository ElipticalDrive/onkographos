# Static-Dynamic Factorization Regularization

Also called: Lslo(r), Static-Dynamic Density Regularization

## Overview
Static-Dynamic Factorization Regularization enforces a physically plausible decomposition of a scene by encouraging the spatial ratio of static to dynamic density to be binary (0 or 1) [S1]. This ensures that any point in space is occupied by either static or dynamic objects, but not both, preventing visual artifacts caused by overlapping static and dynamic components [S1]. It is a specific regularization term used within the SUDS framework for large-scale urban dynamic scene reconstruction [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** The static density field $s_t(\mathbf{x}, t)$ at time $t$, and the temporal neighborhood $t_0 \in [t-1, t+1]$ [S1].
*   **Output:** A scalar regularization loss value $\mathcal{L}_{slo}(r)$ [S1].

**Data Structures:**
*   $s_t(\mathbf{x}, t)$: The predicted static density at spatial location $\mathbf{x}$ and time $t$ [S1].
*   $t$: The current time step [S1].

**Formula:**
The regularization term is defined as:
$$
\mathcal{L}_{slo}(r) = \sum_{t_0 \in [t-1, t+1]} \int_{\mathbf{x}} s_{t_0}(\mathbf{x}, t) \, d\mathbf{x}
$$
*Note: The source text contains garbled PDF math for the exact equation structure, but the summation over the temporal neighborhood and the integral over space of the static density is the recoverable form [S1].*

**Parameters:**
*   Temporal window: $[t-1, t+1]$ [S1].

**Steps:**
1.  For the current time step $t$, identify the temporal neighbors $t_0 \in \{t-1, t, t+1\}$ [S1].
2.  For each neighbor $t_0$, compute the integral of the static density $s_{t_0}(\mathbf{x}, t)$ over the spatial domain $\mathbf{x}$ [S1].
3.  Sum these integrals to produce $\mathcal{L}_{slo}(r)$ [S1].

**Edge Cases and Pitfalls:**
*   The exact mathematical formulation in the source is garbled, so implementation requires careful reconstruction of the integral and summation operations [S1].
*   The regularization encourages sparsity in the static density field, which may lead to under-segmentation if the static background is complex [S1].

## Subtopics

### Static-Dynamic Factorization
This variant involves estimating a second static volume that is not conditioned on the deformation [S2, unofficial]. It introduces soft regularization losses to constrain static scene content [S2, unofficial]. In some implementations, the static NeRF is trained on observations that do not contain moving and deforming parts, using a binary segmentation mask as input [S2, unofficial]. This approach allows the static background to exploit multi-view clues from monocular input recordings across time [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The method is evaluated on the City-1M dataset, which consists of 1.28 million images across 1700 videos gathered across a 105 km² urban area [S1].
*   **Strengths:** Enforces physically plausible scene decomposition [S1].
*   **Limitations:** The exact formula is garbled in the source, making precise implementation challenging [S1].

## Background
The method is part of the SUDS (Scalable Urban Dynamic Scenes) framework, which evaluates large-scale reconstruction abilities [S1]. It was introduced in the paper "SUDS: Scalable Urban Dynamic Scenes" [S1]. The concept of separating static and dynamic components is also discussed in broader reviews of neural rendering advances, where estimating a second static volume is noted as a common technique for handling static backgrounds [S2, unofficial].

## Sources
[S1] SUDS: Scalable Urban Dynamic Scenes, https://doi.org/10.1109/cvpr52729.2023.01191, fetched 2026-10-26
[S2] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
