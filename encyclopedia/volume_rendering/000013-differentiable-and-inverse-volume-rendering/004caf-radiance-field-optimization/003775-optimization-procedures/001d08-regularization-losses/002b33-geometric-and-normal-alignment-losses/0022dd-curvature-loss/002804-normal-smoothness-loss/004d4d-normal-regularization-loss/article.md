# Normal Regularization Loss

Also called: Monocular Normals Regularization

## Overview
Normal Regularization Loss enforces geometric consistency in differentiable volume rendering and Gaussian splatting by combining monocular normal supervision with a total variation smoothness prior on predicted normals [S1]. It distinguishes itself from sibling methods like [Normal Smoothness Loss](4190-normal-smoothness-loss.md) by explicitly incorporating external monocular normal estimates to anchor the geometry, rather than relying solely on internal spatial coherence [S1].

## How it works
The method minimizes a composite loss $\mathcal{L}_{\text{normal}}$ defined as the sum of a monocular normal loss $\mathcal{L}_{\hat{N}}$ and a smoothness prior $\mathcal{L}_{\text{smooth}}$ [S1].

**Inputs and Outputs:**
*   **Inputs:** Predicted normal map $\hat{\boldsymbol{N}} \in \mathbb{R}^{H \times W \times 3}$, monocular normal estimates $\boldsymbol{N}_{\text{mono}}$, and scaling parameters $\boldsymbol{s}_i$ for each Gaussian primitive $i$ [S1].
*   **Outputs:** Scalar loss value $\mathcal{L}_{\text{normal}}$ [S1].

**Data Structures:**
*   $\hat{\boldsymbol{N}}_{i,j}$: The predicted normal vector at pixel coordinates $(i, j)$ [S1].
*   $\boldsymbol{s}_i$: A 3D vector representing the scaling axes of the $i$-th Gaussian [S1].

**Formulas:**
1.  **Monocular Normal Loss ($\mathcal{L}_{\hat{N}}$):**
    An L1 loss between the predicted normals and monocular estimates [S1].
    $$ \mathcal{L}_{\hat{N}} = \sum_{i,j} \| \hat{\boldsymbol{N}}_{i,j} - \boldsymbol{N}_{\text{mono}, i,j} \|_1 $$
2.  **Smoothness Loss ($\mathcal{L}_{\text{smooth}}$):**
    Encourages spatial smoothness using finite differences [S1].
    $$ \mathcal{L}_{\text{smooth}} = \sum_{i,j} \left( |\nabla_i \hat{\boldsymbol{N}}_{i,j}| + |\nabla_j \hat{\boldsymbol{N}}_{i,j}| \right) $$
    Where $\nabla$ is the finite difference operator convolving with $[-1, 1]$ for the $i$-axis and $[-1, 1]^\top$ for the $j$-axis [S1].
3.  **Scale Loss ($\mathcal{L}_{\text{scale}}$):**
    Applied to force Gaussians to become disc-like surfels, aiding normal estimation convergence [S1].
    $$ \mathcal{L}_{\text{scale}} = \sum_i \|\arg\min(\boldsymbol{s}_i)\|_1 $$
    The normal $\hat{\boldsymbol{n}}_i$ is derived from the scaling axes via a rotation matrix $R$ and a one-hot encoding of the minimum axis index [S1]:
    $$ \hat{\boldsymbol{n}}_i = R \cdot \text{OneHot}(\arg\min(\boldsymbol{s}_i)) $$
    The rotation matrix $R$ is constructed from quaternion $q = (w, x, y, z)^\top$ [S1]:
    $$ R = \begin{bmatrix} 1 - 2(y^2 + z^2) & 2(xy - wz) & 2(xz + wy) \\ 2(xy + wz) & 1 - 2(x^2 + z^2) & 2(yz - wx) \\ 2(xz - wy) & 2(yz + wx) & 1 - 2(x^2 + y^2) \end{bmatrix} $$
4.  **Total Loss:**
    $$ \mathcal{L}_{\text{normal}} = \mathcal{L}_{\hat{N}} + \mathcal{L}_{\text{smooth}} $$ [S1].

