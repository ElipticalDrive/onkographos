# Mip-NeRF 360 Optimization Setup

Also called: Mip-NeRF 360 Training Procedure

## Overview
This article defines the specific optimization configuration, network architectures, and loss functions used to train the Mip-NeRF 360 model. It solves the problem of stabilizing the training of unbounded, anti-aliased radiance fields by combining a multi-stage proposal sampling strategy with a robust reconstruction loss. It sets itself apart from standard NeRF training by using a two-stage proposal network to guide sampling and by employing a Charbonnier loss for reconstruction stability [S1].

## How it works

### Inputs and Outputs
*   **Inputs:** A dataset of images with known camera poses (estimated via COLMAP) [S1].
*   **Outputs:** Trained weights for the proposal MLP ($\Theta_{\text{prop}}$) and the NeRF MLP ($\Theta_{\text{NeRF}}$).

### Data Structures
The method utilizes two Multi-Layer Perceptrons (MLPs):
1.  **Proposal MLP:**
    *   Architecture: 4 layers, 256 hidden units per layer.
    *   Activations: ReLU for internal layers, Softplus for the final density output $\tau$ [S1].
2.  **NeRF MLP:**
    *   Architecture: 8 layers, 1024 hidden units per layer.
    *   Activations: ReLU for internal layers, Softplus for the final density output $\tau$ [S1].

### Training Steps
The optimization proceeds in iterations, processing batches of rays. Each iteration involves the following stages:

1.  **Proposal Sampling (Stage 0):**
    *   Evaluate the proposal MLP using 64 samples along each ray.
    *   Produce the first histogram $(\hat{s}_0, \hat{w}_0)$, where $\hat{s}_0$ represents interval endpoints and $\hat{w}_0$ represents weights [S1].
2.  **Proposal Resampling (Stage 1):**
    *   Resample the proposal based on $(\hat{s}_0, \hat{w}_0)$.
    *   Evaluate the proposal MLP again using 64 samples on the resampled intervals.
    *   Produce the second histogram $(\hat{s}_1, \hat{w}_1)$ [S1].
3.  **NeRF Evaluation:**
    *   Resample based on $(\hat{s}_1, \hat{w}_1)$.
    *   Evaluate the NeRF MLP using 32 samples.
    *   Produce the final histogram $(s, w)$ [S1].
4.  **Loss Computation:**
    *   Compute the total loss $\mathcal{L}$ as defined below.
    *   Apply stop-gradient to the proposal loss terms to decouple the optimization of $\Theta_{\text{prop}}$ from $\Theta_{\text{NeRF}}$ [S1].
5.  **Optimization Step:**
    *   Update weights using the Adam optimizer [S1].

### Formulas
The total loss function averaged over all rays in a batch is:

$$
\mathcal{L} = \mathcal{L}_{\text{recon}}(\mathbf{C}(\mathbf{t}), \mathbf{C}^*) + \lambda \mathcal{L}_{\text{dist}}(\mathbf{s}, \mathbf{w}) + \sum_{k=0}^1 \mathcal{L}_{\text{prop}}(\mathbf{s}, \mathbf{w}, \hat{\mathbf{s}}^k, \hat{\mathbf{w}}^k)
$$

Where:
*   $\mathcal{L}_{\text{recon}}$ is the Charbonnier loss: $\sqrt{(x - x^*)^2 + \epsilon^2}$ with $\epsilon = 0.001$ [S1].
*   $\mathcal{L}_{\text{dist}}$ is a regularizer minimizing the weighted size of individual intervals [S1].
*   $\lambda$ is a hyperparameter balancing reconstruction and regularization, set to $0.01$ [S1].
*   $\mathcal{L}_{\text{prop}}$ is the proposal supervision loss. The stop-gradient is applied to this term [S1].

### Parameters
*   **Optimizer:** Adam [S1].
    *   $\beta_1 = 0.9$
    *   $\beta_2 = 0.999$
    *   $\epsilon = 10^{-6}$
*   **Learning Rate Schedule:** Annealed log-linearly from $2 \times 10^{-3}$ to $2 \times 10^{-5}$ [S1].
    *   Warm-up phase: 512 iterations [S1].
*   **Gradient Clipping:** Norm of $10^{-3}$ [S1].
*   **Batch Size:** $2^{14}$ rays [S1].
*   **Total Iterations:** 250,000 [S1].

