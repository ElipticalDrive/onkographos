# Zip-NeRF Training Procedure

Also called: Joint training for Zip-NeRF, Training Procedure, Zip-NeRF-based radiance cache training

## Overview
The Zip-NeRF training procedure is a two-stage optimization method for inverse rendering that jointly trains a physically-based material model, environment map, and a hybrid radiance cache system (comprising a Fast Cache and a NeRF-cache) [S1]. It solves the problem of high-variance gradient estimation in differentiable rendering by using the cache as a control variate to correct bias while maintaining computational efficiency [S1]. It sets itself apart from standard NeRF training by explicitly optimizing the cache parameters alongside the scene geometry and materials, and by employing the "gradient trick" to ensure correct backpropagation through the stochastic radiance estimator [S1].

## How it works
The method consists of two sequential stages: Cache Training and Joint Training.

### Inputs and Outputs
*   **Inputs:** Ground truth images $I(\mathbf{o}, \boldsymbol{\omega}_o)$, camera poses, and initial scene parameters.
*   **Outputs:** Optimized physically-based material parameters $\Phi$, environment map, Fast Cache parameters, NeRF-cache parameters, and occlusion-aware importance sampler parameters.

### Data Structures
*   **Fast Cache:** A lightweight hash grid and small MLP predicting incident radiance [S1].
*   **NeRF-cache:** A neural radiance field cache storing accumulated features [S1].
*   **Physically-based Model:** The primary scene representation being optimized [S1].

### Numbered Steps

**Stage 1: Cache Training**
1.  **Initialize:** Set up the Zip-NeRF-based radiance cache.
2.  **Sample Rays:** Draw a batch of 16,384 rays.
3.  **Compute Loss:** Calculate the Charbonnier loss between the cache prediction and ground truth [S1].
4.  **Update:** Optimize the cache parameters using the specified learning rate schedule [S1].
5.  **Iterate:** Repeat for 100,000 iterations [S1].

**Stage 2: Joint Training**
1.  **Initialize:** Load optimized cache parameters from Stage 1. Initialize physically-based model, environment map, and sampler.
2.  **Sample Rays:** Draw a batch of 1,024 rays [S1].
3.  **Estimate Radiance:**
    *   Use 64 secondary rays to estimate incoming radiance $\hat{L}_{\text{fast}}$ from the Fast Cache [S1].
    *   Use 16 secondary rays to estimate the correction term $\Delta \hat{L}_o$ using both the Fast Cache and NeRF-cache [S1].
4.  **Compute Photometric Loss:** Apply the RawNeRF-style loss using the gradient trick [S1]:
    $$
    \mathcal{L}_{\text{photometric}} = \frac{2 \left( I(\mathbf{o}, \boldsymbol{\omega}_o) - \hat{L}_{\mathrm{i}}(\mathbf{o}, \boldsymbol{\omega}_o ; \Phi) \right) \operatorname{stopgrad}\left( I(\mathbf{o}, \boldsymbol{\omega}_o) - \hat{L}_{\mathrm{i}}(\mathbf{o}, \boldsymbol{\omega}_o ; \Phi) \right)}{\operatorname{stopgrad}\left(\hat{L}_{\mathrm{i}}^{\text{cache}}(\mathbf{o}, \boldsymbol{\omega}_o ; \Phi)\right)}
    $$
    where $\hat{L}_{\mathrm{i}}$ is the estimated radiance and $\hat{L}_{\mathrm{i}}^{\text{cache}}$ is the cache estimator [S1].
5.  **Update:** Optimize all parameters (physically-based model, environment map, fast cache, NeRF-cache, sampler) using the computed loss [S1].
6.  **Iterate:** Repeat for 40,000 iterations (TensoIR-synthetic) or 100,000 iterations (Open Illumination) [S1].

### Parameters
*   **Stage 1 Learning Rate:** Warms up from 0 to 0.01 over the first 10,000 iterations, then linearly decreases to 0.001 for the remaining 90,000 iterations [S1].
*   **Stage 2 Learning Rate:**
    *   Fast Cache and Occlusion-aware Importance Sampler: $1.5625 \times 10^{-4}$ [S1].
    *   All other parameters (Physically-based model, Environment Map, NeRF-cache): $3.125 \times 10^{-5}$ [S1].
*   **Batch Sizes:** 16,384 rays (Stage 1); 1,024 rays (Stage 2) [S1].
*   **Secondary Rays:** 64 for Fast Cache; 16 for NeRF-cache [S1].