**Parameters:**
*   Weight for smoothness prior: Not given in the sources. [S1]
*   Weight for scale loss: Not given in the sources. [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normal_regularization_loss(predicted_normals, monocular_normals, scales):
    # predicted_normals: array of shape (H, W, 3)
    # monocular_normals: array of shape (H, W, 3)
    # scales: array of shape (N, 3)
    
    H, W, _ = predicted_normals.shape
    N = scales.shape[0]
    
    # 1. Monocular Normal Loss (L1)
    l_mono = 0.0
    for i in range(H):
        for j in range(W):
            for k in range(3):
                l_mono += abs(predicted_normals[i, j, k] - monocular_normals[i, j, k])
                
    # 2. Smoothness Loss (Total Variation)
    l_smooth = 0.0
    for i in range(H):
        for j in range(W):
            # i-axis difference (row direction)
            if i < H - 1:
                diff_i = [predicted_normals[i+1, j, k] - predicted_normals[i, j, k] for k in range(3)]
                l_smooth += sum(abs(d) for d in diff_i)
            # j-axis difference (column direction)
            if j < W - 1:
                diff_j = [predicted_normals[i, j+1, k] - predicted_normals[i, j, k] for k in range(3)]
                l_smooth += sum(abs(d) for d in diff_j)
                
    # 3. Scale Loss (auxiliary, not included in L_normal per source formula)
    l_scale = 0.0
    for i in range(N):
        min_idx = 0
        for k in range(1, 3):
            if scales[i, k] < scales[i, min_idx]:
                min_idx = k
        l_scale += min_idx  # ||argmin(s_i)||_1 for a scalar index is the index itself [S1]
        
    return l_mono + l_smooth
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Not given in the sources. [S1]
*   **Precision:** Not given in the sources. [S1]
*   **Degenerate Input:** Not given in the sources. [S1]

## Subtopics

### Total Variation Normal Loss
A variant that conditions the total variation loss on the predicted normal map and the reference image to enforce spatial smoothness, particularly useful in inverse rendering [S2]. It differs from the standard smoothness loss by using image intensity gradients as edge-aware weights [S2].
*   **Definition:** A total variation loss conditioned by the predicted normal map $\hat{\boldsymbol{N}}$ and the reference image $I$ [S2].
*   **Formula:**
    $$ L_n = L_{n-p} + \lambda_{n-TV} \cdot TV_{\text{normal}} $$
    $$ TV_{\text{normal}} = \frac{1}{|\hat{\boldsymbol{N}}|} \sum_{i,j} \left[ \exp(-|I_{i,j} - I_{i-1,j}|) (\hat{\boldsymbol{N}}_{i,j} - \hat{\boldsymbol{N}}_{i-1,j})^2 + \exp(-|I_{i,j} - I_{i,j-1}|) (\hat{\boldsymbol{N}}_{i,j} - \hat{\boldsymbol{N}}_{i,j-1})^2 \right] $$
    Where $|\hat{\boldsymbol{N}}|$ is the number of pixels, $I$ is the reference image, and $\lambda_{n-TV}$ is the smoothing weight [S2].
*   **Parameters:** $\lambda_{n-TV} = 5.0$ [S2].
*   **When to choose:** Use when working with inverse rendering tasks where preserving sharp edges in the normal map corresponding to object boundaries is critical, as the exponential weighting suppresses smoothing across intensity discontinuities [S2].
*   **Link:** [GS-IR](https://doi.org/10.1109/cvpr52733.2024.02045)

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [S1]
*   **Memory Complexity:** Not given in the sources. [S1]
*   **Strengths:** Not given in the sources. [S1]
*   **Limitations:** Not given in the sources. [S1]
*   **When to use:** Not given in the sources. [S1]
*   **When not to use:** Not given in the sources. [S1]

## Background
The method originates from work on Gaussian Splatting with depth and normal priors, specifically DN-Splatter [S1]. The total variation variant is associated with GS-IR [S2], which applies similar principles to inverse rendering for reflectance decomposition [S2].

## Sources
[S1] DN-Splatter: Depth and Normal Priors for Gaussian Splatting and Meshing, https://doi.org/10.1109/wacv61041.2025.00241, fetched 2026-10-26
[S2] GS-IR: 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.1109/cvpr52733.2024.02045, fetched 2026-10-26
