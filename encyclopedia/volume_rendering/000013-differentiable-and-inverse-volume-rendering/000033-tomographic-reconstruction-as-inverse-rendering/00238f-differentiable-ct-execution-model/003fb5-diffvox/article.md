# DiffVox

Also called: Differentiable Voxel-based X-ray Rendering

## Overview
DiffVox is a self-supervised framework for Cone-Beam Computed Tomography (CBCT) reconstruction that directly optimizes a discrete voxel grid representation using physics-based differentiable X-ray rendering [S1, unofficial]. It solves the sparse-view reconstruction problem by parameterizing the unknown volume as a voxel grid and using differentiable forward models (Siddon's method or trilinear interpolation) to render 2D projections, enabling gradient-based optimization of the 3D volume [S1, unofficial]. Unlike traditional iterative methods which may suffer from artifacts in sparse regimes, or neural field approaches which are computationally prohibitive, DiffVox achieves high-quality reconstructions with significantly faster runtimes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ 2D X-ray projection images (ground truth intensities $I_\mu$) with known acquisition geometry (source and detector positions). The number of views $N$ is typically sparse (e.g., 5–60 views) [S1, unofficial].
*   **Outputs:** A reconstructed 3D voxel grid $\hat{\mu}$ of linear attenuation coefficients (LACs) [S1, unofficial].

**Data Structures:**
*   **Voxel Grid $\hat{\mu}$:** A 3D tensor of dimensions $D_x \times D_y \times D_z$ representing the volume. In experiments, this is $501 \times 501 \times 501$ with isotropic spacing of 0.1 mm [S1, unofficial]. The values represent unactivated LACs.
*   **Ray Set $\mathcal{R}$:** A set of photon beams (rays) sampled from the projection images. Each ray $\vec{r}$ is defined by its origin, direction, and the corresponding ground truth intensity $I_\mu(\vec{r})$ [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Initialize the voxel grid $\hat{\mu}$ with zeros [S1, unofficial]. Alternatively, it can be initialized with an estimate from Nesterov or SIRT to reduce runtime [S1, unofficial].
2.  **Activation:** Apply the Softplus activation function to $\hat{\mu}$ to ensure non-negative LACs, as physical attenuation coefficients must be non-negative [S1, unofficial].
    $$ \mu_{\text{active}} = \text{Softplus}(\hat{\mu}, \beta) $$
    where $\beta = 20$ [S1, unofficial].
3.  **Ray Sampling:** Randomly sample a batch of rays $\mathcal{R}$ from all available projection images without replacement, using uniform probability [S1, unofficial]. The batch size is limited by GPU memory (e.g., $|\mathcal{R}| = 5.5 \times 10^5$ for Siddon's method, $1.8 \times 10^6$ for trilinear interpolation) [S1, unofficial].
4.  **Differentiable Forward Rendering:** Compute the rendered intensities $I_{\hat{\mu}}(\vec{r})$ for each ray in the batch using one of two differentiable forward models:
    *   **Siddon's Method:** Computes the exact line integral through the voxel grid according to the discrete Beer-Lambert law [S1, unofficial].
    *   **Trilinear Interpolation:** Approximates the volume density at ray points using trilinear interpolation, effectively allowing non-intersected voxels to influence the pixel intensity [S1, unofficial]. For this method, $M=500$ points are sampled per ray [S1, unofficial].
5.  **Loss Calculation:** Compute the total loss $\mathcal{L}$ using the photometric loss (L1 norm) and Total Variation (TV) regularization [S1, unofficial]:
    $$ \mathcal{L}(\hat{\mu}) = \frac{1}{|\mathcal{R}|} \sum_{\vec{r} \in \mathcal{R}} \|I_{\mu}(\vec{r}) - I_{\hat{\mu}}(\vec{r})\|_1 + \lambda_{\text{TV}} \text{TV}(\hat{\mu}) $$
    where $\lambda_{\text{TV}} = 25$ for Siddon's method and $\lambda_{\text{TV}} = 15$ for trilinear interpolation [S1, unofficial].
6.  **Optimization:** Update $\hat{\mu}$ using the Adam optimizer with an initial learning rate of 1, linearly decaying to 0 over 50 iterations [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def softplus(x, beta=20):
    """Applies Softplus activation to ensure non-negative LACs."""
    return F.softplus(x, beta=beta)

def compute_tv_loss(volume):
    """Computes Total Variation regularization term."""
    # Sum of absolute differences between adjacent voxels
    diff_x = torch.abs(volume[:, 1:, :, :] - volume[:, :-1, :, :])
    diff_y = torch.abs(volume[:, :, 1:, :] - volume[:, :, :-1, :])
    diff_z = torch.abs(volume[:, :, :, 1:] - volume[:, :, :, :-1])
    return torch.sum(diff_x) + torch.sum(diff_y) + torch.sum(diff_z)

def render_siddon(volume, rays, source_pos, detector_pos):
    """
    Differentiable renderer using Siddon's method.
    Computes exact line integrals through the voxel grid.
    rays: Tensor of shape (B, 3) representing ray origins or directions.
    Returns: Tensor of shape (B,) with rendered intensities.
    """
    # Implementation details of Siddon's method are assumed to be available
    # as a differentiable operation (e.g., via autograd).
    # This function computes the Beer-Lambert law integration along each ray.
    pass

def render_trilinear(volume, rays, source_pos, detector_pos, M=500):
    """
    Differentiable renderer using Trilinear Interpolation.
    Samples M points along each ray and interpolates volume values.
    """
    # Sample points along rays
    # Interpolate volume values at these points
    # Compute attenuation
    pass

def diffvox_reconstruction(projections, geometry, num_views=15, num_iters=50):
    """
    Main DiffVox reconstruction loop.
    """
    # 1. Initialize voxel grid
    # Dimensions 501x501x501 as per experiments
    volume = torch.zeros((501, 501, 501), requires_grad=True)
    
    # 2. Setup optimizer
    optimizer = torch.optim.Adam([volume], lr=1.0)
    
    # 3. Optimization loop
    for i in range(num_iters):
        # Linearly decay learning rate
        current_lr = 1.0 * (1 - i / num_iters)
        for param_group in optimizer.param_groups:
            param_group['lr'] = current_lr
            
        # 3.1 Sample rays
        # Randomly sample rays from all projection images without replacement
        rays, ground_truth_intensities = sample_rays(projections, geometry)
        
        # 3.2 Forward pass (Choose one renderer)
        # Option A: Siddon's Method
        # rendered_intensities = render_siddon(volume, rays, geometry.source, geometry.detector)
        
        # Option B: Trilinear Interpolation
        rendered_intensities = render_trilinear(volume, rays, geometry.source, geometry.detector, M=500)
        
        # 3.3 Compute Loss
        # L1 photometric loss
        l1_loss = torch.mean(torch.abs(ground_truth_intensities - rendered_intensities))
        
        # Total Variation regularization
        # Note: Apply TV to the active volume (after softplus) or raw volume?
        # Sources imply regularization on the volume representation.
        tv_loss = compute_tv_loss(volume)
        
        # Combined loss
        # Lambda values: 25 for Siddon, 15 for Trilinear
        lambda_tv = 15.0 
        loss = l1_loss + lambda_tv * tv_loss
        
        # 3.4 Backward pass
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # Apply Softplus activation for physical constraint (non-negative LACs)
        # Note: In practice, this might be applied during rendering or as a constraint.
        # The source states: "apply the Softplus activation function ... to mu_hat prior to rendering"
        # This implies the stored parameter is raw, and rendering uses softplus.
        # For optimization, we optimize the raw parameter.
        
    return volume
```

**Parameters:**
*   **Softplus $\beta$:** 20 [S1, unofficial].
*   **Adam Learning Rate:** Initial 1, linearly decaying to 0 over 50 iterations [S1, unofficial].
*   **TV Regularization Weight ($\lambda_{\text{TV}}$):** 25 for Siddon's method, 15 for trilinear interpolation [S1, unofficial].
*   **Trilinear Sampling Points ($M$):** 500 points per ray [S1, unofficial].
*   **Batch Size:** Limited by GPU memory; $5.5 \times 10^5$ rays for Siddon, $1.8 \times 10^6$ for trilinear [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Overfitting with Siddon's Method:** Siddon's method exactly computes line integrals, which can lead to overfitting in sparse-view settings because it does not allow ground truth pixels to supervise non-intersected voxels [S1, unofficial].
*   **Mitigation via Trilinear Interpolation:** Trilinear interpolation mitigates overfitting by approximating second-order effects (like scatter) and allowing rendered pixel intensities to be influenced by voxels not directly intersected by the ray, providing self-supervision to more voxels [S1, unofficial].
*   **Initialization:** Initializing with zeros is standard, but initializing with Nesterov or SIRT estimates can reduce runtime [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** DiffVox is an order of magnitude faster than neural field methods (e.g., SAX-NeRF, NAF) [S1, unofficial].
*   **Memory Complexity:** Depends on the voxel grid resolution ($501^3$ in experiments) and batch size of rays [S1, unofficial].
*   **Measured Performance:** On a single NVIDIA RTX A6000 GPU, for 60-view reconstruction:
    *   DiffVox (Siddon): 45.1 minutes [S1, unofficial].
    *   DiffVox (Trilinear): 39.1 minutes [S1, unofficial].
    *   SAX-NeRF: 10.3 hours [S1, unofficial].
    *   NAF: 30.6 hours [S1, unofficial].
    *   Nesterov (traditional): $\le$ 5 minutes [S1, unofficial].
*   **Quality:** DiffVox outperforms baselines in SSIM, PSNR, PCC, and MSE for sparse views ($\le$ 20) [S1, unofficial]. For 30-60 views, SAX-NeRF achieves higher PSNR than DiffVox [S1, unofficial]. Siddon's method achieves higher accuracy than trilinear interpolation but with moderately increased runtime [S1, unofficial].
*   **Strengths:** High-quality reconstructions in sparse-view regimes ($\le$ 20 views), fast convergence, ability to use well-validated reconstruction algorithms for initialization [S1, unofficial].
*   **Limitations:** Slower than traditional iterative methods (like FDK, Nesterov) for dense views; Siddon's method prone to overfitting in very sparse settings without sufficient regularization [S1, unofficial].

## Background
DiffVox was presented in a paper by Mohammadhossein Momeni, Vivek Gopalakrishnan, Neel Dey, Polina Golland, and Sarah Frisken from Brigham and Women's Hospital and MIT CSAIL [S1, unofficial]. It builds on Siddon's method (1985) [S1, unofficial], Total Variation regularization (1992) [S1, unofficial], and the Adam optimizer (2015) [S1, unofficial]. It is used for CBCT reconstruction from sparse X-ray views, particularly in regimes where traditional and neural methods fail ($\le$ 20 views) [S1, unofficial].

## Sources
[S1] Differentiable Voxel-based X-ray Rendering Improves Sparse-View 3D CBCT Reconstruction, https://doi.org/10.48550/arxiv.2411.19224, fetched 2026-10-26 (unofficial)
