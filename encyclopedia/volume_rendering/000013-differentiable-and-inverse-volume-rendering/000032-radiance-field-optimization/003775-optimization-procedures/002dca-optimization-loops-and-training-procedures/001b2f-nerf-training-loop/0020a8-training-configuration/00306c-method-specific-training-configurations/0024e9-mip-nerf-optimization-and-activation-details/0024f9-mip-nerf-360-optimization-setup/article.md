# Mip-NeRF 360 Optimization Setup

Also called: Mip-NeRF 360 Training Procedure

## Overview
This article specifies the training configuration, network architecture, and loss functions for the Mip-NeRF 360 method, which extends mip-NeRF to unbounded, "in-the-wild" scenes [S1]. It solves the problem of training stability and rendering quality in complex environments with large depth ranges and varying lighting by introducing a two-stage proposal sampling strategy and a specific regularization scheme [S1]. It differs from standard NeRF training by using a proposal network to guide sample placement and from basic mip-NeRF by employing a Charbonnier reconstruction loss and a log-linear learning rate schedule to handle the scale and complexity of outdoor/indoor hybrid scenes [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera poses (estimated via COLMAP), images, and intrinsic parameters.
*   **Outputs:** Trained proposal MLP parameters $\Theta_{\text{prop}}$ and NeRF MLP parameters $\Theta_{\text{NeRF}}$.

**Data Structures:**
1.  **Proposal MLP:**
    *   Architecture: 4 layers, 256 hidden units per layer.
    *   Activations: ReLU for internal layers, softplus for density output $\tau$.
2.  **NeRF MLP:**
    *   Architecture: 8 layers, 1024 hidden units per layer.
    *   Activations: ReLU for internal layers, softplus for density output $\tau$.
3.  **Sampling Intervals:**
    *   Proposal samples produce intervals $(\hat{s}_k, \hat{w}_k)$ for $k \in \{0, 1\}$.
    *   NeRF samples produce intervals $(s, w)$.
    *   Each interval is defined by a midpoint and a width (size).

**Training Steps:**
1.  **Initialization:** Initialize the Proposal MLP and NeRF MLP with the architectures described above.
2.  **Proposal Sampling (Stage 0):** Evaluate the Proposal MLP with 64 samples along each ray to produce the first set of weighted intervals $(\hat{s}_0, \hat{w}_0)$.
3.  **Proposal Sampling (Stage 1):** Resample based on $(\hat{s}_0, \hat{w}_0)$ and evaluate the Proposal MLP again with 64 samples to produce the second set of weighted intervals $(\hat{s}_1, \hat{w}_1)$.
4.  **NeRF Sampling:** Resample based on $(\hat{s}_1, \hat{w}_1)$ and evaluate the NeRF MLP with 32 samples to produce the final weighted intervals $(s, w)$.
5.  **Rendering:** Compute the rendered color $\mathbf{C}(\mathbf{t})$ using the final intervals $(s, w)$ and the NeRF MLP's color predictions.
6.  **Loss Computation:** Compute the total loss $\mathcal{L}$ as defined below.
7.  **Optimization:** Update parameters using Adam optimizer.

**Formulas:**
The total loss $\mathcal{L}$ is averaged over all rays in a batch:
$$ \mathcal{L} = \mathcal{L}_{\text{recon}}(\mathbf{C}(\mathbf{t}), \mathbf{C}^*) + \lambda \mathcal{L}_{\text{dist}}(\mathbf{s}, \mathbf{w}) + \sum_{k=0}^1 \mathcal{L}_{\text{prop}}(\mathbf{s}, \mathbf{w}, \hat{\mathbf{s}}^k, \hat{\mathbf{w}}^k) $$

Where:
*   $\mathcal{L}_{\text{recon}}$ is the Charbonnier loss: $\sqrt{(x - x^*)^2 + \epsilon^2}$ with $\epsilon = 0.001$.
*   $\mathcal{L}_{\text{dist}}$ is a regularization term minimizing the weighted size of individual intervals.
*   $\lambda = 0.01$ balances the reconstruction and regularization terms.
*   $\mathcal{L}_{\text{prop}}$ is the proposal supervision loss. A stop-gradient operation is applied to $\mathcal{L}_{\text{prop}}$ with respect to the NeRF parameters, making the optimization of $\Theta_{\text{prop}}$ independent of $\Theta_{\text{NeRF}}$. Consequently, no scaling hyperparameter is needed for $\mathcal{L}_{\text{prop}}$ [S1].

**Parameters:**
*   **Optimizer:** Adam [S1].
    *   $\beta_1 = 0.9$
    *   $\beta_2 = 0.999$
    *   $\epsilon = 10^{-6}$
*   **Learning Rate Schedule:** Annealed log-linearly from $2 \times 10^{-3}$ to $2 \times 10^{-5}$ [S1].
    *   Warm-up phase: 512 iterations.
*   **Gradient Clipping:** Norm of $10^{-3}$ [S1].
*   **Training Iterations:** 250,000 [S1].
*   **Batch Size:** $2^{14}$ rays [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_mipnerf_360():
    """
    Initializes the Proposal and NeRF MLPs and optimizer state.
    """
    # Proposal MLP: 4 layers, 256 hidden units, ReLU internal, softplus density
    proposal_mlp = MLP(layers=4, hidden_units=256, activation='relu', density_activation='softplus')
    
    # NeRF MLP: 8 layers, 1024 hidden units, ReLU internal, softplus density
    nerf_mlp = MLP(layers=8, hidden_units=1024, activation='relu', density_activation='softplus')
    
    # Adam Optimizer parameters
    beta1 = 0.9
    beta2 = 0.999
    eps = 1e-6
    lr_start = 2e-3
    lr_end = 2e-5
    warmup_iters = 512
    total_iters = 250000
    grad_clip_norm = 1e-3
    lambda_dist = 0.01
    charbonnier_eps = 0.001
    
    optimizer = Adam(params=[proposal_mlp.parameters(), nerf_mlp.parameters()], 
                     lr=lr_start, beta1=beta1, beta2=beta2, eps=eps)
    
    return proposal_mlp, nerf_mlp, optimizer, {
        'beta1': beta1, 'beta2': beta2, 'eps': eps,
        'lr_start': lr_start, 'lr_end': lr_end, 'warmup_iters': warmup_iters,
        'total_iters': total_iters, 'grad_clip_norm': grad_clip_norm,
        'lambda_dist': lambda_dist, 'charbonnier_eps': charbonnier_eps
    }

def compute_loss(rendered_color, ground_truth_color, s, w, s_hat_0, w_hat_0, s_hat_1, w_hat_1, params):
    """
    Computes the total loss for Mip-NeRF 360.
    """
    # Reconstruction Loss (Charbonnier)
    diff = rendered_color - ground_truth_color
    recon_loss = np.sqrt(diff**2 + params['charbonnier_eps']**2)
    
    # Distance Regularization Loss
    # Minimizes weighted size of intervals
    dist_loss = compute_weighted_interval_size(s, w)
    
    # Proposal Loss
    # Stop-gradient on NeRF parameters (s, w) for the proposal loss calculation
    # L_prop is computed between (s, w) and (s_hat_k, w_hat_k)
    # Note: The exact formulation of L_prop is not fully detailed in the source text 
    # beyond the stop-gradient and summation, but it supervises the proposal histograms.
    prop_loss_0 = compute_proposal_loss(s, w, s_hat_0, w_hat_0)
    prop_loss_1 = compute_proposal_loss(s, w, s_hat_1, w_hat_1)
    
    total_loss = recon_loss + params['lambda_dist'] * dist_loss + prop_loss_0 + prop_loss_1
    
    return total_loss

def train_step(batch, proposal_mlp, nerf_mlp, optimizer, params):
    """
    Performs one training step.
    """
    rays = batch.rays
    ground_truth_colors = batch.colors
    
    # Stage 0: Proposal Sampling
    # 64 samples
    s_hat_0, w_hat_0 = sample_proposal(proposal_mlp, rays, num_samples=64)
    
    # Stage 1: Proposal Resampling and Evaluation
    # 64 samples
    s_hat_1, w_hat_1 = sample_proposal(proposal_mlp, rays, num_samples=64, prev_intervals=(s_hat_0, w_hat_0))
    
    # NeRF Sampling
    # 32 samples
    s, w = sample_nerf(proposal_mlp, rays, num_samples=32, prev_intervals=(s_hat_1, w_hat_1))
    
    # Render
    rendered_colors = render_nerf(nerf_mlp, s, w)
    
    # Compute Loss
    loss = compute_loss(rendered_colors, ground_truth_colors, s, w, s_hat_0, w_hat_0, s_hat_1, w_hat_1, params)
    
    # Backpropagation
    gradients = compute_gradients(loss, [proposal_mlp, nerf_mlp])
    
    # Gradient Clipping
    grad_norm = np.linalg.norm(gradients)
    if grad_norm > params['grad_clip_norm']:
        gradients = [g * (params['grad_clip_norm'] / grad_norm) for g in gradients]
        
    # Update Learning Rate
    current_iter = optimizer.iteration
    if current_iter < params['warmup_iters']:
        lr = params['lr_start'] * (current_iter / params['warmup_iters'])
    else:
        # Log-linear annealing
        progress = (current_iter - params['warmup_iters']) / (params['total_iters'] - params['warmup_iters'])
        lr = params['lr_start'] * (params['lr_end'] / params['lr_start']) ** progress
        
    optimizer.set_lr(lr)
    optimizer.step(gradients)
    
    return loss

def sample_proposal(mlp, rays, num_samples, prev_intervals=None):
    """
    Samples intervals using the proposal MLP.
    If prev_intervals is provided, resamples based on them.
    """
    # Implementation details of sampling/resampling are assumed to follow standard 
    # inverse CDF sampling based on the MLP's density output.
    # The source specifies 64 samples for both proposal stages.
    pass

def sample_nerf(proposal_mlp, rays, num_samples, prev_intervals):
    """
    Samples intervals using the proposal MLP's output for resampling, 
    then evaluates NeRF MLP.
    """
    # Implementation details of sampling/resampling are assumed to follow standard 
    # inverse CDF sampling based on the proposal MLP's density output.
    # The source specifies 32 samples for the NeRF stage.
    pass

def compute_weighted_interval_size(s, w):
    """
    Computes the regularization term minimizing weighted size of intervals.
    """
    # Source mentions minimizing weighted size of each individual interval.
    # Exact formula not provided, but typically involves sum of widths.
    pass

def compute_proposal_loss(s, w, s_hat, w_hat):
    """
    Computes the interlevel loss between NeRF and Proposal histograms.
    """
    # Source mentions L_prop supervises the proposal network.
    # Stop-gradient is applied to NeRF parameters.
    pass
```

## Subtopics

### [Anti-Aliased Interlevel Loss](3457-anti-aliased-interlevel-loss.md)
The Anti-Aliased Interlevel Loss is a proposal supervision loss function that measures the distance between a NeRF histogram and a proposal histogram by blurring the NeRF histogram into a piecewise-linear spline and resampling it into the proposal's interval endpoints. Unlike the piecewise constant loss used in Mip-NeRF 360, which suffers from aliasing when NeRF histogram bins are narrow or translate relative to the proposal, this method produces a smooth loss surface that prevents aliasing artifacts during training.

### [Isotropic Gaussian Scale Contraction](5407-isotropic-gaussian-scale-contraction.md)
Isotropic Gaussian Scale Contraction is a technique for efficiently adapting the scale of isotropic Gaussians when a scene contraction function $C(\mathbf{x})$ is applied. It solves the problem of maintaining correct anti-aliasing properties in contracted coordinate spaces without the computational overhead of general multivariate Gaussian transformations. It sets itself apart from the standard Mip-NeRF 360 approach by linearizing the contraction around each Gaussian center to derive an isotropic scale from the Jacobian determinant, rather than contracting a full covariance matrix.

### AligNeRF
AligNeRF is a training strategy for Neural Radiance Fields that addresses image misalignment caused by moving objects or camera calibration errors, while simultaneously improving high-frequency detail recovery in high-resolution settings [S1]. It distinguishes itself from standard NeRF training by introducing a convolution-augmented architecture and a staged optimization process that switches from random ray sampling to patch-based sampling with an alignment-aware loss, allowing the model to leverage 2D local neighborhood information [S1].

### Smooth Proposal Supervision
Smooth Proposal Supervision is a method to resolve 'z-aliasing' artifacts in proposal networks by replacing the discrete interlevel loss with a continuous, smooth loss function that penalizes partial overlaps between proposal and NeRF histograms [S2]. It identifies z-aliasing artifacts where the proposal MLP learns a non-smooth mapping, causing rays to 'skip' scene content, especially when using iNGP backends for rapid optimization [S2]. The original Mip-NeRF 360 interlevel loss assigns equivalent loss to NeRF and proposal histogram bins regardless of overlap extent, penalizing only complete failures to overlap, leading to non-smooth density maps [S2]. The new loss prevents the proposal network from learning non-smooth mappings that occur with the previous loss, particularly when the proposal network has high capacity (e.g., iNGP backend) [S2].

## Operating characteristics
*   **Time Complexity:** The method uses a two-stage proposal sampling strategy (64 samples each) followed by a final NeRF evaluation (32 samples). This increases the number of MLP evaluations per ray compared to single-stage NeRF but improves sampling efficiency by focusing samples on high-density regions.
*   **Memory Complexity:** Requires storing parameters for two MLPs: a smaller Proposal MLP (4x256) and a larger NeRF MLP (8x1024).
*   **Measured Performance:** Evaluated on 9 scenes (5 outdoors, 4 indoors) with complex central objects and detailed backgrounds [S1].
*   **Strengths:** Improved stability and rendering quality in unbounded, "in-the-wild" scenes compared to prior strategies [S1]. The use of Charbonnier loss and log-linear learning rate schedule aids convergence [S1].
*   **Limitations:** The proposal network can learn non-smooth mappings leading to "z-aliasing" artifacts (rays skipping content), especially with high-capacity backends like iNGP [S2].
*   **When to use:** Suitable for training NeRFs on complex, unbounded scenes with varying scales and lighting conditions where standard NeRF training is unstable or produces poor quality [S1].
*   **When not to use:** Not ideal for scenes where z-aliasing artifacts are critical and cannot be mitigated by alternative loss functions like Smooth Proposal Supervision [S2].

## Background
Mip-NeRF 360 was introduced in the paper "Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields" [S1]. It builds on the original Mip-NeRF method [S1]. The method was designed to handle "in-the-wild" photo collections, specifically focusing on scale and complexity rather than just photometric variation [S1]. Camera poses are estimated using COLMAP [S1].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
