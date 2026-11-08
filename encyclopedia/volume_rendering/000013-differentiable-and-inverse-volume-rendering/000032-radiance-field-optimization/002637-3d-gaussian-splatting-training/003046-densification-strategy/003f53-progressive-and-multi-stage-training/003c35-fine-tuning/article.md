# Fine-tuning
Also called: 3DGS Fine-tuning, Gaussian Splatting Fine-tuning
## Overview
Fine-tuning is a post-initialization optimization phase in 3D Gaussian Splatting where a pre-generated set of Gaussian primitives is refined to achieve photorealistic reconstruction. It solves the problem of initializing with redundant or low-fidelity primitives by enabling the growing and pruning of 3D primitives during the optimization process [S1]. It sets itself apart from pure initialization methods by actively managing the Gaussian population size and density through densification, leveraging pretrained priors to accelerate convergence and reduce memory consumption compared to training from scratch [S1].
## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussian primitives $\mathcal{G}_i$ initialized via a direct feed-forward process, where both geometry (mean, covariance, scale) and color attributes (SH coefficients) are pre-generated [S1].
*   **Output:** Optimized geometry and color attributes for the surviving Gaussian primitives [S1].

**Data Structures:**
*   **Gaussian Buffer:** Stores the set of active Gaussians. As noted in the parent [Parent], this requires GPU buffer management with pre-allocation (typically 3x initial count) and compaction logic to handle pruning [Parent][S4].

**Numbered Steps:**
1.  **Initialization:** Predict a set of 3D Gaussian primitives for initialization via a direct feed-forward process. Both geometry and color attributes are generated simultaneously [S1].
2.  **Optimization Loop:** Optimize all geometry and color attributes directly following the vanilla 3DGS differentiable rendering pipeline [S1].
3.  **Densification Control:** Enable growing (splitting/cloning) and pruning of 3D primitives during fine-tuning [S1]. This step is critical because the initial feed-forward Gaussians are often redundant, and the number of Gaussians is significantly reduced during this phase to maintain high fidelity while lowering memory usage [S1].
4.  **Loss Computation:** Compute the total loss $\mathcal{L}$, which typically includes SSIM and entropy terms as per the EVolSplat formulation: $\mathcal{L} = \mathcal{L}_{\text{ssim}} + \lambda_e \mathcal{L}_{\text{entropy}}$ [S1].
5.  **Parameter Update:** Update parameters using an Adam optimizer [S1].

**Formulas:**
*   **Total Loss:** $\mathcal{L} = \mathcal{L}_{\text{ssim}} + \lambda_e \mathcal{L}_{\text{entropy}}$ [S1].
    *   $\mathcal{L}_{\text{ssim}}$: Structural Similarity Index loss.
    *   $\mathcal{L}_{\text{entropy}}$: Entropy loss term.
    *   $\lambda_e$: Weight for the entropy loss.

**Parameters:**
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** $1 \times 10^{-3}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fine_tune_gaussians(initial_gaussians, images, camera_poses, lr=1e-3):
    """
    Fine-tune pre-initialized 3D Gaussians using differentiable rendering.
    """
    # Initialize optimizer with geometry and color attributes
    optimizer = Adam([initial_gaussians.geometry, initial_gaussians.color], lr=lr)
    
    # Note: Densification (split/clone/prune) is enabled during this loop
    # as per [S1] and parent principles [Parent]
    
    for step in range(max_steps):
        # 1. Render image from current Gaussians
        rendered_image = rasterize(initial_gaussians, camera_poses)
        
        # 2. Compute Loss (SSIM + Entropy)
        # Note: Exact lambda_e value not given in sources [S1]
        loss = compute_ssim_loss(rendered_image, target_image) + \
               lambda_e * compute_entropy_loss(initial_gaussians)
        
        # 3. Backpropagate and Update
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # 4. Densification Step (Triggered periodically per parent [Parent])
        if step % refine_every == 0:
            initial_gaussians = densify_and_prune(initial_gaussians, loss)
            
    return initial_gaussians
```

**Edge Cases and Pitfalls:**
*   **Redundancy:** The initial feed-forward Gaussians are redundant; failing to prune them leads to excessive memory consumption [S1].
*   **Convergence:** Pretrained priors accelerate training; skipping fine-tuning may result in slower convergence or lower fidelity [S1].

## Subtopics

### [Exponentially Ascending Gradient Threshold](9684-exponentially-ascending-gradient-threshold.md)
A densification control mechanism that uses an exponentially ascending gradient threshold to regulate splitting and cloning. It solves the trade-off between early-stage convergence speed and late-stage overfitting by dynamically adjusting sensitivity based on normalized training progress. Choose this when you need to prevent over-densification in dense scenes while maintaining fast initial convergence.

## Operating characteristics
*   **Time Complexity:** Accelerated convergence compared to training from scratch due to pretrained priors [S1].
*   **Memory Complexity:** Significantly reduced memory consumption compared to the initial feed-forward state, as the number of Gaussians is reduced during fine-tuning [S1].
*   **Strengths:** Achieves photorealism on par with or surpassing other methods; leverages powerful pretrained weights [S1].
*   **Limitations:** Relies on the quality of the initial feed-forward initialization; requires densification logic to manage primitive count [S1].
*   **When to Use:** When a coarse initialization is available and high-fidelity refinement is needed with limited memory resources [S1].
*   **When Not to Use:** When no pretrained initialization is available or when the scene geometry is too complex for the initial feed-forward pass to capture [S1].

## Background
Originating from the EVolSplat method [S1], this approach applies fine-tuning to urban view synthesis. It builds on the vanilla 3DGS optimization framework [S1] but integrates a direct feed-forward initialization step. The method demonstrates that pretrained priors can accelerate network training and enable faster convergence compared to other test-time optimization methods [S1].

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
