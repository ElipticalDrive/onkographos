# DDPM Sampling at Inference

Also called: DDPM Inference, DDPM Sampling

## Overview
DDPM Sampling at Inference is the process of generating voxelized 3D shapes by reversing a learned Markov chain, as used in [Voxel Generation with Diffusion Models](10976-voxel-generation-with-diffusion-models.md). It differs from [Sharf](18611-sharf.md) by relying on iterative denoising of a latent representation (typically triplane features) rather than direct geometric construction. The method produces occupancy fields that are subsequently decoded into meshes.

## How it works
**Inputs:** A noise tensor $z_T \sim \mathcal{N}(0, I)$ of the same shape as the target latent representation (e.g., triplane features); the number of inference steps $T$; the precomputed noise schedule parameters $\alpha_t$ and $\bar{\alpha}_t$ (alphas_cumprod).

**Outputs:** A denoised latent representation $z_0$, which represents the occupancy field or triplane features.

**Data Structures:**
*   **Noise Schedule:** A tensor of size $T+1$ containing $\bar{\alpha}_t = \prod_{i=1}^t \alpha_i$. In standard DDPM implementations, $T=1000$.
*   **Latent Tensor:** Shape $(B, C, H, W)$ for triplanes, where $B$ is batch size, $C$ is channels, and $H, W$ are spatial dimensions.

**Steps:**
1.  **Initialization:** Sample $z_T$ from a standard normal distribution.
2.  **Iterative Denoising:** For $t = T, T-1, \dots, 1$:
    a.  Compute the variance term $\sigma_t^2 = \frac{1-\bar{\alpha}_{t-1}}{1-\bar{\alpha}_t} \frac{1-\bar{\alpha}_t}{\bar{\alpha}_t}$ [S1].
    b.  Predict the noise $\epsilon_\theta(z_t, t)$ using the trained diffusion model (UNet) [S1].
    c.  Compute the predicted mean $\mu_\theta(z_t, t) = \frac{1}{\sqrt{\bar{\alpha}_t}} (z_t - \frac{1-\bar{\alpha}_t}{\sqrt{1-\bar{\alpha}_t}} \epsilon_\theta(z_t, t))$ [S1].
    d.  Sample $z_{t-1} = \mu_\theta(z_t, t) + \sigma_t \epsilon$, where $\epsilon \sim \mathcal{N}(0, I)$ [S1].
3.  **Decoding:** Evaluate the resulting latent field using an MLP at resolution $128^3$ and extract the mesh using marching cubes [S1].

**Parameters:**
*   $T$: Number of iterations. Default is 1000 [S1]. Can be reduced to 250 for faster generation with minor visual fidelity loss [S1].
*   Resolution: $128^3$ for mesh extraction [S1].

**Edge Cases:**
*   **Precision:** The noise schedule and intermediate tensors should be maintained in high precision to avoid accumulation errors during the 1000-step reverse process [S1].
*   **Interpolation:** Using DDIM for interpolation results in visually worse-quality meshes compared to DDPM sampling [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def ddpm_sample(model, T=1000, device="cuda"):
    """
    Performs DDPM sampling for 3D shape generation.
    
    Args:
        model: The trained diffusion model (UNet) that predicts noise.
        T: Number of inference steps (default 1000).
        device: Device to run inference on.
        
    Returns:
        z_0: The denoised latent representation (triplane features).
    """
    # Step 1: Initialize noise
    # Assuming latent shape is (1, C, H, W) for a single example
    z = torch.randn(1, 3, 64, 64, device=device) # [G] Typical triplane shape
    
    # Precompute noise schedule parameters
    # beta_t is usually linearly spaced in standard DDPM
    betas = torch.linspace(0.0001, 0.02, T, device=device)
    alphas = 1.0 - betas
    alphas_cumprod = torch.cumprod(alphas, dim=0)
    
    # Add alpha_0 = 1 for convenience
    alphas_cumprod = torch.cat([torch.tensor([1.0], device=device), alphas_cumprod])
    
    for t in range(T, 0, -1):
        t_idx = t - 1 # 0-indexed for tensor access if needed, but t is 1..T
        
        # Step 2a: Predict noise
        # model takes (z_t, t) and outputs predicted noise epsilon
        epsilon_pred = model(z, t)
        
        # Step 2b: Compute coefficients
        # alpha_t_bar is alphas_cumprod[t]
        alpha_t_bar = alphas_cumprod[t]
        alpha_t = alphas[t]
        
        # Coefficient for z_t
        coeff1 = 1.0 / math.sqrt(alpha_t_bar)
        
        # Coefficient for epsilon_pred
        # (1 - alpha_t_bar) / sqrt(1 - alpha_t_bar) = sqrt(1 - alpha_t_bar)
        coeff2 = (1.0 - alpha_t_bar) / math.sqrt(1.0 - alpha_t_bar)
        
        # Step 2c: Compute mean
        mu = coeff1 * (z - coeff2 * epsilon_pred)
        
        # Step 2d: Add noise
        if t > 1:
            # Variance sigma_t^2 = (1 - alpha_{t-1}) / (1 - alpha_t) * (1 - alpha_t) / alpha_t
            # Simplifies to: (1 - alpha_{t-1}) / alpha_t
            alpha_prev = alphas_cumprod[t-1]
            sigma_sq = (1.0 - alpha_prev) / alpha_t
            sigma = torch.sqrt(sigma_sq)
            noise = torch.randn_like(z)
            z = mu + sigma * noise
        else:
            # No noise at the final step
            z = mu
            
    return z
```

## Subtopics

### 1-Step DDPM Scheduler
A variant that configures the DDPM scheduler for single-step inference by setting timesteps to 1 and moving `alphas_cumprod` to CUDA. It loads the scheduler from `stabilityai/sd-turbo` and sets timesteps to 1 via `set_timesteps(1, device="cuda")` [S2, unofficial]. This reduces generation time significantly but may alter the quality characteristics compared to the standard iterative approach [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of iterations $T$.
*   **Memory Complexity:** Proportional to the latent volume size and the model size.
*   **Performance:** Generating a set of triplane features for a single example takes roughly 20 seconds on a single NVIDIA A6000 GPU with 1000 iterations [S1]. Decoding the resulting occupancy field and extracting a mesh at $128^3$ resolution takes about 5 seconds per mesh, including MLP evaluation and marching cubes [S1].
*   **Strengths:** Produces high-fidelity 3D shapes; allows for interpolation (though DDIM is preferred for interpolation, DDPM is used for generation) [S1].
*   **Limitations:** Slow generation time due to iterative nature; reducing iterations to 250 causes a small visual fidelity reduction [S1].

## Background
The method originates from "3D Neural Field Generation Using Triplane Diffusion" [S1], which introduced the use of DDPM to generate triplane features for 3D shape synthesis. The approach leverages the stability of DDPM sampling over DDIM for final shape generation, despite DDIM's faster convergence, due to better visual quality [S1].

## Sources
[S1] 3D Neural Field Generation Using Triplane Diffusion, https://doi.org/10.1109/cvpr52729.2023.02000, fetched 2026-10-26
[S2] nv-tlabs/Difix3D, https://github.com/nv-tlabs/Difix3D, fetched 2026-10-26 (unofficial)
