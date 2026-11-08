# Fine-tuning
Also called: 3DGS Fine-tuning, Gaussian Splatting Fine-tuning
## Overview
Fine-tuning is the optimization phase in 3D Gaussian Splatting pipelines that utilize pretrained geometric or color priors, where initial Gaussian primitives are refined to achieve photorealism [S1]. It distinguishes itself from standard training by starting with a dense, redundant set of primitives generated via a feed-forward process, then actively reducing this count through pruning while optimizing all geometry and color attributes [S1]. This approach leverages pretrained weights to accelerate convergence and reduce memory consumption compared to training from scratch [S1].
## How it works
**Inputs and Outputs:**
*   **Input:** A set of initial 3D Gaussian primitives $\{\mathcal{G}_i\}$ with geometry (mean, covariance, scale) and color attributes (SH coefficients) predicted by a feed-forward network [S1].
*   **Output:** A refined set of Gaussians with optimized attributes and reduced cardinality [S1].

**Data Structures:**
*   **Gaussian Buffer:** Stores mean position $\mu$, covariance $\Sigma$ (or scale/rotation), opacity $\alpha$, and spherical harmonic coefficients $c_{lm}$ for each Gaussian [S1, Parent].
*   **Capacity:** Pre-allocated to handle initial dense set plus potential splits, though fine-tuning specifically targets reduction [Parent].

**Numbered Steps:**
1.  **Initialization:** Predict initial 3D Gaussian primitives via a direct feed-forward process where both geometry and color attributes are generated [S1].
2.  **Optimization Loop:** Optimize all geometry and color attributes directly using the vanilla 3DGS differentiable rendering pipeline [S1].
    *   **Loss Function:** Minimize a combination of photometric loss and regularization terms, such as:
        $$ \mathcal{L} = \mathcal{L}_{\text{ssim}} + \lambda_e \mathcal{L}_{\text{entropy}} $$
        where $\mathcal{L}_{\text{ssim}}$ is the structural similarity loss and $\mathcal{L}_{\text{entropy}}$ is an entropy regularizer [S1].
    *   **Optimizer:** Use Adam optimizer with a learning rate of $1 \times 10^{-3}$ [S1].
3.  **Densification Control:** Enable growing (splitting/cloning) and pruning of 3D primitives during fine-tuning [S1].
    *   **Pruning:** Remove redundant Gaussians to significantly reduce the total number of primitives [S1].
    *   **Growing:** Apply standard densification strategies (split/clone) based on gradient norms and screen size to recover detail where the initial prediction was insufficient [S1, Parent].
4.  **Convergence:** Continue until the model achieves photorealism, leveraging the pretrained priors for faster convergence [S1].

**Parameters:**
*   **Learning Rate:** $1 \times 10^{-3}$ for Adam optimizer [S1].
*   **Loss Weight:** $\lambda_e$ for entropy term (value not specified in sources) [S1].
*   **Library:** `gsplat` for Gaussian rasterization [S1].

**Edge Cases and Pitfalls:**
*   **Redundancy:** The initial feed-forward prediction often produces redundant Gaussians, requiring aggressive pruning to avoid excessive memory usage [S1].
*   **Precision:** Standard 3DGS optimization applies; ensure gradient stability during the transition from feed-forward initialization to differentiable optimization [S1].
## Subtopics

### [Exponentially Ascending Gradient Threshold](9684-exponentially-ascending-gradient-threshold.md)
A densification control mechanism that uses an exponentially ascending gradient threshold to regulate splitting and cloning, solving the trade-off between early-stage convergence speed and late-stage overfitting.

### [Hard Gaussian Splatting](10590-hard-gaussian-splatting.md)
Addresses artifacts caused by averaged view-space gradients by identifying "hard Gaussians" via multi-view significant positional gradients and rendering errors to recover complete 3D scenes.

## Operating characteristics
*   **Time Complexity:** Faster convergence compared to training from scratch due to pretrained priors [S1].
*   **Memory Complexity:** Significantly reduced memory consumption compared to the initial dense feed-forward prediction due to pruning [S1].
*   **Strengths:** Achieves photorealism on par with or surpassing other methods; leverages powerful pretrained weights [S1].
*   **Limitations:** Relies on the quality of the initial feed-forward prediction; redundant initial primitives require effective pruning strategies [S1].
*   **When to Use:** When pretrained geometric or color priors are available and efficient, high-fidelity reconstruction is required [S1].
*   **When Not to Use:** When no pretrained model is available or when the scene distribution differs significantly from the training data of the pretrained model [S1].
## Background
Fine-tuning in the context of 3D Gaussian Splatting was introduced in EVolSplat for efficient volume-based Gaussian Splatting for urban view synthesis [S1]. It builds on the vanilla 3DGS optimization framework [15] but integrates a feed-forward initialization step [S1]. The method was demonstrated to accelerate network training and enable faster convergence compared to other test-time optimization methods [S1].
## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
