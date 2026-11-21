# Temporal Sensitivity Pruning


## Overview
Temporal Sensitivity Pruning (TSP) is a sparsification method for dynamic 3D Gaussian Splatting that removes low-impact primitives by aggregating gradient-based sensitivity scores across multiple training views and timesteps [S1, unofficial]. It differs from standard opacity-based pruning by explicitly accounting for the temporal consistency and deformation dynamics of the scene, ensuring that Gaussians contributing to stable, high-fidelity reconstruction over time are retained while those causing redundant neural inference load are eliminated [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\mathcal{G} = \{\mathcal{G}_1, \dots, \mathcal{G}_N\}$, where each $\mathcal{G}_i$ has parameters $\{\mu_i, r_i, s_i, \alpha_i, \mathbf{c}_i\}$ and a deformation field $\mathcal{D}$ mapping static parameters to time-dependent states. A set of training poses and timesteps $\mathcal{P}_{gt}$.
*   **Output:** A binary mask or updated set of Gaussians indicating which primitives are pruned.

**Data Structures:**
*   $\tilde{U}_{\mathcal{G}_i}$: A scalar score per Gaussian, initialized to 0.
*   $\mathcal{P}_{gt}$: A list or set of tuples $(\phi, t)$, where $\phi$ is a camera pose and $t$ is a timestep.

**Steps:**
1.  **Initialize Scores:** For each Gaussian $\mathcal{G}_i$, initialize the temporal sensitivity score $\tilde{U}_{\mathcal{G}_i} = 0$.
2.  **Iterate Training Data:** For each training configuration $(\phi, t) \in \mathcal{P}_{gt}$:
    a.  **Deform:** Compute the time-dependent parameters for $\mathcal{G}_i$ using the deformation field: $(\mu'_i, r'_i, s'_i) = \mathcal{D}(\mu_i, r_i, s_i, t)$.
    b.  **Render Gradient:** Calculate the gradient of the rendered image $I_{\mathcal{G}_t}(\phi)$ with respect to the projected value $g_i$ of the Gaussian $\mathcal{G}_i$ in image space. Let this be $\nabla_{g_i} I_{\mathcal{G}_t}(\phi)$.
    c.  **Accumulate:** Add the squared gradient to the score:
        $$ \tilde{U}_{\mathcal{G}_i} \leftarrow \tilde{U}_{\mathcal{G}_i} + \left( \nabla_{g_i} I_{\mathcal{G}_t}(\phi) \right)^2 $$
3.  **Prune:** Periodically during training, compare $\tilde{U}_{\mathcal{G}_i}$ against a threshold (or rank order). Prune Gaussians with low cumulative scores to reduce the number of primitives and neural inference load [S1, unofficial].

**Formula:**
The TSP score $\tilde{U}_{\mathcal{G}_i}$ approximates the second-order sensitivity to the $L_2$ loss:
$$ \tilde{U}_{\mathcal{G}_i} \approx \nabla_{g_i}^2 L_2 \approx \sum_{(\phi, t) \in \mathcal{P}_{gt}} \left( \nabla_{g_i} I_{\mathcal{G}_t}(\phi) \right)^2 $$
Where:
*   $\tilde{U}_{\mathcal{G}_i}$ is the temporal sensitivity score for Gaussian $\mathcal{G}_i$.
*   $g_i$ is the value of $\mathcal{G}_i$ projected onto the image space.
*   $\mathcal{P}_{gt}$ is the set of training poses $\phi$ and timesteps $t$.
*   $I_{\mathcal{G}_t}(\phi)$ is the rendered image at pose $\phi$ and timestep $t$.

**Parameters:**
*   $\mathcal{P}_{gt}$: The set of training poses and timesteps used for sensitivity aggregation. The choice of this set determines the temporal coverage of the sensitivity analysis [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Gradient Noise:** Since gradients are computed from discrete training frames, they may be noisy. This can lead to the retention of "floaters" (unstable Gaussians) that appear static in observed views but drift at unseen timestamps [S1, unofficial]. This is often addressed by combining TSP with Temporal Sensitivity Sampling (TSS) [S1, unofficial].
*   **Deformation Variability:** Gradients are inherently time-dependent due to deformation updates. Incorrectly handling the deformation field $\mathcal{D}$ during gradient computation can lead to inaccurate sensitivity scores [S1, unofficial].

## Subtopics

### [Sensitivity Calculation and Pruning](11423-sensitivity-calculation-and-pruning.md)
Uses first-order gradients of total image energy. Simpler but less robust to temporal dynamics than TSP.

### [Temporal Sensitivity Sampling](6350-temporal-sensitivity-sampling.md)
Perturbs timestamps during sensitivity estimation to suppress floaters. Used in conjunction with TSP.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot |\mathcal{P}_{gt}|)$, where $N$ is the number of Gaussians and $|\mathcal{P}_{gt}|$ is the number of training poses/timesteps. Requires rendering and gradient computation for each Gaussian across all training views.
*   **Memory Complexity:** $O(N)$ to store the sensitivity scores $\tilde{U}_{\mathcal{G}_i}$.
*   **Performance:** In dynamic scenes, integrating TSP into DeformableGS accelerated rendering by 6.78× on average while maintaining neural-field fidelity and using 10× fewer primitives [S1, unofficial].
*   **Strengths:** Reduces neural inference load by removing low-impact Gaussians. Stabilizes temporal pruning by considering temporal consistency.
*   **Limitations:** Computationally expensive to calculate sensitivity scores for all Gaussians across all timesteps. May require complementary methods (like TSS) to handle floaters.

## Background
TSP was introduced in the context of Speedy Deformable 3D Gaussian Splatting (SpeeDe3DGS) to address the computational expense of neural inference in dynamic 3D Gaussian Splatting models [S1, unofficial]. It builds on DeformableGS and aims to bridge the efficiency-fidelity gap by removing redundant primitives through temporally aggregated sensitivity analysis [S1, unofficial].

## Sources
[S1] SpeeDe3DGS: Speedy Deformable 3D Gaussian Splatting with Temporal Pruning and Motion Grouping, https://speede3dgs.github.io/, fetched 2026-10-29 (unofficial)