### Edge Cases and Pitfalls
*   **Z-aliasing:** The standard Mip-NeRF 360 interlevel loss can cause the proposal MLP to learn non-smooth mappings, leading to rays "skipping" scene content. This is particularly visible when using high-capacity backends like iNGP [S2].
*   **Proposal Independence:** The stop-gradient on $\mathcal{L}_{\text{prop}}$ is critical. Without it, the optimization of the proposal network would be coupled with the NeRF network, potentially destabilizing training [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def train_mipnerf_360(dataset, iterations=250000):
    # Initialize networks
    # Proposal MLP: 4 layers, 256 hidden, ReLU internal, Softplus density
    prop_mlp = initialize_mlp(layers=4, hidden=256, activation='relu', density_activation='softplus')
    
    # NeRF MLP: 8 layers, 1024 hidden, ReLU internal, Softplus density
    nerf_mlp = initialize_mlp(layers=8, hidden=1024, activation='relu', density_activation='softplus')
    
    # Optimizer setup
    optimizer = Adam(lr=2e-3, beta1=0.9, beta2=0.999, eps=1e-6)
    
    # Hyperparameters
    lambda_dist = 0.01
    epsilon_charb = 0.001
    gradient_clip_norm = 1e-3
    warmup_iters = 512
    lr_start = 2e-3
    lr_end = 2e-5
    
    for i in range(iterations):
        # 1. Learning Rate Schedule
        if i < warmup_iters:
            lr = lr_start * (i / warmup_iters)
        else:
            # Log-linear annealing
            progress = (i - warmup_iters) / (iterations - warmup_iters)
            lr = lr_start * (lr_end / lr_start) ** progress
            
        optimizer.set_learning_rate(lr)
        
        # 2. Sample Batch
        batch_rays = dataset.sample_batch(batch_size=2**14)
        
        # 3. Proposal Stage 0
        # 64 samples per ray
        samples_0 = sample_proposal(prop_mlp, batch_rays, num_samples=64)
        hist_0 = compute_histogram(samples_0, prop_mlp) # (s_hat_0, w_hat_0)
        
        # 4. Proposal Stage 1
        # Resample based on hist_0
        samples_1 = resample_proposal(samples_0, hist_0)
        # 64 samples per ray
        samples_1 = sample_proposal(prop_mlp, samples_1, num_samples=64)
        hist_1 = compute_histogram(samples_1, prop_mlp) # (s_hat_1, w_hat_1)
        
        # 5. NeRF Evaluation
        # Resample based on hist_1
        samples_nerf = resample_proposal(samples_1, hist_1)
        # 32 samples per ray
        samples_nerf = sample_proposal(nerf_mlp, samples_nerf, num_samples=32)
        hist_nerf = compute_histogram(samples_nerf, nerf_mlp) # (s, w)
        
        # 6. Compute Loss
        # Reconstruct color C(t) from hist_nerf
        C_recon = reconstruct_color(hist_nerf, nerf_mlp)
        C_target = batch_rays.target_color
        
        # Charbonnier Loss
        recon_loss = charbonnier_loss(C_recon, C_target, epsilon=epsilon_charb)
        
        # Distance Regularizer
        dist_loss = compute_distance_regularizer(hist_nerf)
        
        # Proposal Loss (with stop-gradient)
        # Note: The sources indicate L_prop is summed for k=0 and k=1
        # The exact implementation of L_prop is not fully detailed in the provided text,
        # but it compares (s, w) against (s_hat_k, w_hat_k).
        # We assume a standard histogram comparison or MSE on the resampled values.
        # The stop-gradient is applied to the proposal loss terms.
        prop_loss_0 = compute_proposal_loss(hist_nerf, hist_0)
        prop_loss_1 = compute_proposal_loss(hist_nerf, hist_1)
        
        # Apply stop-gradient to proposal losses to decouple optimization
        prop_loss_0 = stop_gradient(prop_loss_0)
        prop_loss_1 = stop_gradient(prop_loss_1)
        
        total_loss = recon_loss + lambda_dist * dist_loss + prop_loss_0 + prop_loss_1
        
        # 7. Backpropagation and Update
        gradients = compute_gradients(total_loss, [prop_mlp, nerf_mlp])
        
        # Gradient Clipping
        gradients = clip_gradients(gradients, max_norm=gradient_clip_norm)
        
        optimizer.step([prop_mlp, nerf_mlp], gradients)
```

## Subtopics

### [Anti-Aliased Interlevel Loss](3457-anti-aliased-interlevel-loss.md)
The Anti-Aliased Interlevel Loss is a proposal supervision loss function that measures the distance between a NeRF histogram and a proposal histogram by blurring the NeRF histogram into a piecewise-linear spline and resampling it into the proposal's interval endpoints. Unlike the piecewise constant loss used in mip-NeRF 360, which suffers from aliasing when NeRF histogram bins are narrow or translate relative to the proposal, this method produces a smooth loss surface that prevents aliasing artifacts during training.

### [Isotropic Gaussian Scale Contraction](5407-isotropic-gaussian-scale-contraction.md)
Isotropic Gaussian Scale Contraction is a technique for efficiently adapting the scale of isotropic Gaussians when a scene contraction function $C(\mathbf{x})$ is applied. It solves the problem of maintaining correct anti-aliasing properties in contracted coordinate spaces without the computational overhead of general multivariate Gaussian transformations. It sets itself apart from the standard mip-NeRF 360 approach by linearizing the contraction around each Gaussian center to derive an isotropic scale from the Jacobian determinant, rather than contracting a full covariance matrix.

### Smooth Proposal Supervision
Smooth Proposal Supervision is a method to resolve 'z-aliasing' artifacts in proposal networks by replacing the discrete interlevel loss with a continuous, smooth loss function that penalizes partial overlaps between proposal and NeRF histograms [S2]. It identifies z-aliasing artifacts where the proposal MLP learns a non-smooth mapping, causing rays to 'skip' scene content, especially when using iNGP backends for rapid optimization [S2]. The original Mip-NeRF 360 interlevel loss assigns equivalent loss to NeRF and proposal histogram bins regardless of overlap extent, penalizing only complete failures to overlap, leading to non-smooth density maps [S2]. The new loss prevents the proposal network from learning non-smooth mappings that occur with the previous loss, particularly when the proposal network has high capacity (e.g., iNGP backend) [S2].

### MipNeRF-360 Depth Rendering
MipNeRF-360 Depth Rendering is an unbounded extension of MipNeRF that uses a contract function to parameterize 3D Euclidean space within a ball of radius 2, allowing direct rendering of images and depth maps in the contracted space [S3]. It renders images and depth maps in the contracted space using standard NeRF equations (Eq. 1 and Eq. 2) [S3].

## Operating characteristics
*   **Time Complexity:** The method uses a multi-stage sampling strategy (2 proposal stages + 1 NeRF stage), which increases the number of forward passes per ray compared to single-stage NeRF, but improves sampling efficiency by focusing samples on high-density regions.
*   **Memory Complexity:** The memory usage is determined by the MLP sizes (Proposal: 4x256, NeRF: 8x1024) and the batch size ($2^{14}$ rays). The proposal histograms and intermediate samples are stored in memory during the forward pass.
*   **Performance:** The method achieves stable optimization and high-quality rendering on complex "in the wild" scenes (5 outdoors, 4 indoors) [S1]. The use of Charbonnier loss provides slightly more stable optimization than Mean Squared Error [S1].
*   **Strengths:** Robust to unbounded scenes, anti-aliased rendering, stable training with large capacity networks (when using smooth supervision) [S1, S2].
*   **Limitations:** The standard interlevel loss can cause z-aliasing artifacts, especially with high-capacity backends like iNGP [S2]. Requires careful tuning of the learning rate schedule and gradient clipping [S1].

## Background
Mip-NeRF 360 was introduced as an unbounded extension of mip-NeRF, proposing the use of a contract function to parameterize 3D Euclidean space within a ball of radius 2 [S3]. The optimization setup described here was detailed in the paper "Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields" [S1]. The method was evaluated on a novel dataset of 9 scenes (5 outdoors, 4 indoors) with complex central objects and detailed backgrounds [S1]. Camera poses were estimated using COLMAP [S1]. The approach significantly improves speed and rendering quality compared to prior strategies of learning one or multiple NeRFs supervised using image reconstruction [S2].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S2] Zip-NeRF: Anti-Aliased Grid-Based Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01804, fetched 2026-10-26
[S3] Digging into Depth Priors for Outdoor Neural Radiance Fields, https://doi.org/10.1145/3581783.3612306, fetched 2026-10-30
