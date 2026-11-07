# Training Objective for Fine Representation

Also called: Fine-stage Training Configuration

## Overview
This configuration defines the loss function and optimization strategy for the fine-stage of a two-stage radiance field optimization pipeline. It builds on [Training Configuration](2332-training-configuration.md) by specifying how to handle the second stage of training, typically following a coarse-stage initialization. The primary distinction from the coarse stage is the reduction of regularization weights to prioritize photometric accuracy and geometric detail over structural priors. It is distinct from [Training Loop Setup](1522-training-loop-setup.md) which handles the loop infrastructure, and [Regularization and Priors](6281-regularization-and-priors.md) which defines the specific prior terms.

## How it works
The fine-stage training objective minimizes a composite loss function $\mathcal{L}_{fine}$ composed of photometric reconstruction loss and regularization terms.

**Inputs and Outputs:**
- **Inputs:** Rendered images $\hat{I}$, ground truth images $I_{gt}$, accumulated alpha $\alpha$, density field $V(\rho)$, and scene geometry parameters.
- **Outputs:** Scalar loss value $\mathcal{L}_{fine}$ for backpropagation.

**Data Structures:**
- Loss weights are stored as scalar hyperparameters.
- Regularization terms (e.g., density smoothness, sparsity) are computed per-sample or per-batch.

**Steps:**
1. **Compute Photometric Loss:** Calculate the pixel-wise difference between the rendered image $\hat{I}$ and the ground truth $I_{gt}$. Commonly, this is the L1 or L2 loss:
   $$ \mathcal{L}_{rgb} = \sum_{p} \| \hat{I}_p - I_{gt, p} \| $$
2. **Compute Regularization Losses:** Calculate regularization terms $\mathcal{L}_{reg}$ based on the density field $V(\rho)$ and accumulated alpha. This may include sparsity constraints or smoothness penalties.
3. **Apply Fine-Stage Weights:** Scale the regularization losses by a smaller weight factor $w_{reg}^{fine}$ compared to the coarse stage weight $w_{reg}^{coarse}$. The photometric loss weight $w_{rgb}$ remains typically unchanged or is adjusted slightly.
   $$ \mathcal{L}_{fine} = w_{rgb} \mathcal{L}_{rgb} + w_{reg}^{fine} \mathcal{L}_{reg} $$
   where $w_{reg}^{fine} < w_{reg}^{coarse}$.
4. **Backpropagate:** Compute gradients of $\mathcal{L}_{fine}$ with respect to the fine-stage network parameters and optimize.

**Parameters:**
- $w_{reg}^{fine}$: Weight for regularization losses in the fine stage. Typical value: Not given in the sources, but empirically smaller than coarse stage.
- $w_{rgb}$: Weight for photometric loss. Typical value: 1.0.

**Edge Cases and Pitfalls:**
- **Over-regularization:** If $w_{reg}^{fine}$ is not reduced sufficiently, the model may fail to capture high-frequency details.
- **Precision:** Ensure that the reduction in regularization weight does not lead to instability in the density field, particularly in unknown space.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_fine_stage_loss(rendered_image, ground_truth_image, density_field, alpha, w_rgb=1.0, w_reg_fine=0.1):
    """
    Computes the fine-stage training loss.
    
    Args:
        rendered_image: Rendered RGB image.
        ground_truth_image: Ground truth RGB image.
        density_field: Density field V(rho).
        alpha: Accumulated alpha values.
        w_rgb: Weight for photometric loss.
        w_reg_fine: Weight for regularization losses in fine stage.
        
    Returns:
        total_loss: Scalar loss value.
    """
    # Step 1: Compute photometric loss (L1 loss)
    rgb_loss = torch.mean(torch.abs(rendered_image - ground_truth_image))
    
    # Step 2: Compute regularization losses
    # Example: Sparsity regularization on density
    reg_loss = torch.mean(density_field)
    
    # Step 3: Apply fine-stage weights
    # Note: w_reg_fine is typically smaller than coarse stage weight
    total_loss = w_rgb * rgb_loss + w_reg_fine * reg_loss
    
    return total_loss
```

## Subtopics

### StyleGAN2-based Discriminator Integration
Adapts a StyleGAN2 discriminator to evaluate NeRF-rendered images, configured for 64x64 RGB images with specific channel and resolution parameters. Instantiate discriminator only if `adv_loss_mult > 0.0`. Configure discriminator with empty `block_kwargs` and `mapping_kwargs`, and `mbstd_group_size=4` in epilogue. Set base channels to 16384, max channels to 512, FP16 resolution steps to 4, and convolution clamp to 256. Set image channels to 3, class conditioning dimension to 0, and resolution to 64. Parameters: `mbstd_group_size` (default: 4) for minibatch standard deviation layer; `conv_clamp` (default: 256) for maximum absolute value for convolution outputs to stabilize training [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Similar to coarse stage, but potentially faster convergence due to reduced regularization overhead.
- **Memory Complexity:** Same as coarse stage, as the network architecture remains unchanged.
- **Strengths:** Improves high-frequency detail reconstruction by focusing on photometric accuracy.
- **Limitations:** May be prone to overfitting if regularization is too weak.
- **When to Use:** After coarse stage convergence, when fine details are needed.
- **When Not to Use:** If the coarse stage already captures sufficient detail or if computational resources are limited.

## Background
Originated in the context of two-stage NeRF optimization methods, such as Direct Voxel Grid Optimization [S1], where a coarse representation is first optimized followed by a fine representation. The approach was empirically found to yield better quality by reducing regularization weights in the fine stage.

## Sources
[S1] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S2] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
