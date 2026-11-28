# Temporal Sensitivity Pruning


## Overview
Temporal Sensitivity Pruning (TSP) is a method for eliminating redundant 3D Gaussian primitives in dynamic scene reconstruction by evaluating their cumulative impact on the reconstruction error over time [S1, unofficial]. It distinguishes itself from static pruning methods by aggregating sensitivity scores across multiple training poses and timesteps, thereby identifying Gaussians that contribute negligibly to the dynamic scene's appearance across the entire temporal sequence [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** The set of 3D Gaussians $\mathcal{G} = \{\mathcal{G}_1, \dots, \mathcal{G}_N\}$, where each $\mathcal{G}_i$ has parameters including position, rotation, scale, and opacity. The set of training views $\mathcal{P}_{gt}$, consisting of camera poses $\phi$ and corresponding timesteps $t$.
*   **Output:** A binary mask or updated set $\mathcal{G}'$ containing only Gaussians with a temporal sensitivity score $\tilde{U}_{\mathcal{G}_i}$ above a defined threshold.

**Data Structures**
*   **Gaussian Parameters:** Each $\mathcal{G}_i$ is defined by its 3D mean $\mu_i$, rotation quaternion $q_i$, scale vector $s_i$, and opacity $\alpha_i$. In dynamic settings, these are often functions of time $t$ via a deformation field.
*   **Sensitivity Score:** A scalar value $\tilde{U}_{\mathcal{G}_i}$ stored per Gaussian, initialized to zero.

**Algorithm Steps**
1.  **Iterate over Gaussians:** For each Gaussian $\mathcal{G}_i$ in the current set $\mathcal{G}$:
2.  **Initialize Score:** Set $\tilde{U}_{\mathcal{G}_i} = 0$.
3.  **Aggregate Sensitivity:** For each training view $(\phi, t) \in \mathcal{P}_{gt}$:
    a. Render the image $I_{\mathcal{G}_t}(\phi)$ using the current Gaussian parameters at timestep $t$ and pose $\phi$.
    b. Compute the gradient of the rendered image with respect to the projected value of $\mathcal{G}_i$. Let $g_i$ denote the contribution of $\mathcal{G}_i$ to the pixel colors in the image space. Calculate $\nabla_{g_i} I_{\mathcal{G}_t}(\phi)$.
    c. Square this gradient: $\left( \nabla_{g_i} I_{\mathcal{G}_t}(\phi) \right)^2$.
    d. Add the squared gradient to $\tilde{U}_{\mathcal{G}_i}$.
4.  **Prune:** After computing $\tilde{U}_{\mathcal{G}_i}$ for all Gaussians, remove any $\mathcal{G}_i$ where $\tilde{U}_{\mathcal{G}_i} < \tau$, where $\tau$ is a pruning threshold. [S1, unofficial]

**Formula**
The temporal sensitivity score $\tilde{U}_{\mathcal{G}_i}$ approximates the second-order sensitivity to the $L_2$ loss:
$$
\tilde{U}_{\mathcal{G}_i} \approx \nabla_{g_i}^2 L_2 \approx \sum_{\phi, t \in \mathcal{P}_{gt}} \left( \nabla_{g_i} I_{\mathcal{G}_t}(\phi) \right)^2
$$
Where:
*   $\tilde{U}_{\mathcal{G}_i}$ is the TSP score for Gaussian $\mathcal{G}_i$.
*   $g_i$ is the value of $\mathcal{G}_i$ projected onto image space.
*   $\mathcal{P}_{gt}$ is the set of training poses and timesteps.
*   $I_{\mathcal{G}_t}(\phi)$ is the rendered image at pose $\phi$ and timestep $t$. [S1, unofficial]

**Parameters**
*   $\mathcal{P}_{gt}$: The set of training poses and timesteps used for sensitivity aggregation. [S1, unofficial]
*   Threshold $\tau$: Not given in the sources.

**Edge Cases and Pitfalls**
*   **Gradient Computation:** The gradients $\nabla_{g_i} I_{\mathcal{G}_t}(\phi)$ must be computed with respect to the projected Gaussian value, which depends on the deformation field at time $t$. Since deformation updates vary across timesteps, these gradients are inherently time-dependent [S1, unofficial].
*   **Computational Cost:** Calculating gradients for all Gaussians across all training views and timesteps can be expensive. TSP is typically applied periodically during training to reduce neural inference load rather than at every step [S1, unofficial].
*   **Floater Removal:** While TSP identifies redundant Gaussians based on observed views, it may not fully address "floaters" (unstable Gaussians that appear static in observed views but drift at unseen timestamps). This is often addressed by complementary methods like Temporal Sensitivity Sampling (TSS) [S1, unofficial].

## Subtopics

### [Sensitivity Calculation and Pruning](11423-sensitivity-calculation-and-pruning.md)
Sensitivity Calculation and Pruning is a method to eliminate 3D Gaussians that have no impact on training images by calculating sensitivity using the gradient of total image energy with respect to parameters.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot |\mathcal{P}_{gt}|)$, where $N$ is the number of Gaussians and $|\mathcal{P}_{gt}|$ is the number of training views/timesteps. Gradient computation per view is required.
*   **Memory Complexity:** $O(N)$ to store the sensitivity scores.
*   **Performance:** Integrating TSP into DeformableGS accelerates rendering by 6.78× on average while maintaining neural-field fidelity and using 10× fewer primitives [S1, unofficial].
*   **Strengths:** Reduces redundant primitives, stabilizes temporal pruning, and lowers deformation cost [S1, unofficial].
*   **Limitations:** Relies on gradients from discrete training frames, potentially overlooking floaters [S1, unofficial].

## Background
Temporal Sensitivity Pruning was introduced in the context of Speedy Deformable 3D Gaussian Splatting (SpeeDe3DGS) [S1, unofficial]. It builds on DeformableGS and aims to bridge the efficiency-fidelity gap in dynamic scene reconstruction by removing low-impact Gaussians via temporally aggregated sensitivity analysis [S1, unofficial].

## Sources
[S1] SpeeDe3DGS: Speedy Deformable 3D Gaussian Splatting with Temporal Pruning and Motion Grouping, https://speede3dgs.github.io/, fetched 2026-10-29 (unofficial)