### Edge Cases and Pitfalls
*   **Gradient Flow:** The loss cannot be minimized directly because the expected value of the radiance estimator lacks an analytic expression. Applying the gradient trick (stopping gradients on the cache estimator) is required to obtain correct gradients in expectation for the physically-based model [S1].
*   **Precision:** Not given in the sources.
*   **Boundaries:** Not given in the sources.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_zip_nrf_stage1(cache, ground_truth_images, num_iterations=100000):
    """
    Stage 1: Optimize the Zip-NeRF-based radiance cache.
    """
    batch_size = 16384
    lr_schedule = LinearWarmupDecay(start_lr=0.0, peak_lr=0.01, warmup_steps=10000, end_lr=0.001, total_steps=num_iterations)
    
    for step in range(num_iterations):
        # Sample batch of rays
        rays = sample_rays(ground_truth_images, batch_size)
        
        # Predict radiance using cache
        predicted_radiance = cache.forward(rays)
        
        # Compute Charbonnier loss
        loss = charbonnier_loss(predicted_radiance, rays.gt_radiance)
        
        # Update cache parameters
        optimizer = Optimizer(cache.parameters(), lr=lr_schedule.get_lr(step))
        optimizer.step(loss)
        
    return cache

def train_zip_nrf_stage2(phys_model, env_map, fast_cache, nerf_cache, sampler, ground_truth_images, num_iterations, dataset_type):
    """
    Stage 2: Jointly optimize physically-based model, environment map, and caches.
    """
    batch_size = 1024
    fast_cache_samples = 64
    nerf_cache_samples = 16
    
    # Learning rates
    lr_fast_cache = 1.5625e-4
    lr_other = 3.125e-5
    
    optimizer_fast = Optimizer([fast_cache.parameters(), sampler.parameters()], lr=lr_fast_cache)
    optimizer_other = Optimizer([phys_model.parameters(), env_map.parameters(), nerf_cache.parameters()], lr=lr_other)
    
    for step in range(num_iterations):
        # Sample batch of rays
        rays = sample_rays(ground_truth_images, batch_size)
        
        # Estimate incoming radiance from Fast Cache
        L_hat_fast = fast_cache.forward(rays, num_samples=fast_cache_samples)
        
        # Estimate correction term using Fast Cache and NeRF-cache
        delta_L_hat = nerf_cache.forward(rays, L_hat_fast, num_samples=nerf_cache_samples)
        
        # Combine to get final radiance estimate
        L_hat_i = L_hat_fast + delta_L_hat
        
        # Compute Photometric Loss with Gradient Trick
        # Note: stopgrad is applied to the cache estimator and the residual term
        residual = rays.gt_radiance - L_hat_i
        loss = 2.0 * residual * stopgrad(residual) / stopgrad(L_hat_fast)
        
        # Update all parameters
        optimizer_fast.step(loss)
        optimizer_other.step(loss)
        
    return phys_model, env_map, fast_cache, nerf_cache, sampler
```

## Subtopics

### [Fast Cache (Control Variate Radiance Cache)](9552-fast-cache-control-variate-radiance-cache.md)
A computationally efficient approximation of incoming radiance using a lightweight hash grid and small MLP, serving as a control variate to correct bias.

### [NeRF-Casting Reflection Model](11495-nerf-casting-reflection-model.md)
A rendering procedure for modeling specular reflections by casting reflected rays through a volume representation and using a lightweight decoder.

### [Conical Reflected Features](11502-conical-reflected-features.md)
Estimates expected volume-rendered features over a vMF distribution of reflected rays to decode reflected color, avoiding Monte Carlo integration costs.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Stage 1 takes 5.5 hours on a single A100 for 100,000 iterations. Stage 2 takes 6.5 hours on a single A100 for 100,000 iterations [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Reduces bias in radiance cache-based inverse rendering by using a hybrid cache system and joint optimization [S1].
*   **Limitations:** Requires two distinct training stages and careful tuning of learning rates for different components [S1].
*   **When to use:** When high-fidelity inverse rendering is required and computational efficiency is a concern, particularly for scenes with complex lighting and materials [S1].
*   **When not to use:** When real-time rendering is required or when the scene geometry is simple enough that a standard NeRF suffices [S1].

## Background
The Zip-NeRF training procedure was introduced in "Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering" [S1]. It builds upon the Zip-NeRF acceleration structure and integrates it with a physically-based rendering framework to enable efficient inverse rendering [S1].

## Sources
[S1] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
