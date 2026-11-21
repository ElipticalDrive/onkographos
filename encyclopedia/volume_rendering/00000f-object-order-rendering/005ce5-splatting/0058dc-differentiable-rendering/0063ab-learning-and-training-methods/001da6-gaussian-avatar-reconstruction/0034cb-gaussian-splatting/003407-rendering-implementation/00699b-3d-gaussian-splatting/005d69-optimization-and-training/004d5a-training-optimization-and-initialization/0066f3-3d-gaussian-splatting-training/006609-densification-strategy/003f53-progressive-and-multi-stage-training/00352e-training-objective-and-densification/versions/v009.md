# Training Objective and Densification

Also called: MeshGS Training, Tightly/Loosely-bound Training

## Overview
This method defines the loss functions and densification protocols for **MeshGS** [S1], a variant of 3D Gaussian Splatting that aligns primitives with an explicit mesh surface. It solves the problem of Gaussian splats drifting away from geometry or expanding uncontrollably by introducing surface-aware constraints. It differs from standard 3DGS [22] by splitting the primitive set into **tightly-bound** ($G^t$) and **loosely-bound** ($G^l$) splats, applying mesh-specific losses and densification only to the former, while retaining standard 3DGS behavior for the latter.

## How it works
**Inputs:**
*   Rendered image $\hat{I}_i$ and ground truth image $I_i$.
*   Set of tightly-bound Gaussian splats $G^t$ and loosely-bound splats $G^l$.
*   Mesh $M$ consisting of faces $F$, where each face $f_j$ has a normal $n_f$ and vertices $v_i$.
*   For each Gaussian $i \in G^t$: center $P_i$, scale parameters $S_i = (s_1, s_2, s_3)$, rotation matrix $R_i$.

**Outputs:**
*   Total loss $L$ for backpropagation.
*   Updated set of Gaussian splats (via densification).

**Data Structures:**
*   **Gaussian Splat:** Position $P_i \in \mathbb{R}^3$, Scale $S_i \in \mathbb{R}^3$, Rotation $R_i \in SO(3)$, Opacity, Color/SH coefficients.
*   **Mesh Face:** Vertices $v_i \in \mathbb{R}^3$, Normal $n_f \in \mathbb{R}^3$.

**Parameters:**
*   $\lambda_{img} = 0.001$ [S1]
*   $\lambda_{nc} = 0.1$ [S1]
*   $\lambda_{min} = 0.1$ [S1]
*   $\lambda_{max} = 10$ [S1]
*   $\lambda_{proj} = 50$ [S1]
*   $\rho = 0.1$ (scale threshold) [S1]
*   $d_{th} = 0.01$ (distance threshold for binding) [S1]
*   Split scale factor: $1.6$ [S1]

**Steps:**

1.  **Image Loss:** Compute the combination of L1 loss and D-SSIM between the rendered image $\hat{I}_i$ and ground truth $I_i$ [S1]:
    $$L_{img} = (1-\lambda_{img})L_{1}(\hat{I}_i, I_i) + \lambda_{img}SSIM(\hat{I}_i, I_i)$$

2.  **Normal Consistency Loss:** For each tightly-bound splat $i \in G^t$, estimate its normal $n_i$ by identifying the minimum scale axis among $S_i$ and applying the rotation matrix $R_i$ to transform it into world coordinates. Compute the cosine similarity between $n_i$ and the normal $n_f$ of the corresponding mesh face $f \in F$ [S1]:
    $$L_{nc} = \sum_{i \in G^t, f \in F} (1 - n_i \cdot n_f)$$

3.  **Scale Regularization Loss:** Regularize the scale parameters $S_i = (s_1, s_2, s_3)$ of tightly-bound splats to prevent excessive expansion. Penalize the minimum scale and the deviation of the maximum scale from the threshold $\rho$ [S1]:
    $$L_{scale} = \sum_{i \in G^t} (\lambda_{min}|\min(s_1, s_2, s_3)| + \lambda_{max}|\max(s_1, s_2, s_3) - \rho|)$$

4.  **Projection Loss:** Project the center $P_i$ of each tightly-bound splat to the closest point $s_i$ on the mesh surface defined by face $f_j$ and vertex $v_i$. The projection is calculated as [S1]:
    $$s_i = P_i + ((P_i - v_i) \cdot f_j) \cdot f_j$$
    Compute the distance loss [S1]:
    $$L_{proj} = \sum_{i \in G^t} \|s_i - P_i\|_2$$

5.  **Total Loss:** Combine all terms [S1]:
    $$L = L_{img} + \lambda_{nc} L_{nc} + L_{scale} + \lambda_{proj} L_{proj}$$

6.  **Densification Strategy:**
    *   **Loosely-bound splats ($G^l$):** Follow the original 3DGS densification strategy (gradient-based cloning and splitting) [S1].
    *   **Tightly-bound splats ($G^t$):** Perform cloning and splitting by sampling points from the corresponding mesh triangle. Initialize new Gaussian splats at these sampled positions [S1].
        *   **Cloning:** Copy additional attributes from the parent Gaussian to the new splat [S1].
        *   **Splitting:** Divide the scale of the parent Gaussian by a factor of $1.6$ [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The projection formula assumes $f_j$ is a normalized vector. If not, normalization is required before dot products [G].
*   **Binding Threshold:** The distinction between $G^t$ and $G^l$ relies on a distance threshold $d_{th} = 0.01$. Splats within this distance of the mesh are tightly-bound [S1].
*   **Scale Threshold:** The parameter $\rho = 0.1$ limits the maximum scale of tightly-bound splats. Values significantly larger or smaller may require tuning for specific scene scales [S1].

## Subtopics

## Operating characteristics
*   **Time Complexity:** The projection loss requires finding the closest mesh face for each tightly-bound splat, which is $O(|G^t| \cdot |F|)$ if brute-force, or $O(|G^t| \log |F|)$ with spatial indexing. Normal estimation is $O(|G^t|)$. Total loss computation is linear in the number of splats and faces.
*   **Memory Complexity:** Requires storing the mesh geometry ($F$) and the additional loss terms for tightly-bound splats. No significant memory overhead compared to standard 3DGS beyond the mesh representation.
*   **Performance:** Evaluated on Mip-NeRF360 and Deep Blending datasets. Training runs for 30,000 iterations [S1].
*   **Strengths:** Prevents Gaussian splats from drifting away from the mesh surface and controls scale expansion, leading to high-quality rendering aligned with explicit geometry.
*   **Limitations:** Relies on the quality of the input mesh. The fixed threshold $d_{th}$ and $\rho$ may not generalize perfectly across scenes with varying scales without adjustment.
*   **When to use:** When an explicit mesh is available and high-fidelity alignment with that mesh is required.
*   **When not to use:** When no mesh is available, or when the scene geometry is highly complex/degenerate such that mesh projection is ambiguous.

## Background
This method originates from **MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering** [S1], published in 2024. It builds upon the original 3D Gaussian Splatting [22] and integrates concepts from neural implicit surface reconstruction (e.g., LTM [7]) and mesh-based Gaussian representations (e.g., SuGaR [13]). The approach addresses the limitation of standard 3DGS in explicitly representing geometry by binding a subset of Gaussians to a mesh surface.

## Sources
[S1] MeshGS: Adaptive Mesh-Aligned Gaussian Splatting for High-Quality Rendering, https://arxiv.org/abs/2410.08941v1, fetched 2026-10-26
