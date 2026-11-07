# Zip-NeRF Training Procedure

Also called: Joint training for Zip-NeRF, Training Procedure, Zip-NeRF-based radiance cache training

## Overview
The Zip-NeRF training procedure is a two-stage optimization method for inverse rendering that jointly trains a physically-based scene model, environment map, and Zip-NeRF-based radiance caches (fast cache and NeRF-cache) [S1]. It distinguishes itself from standard NeRF training by using a control variate approach where the radiance cache estimates the expected radiance, allowing the physically-based model to learn the residual [S1]. The method employs a specific "gradient trick" to handle the non-analytic expected value of the radiance estimator, ensuring correct gradient propagation during the joint optimization phase [S1].

## How it works
The procedure consists of two sequential stages: Cache training and Joint training.

### Inputs and Outputs
*   **Inputs:** Ground truth images $I(\mathbf{o}, \boldsymbol{\omega}_o)$, camera poses, scene geometry (implicit or explicit), and initial parameters for the physically-based model, environment map, fast cache, NeRF-cache, and occlusion-aware importance sampler.
*   **Outputs:** Optimized parameters for the physically-based model, environment map, fast cache, NeRF-cache, and occlusion-aware importance sampler.

### Data Structures
*   **Fast Cache:** A Zip-NeRF-based radiance cache storing incoming radiance estimates.
*   **NeRF-cache:** A Zip-NeRF-based radiance cache storing the residual radiance $\Delta L$.
*   **Physically-based Model:** Parameters $\Phi$ defining the scene's material and lighting properties.
*   **Occlusion-aware Importance Sampler:** Parameters for sampling secondary rays.

### Stage 1: Cache Training
This stage optimizes the Zip-NeRF-based radiance cache independently.

1.  **Sampling:** Sample rays with a batch size of 16,384.
2.  **Loss Calculation:** Compute the Charbonnier loss between the cache prediction and ground truth.
3.  **Optimization:** Update cache parameters for 100,000 iterations.
4.  **Learning Rate Schedule:**
    *   Iterations 0–10,000: Linear warmup from 0 to 0.01.
    *   Iterations 10,001–100,000: Linear decay from 0.01 to 0.001.

### Stage 2: Joint Training
This stage optimizes the physically-based model, environment map, fast cache, NeRF-cache, and occlusion-aware importance sampler jointly.

1.  **Sampling:**
    *   Sample primary rays with a batch size of 1,024.
    *   For each primary ray, generate 64 secondary rays to estimate incoming radiance $\hat{L}_{fast}$ from the fast cache.
    *   Generate 16 secondary rays to estimate the residual $\Delta \hat{L}_o$ using both the fast cache and NeRF-cache.
2.  **Loss Calculation:**
    The photometric loss is defined using the gradient trick to handle the expected value of the estimator [S1].
    
    The target loss function is:
    $$ \mathcal{L}_{\text{photometric}} = \frac{2 \left( I(\mathbf{o}, \boldsymbol{\omega}_o) - \hat{L}_i(\mathbf{o}, \boldsymbol{\omega}_o; \Phi) \right) \operatorname{stopgrad}\left( I(\mathbf{o}, \boldsymbol{\omega}_o) - \hat{L}_i(\mathbf{o}, \boldsymbol{\omega}_o; \Phi) \right)}{\operatorname{stopgrad}\left( \hat{L}_i^{\text{cache}}(\mathbf{o}, \boldsymbol{\omega}_o; \Phi) \right)} $$
    
    Where:
    *   $I(\mathbf{o}, \boldsymbol{\omega}_o)$ is the ground truth pixel color.
    *   $\hat{L}_i(\mathbf{o}, \boldsymbol{\omega}_o; \Phi)$ is the predicted radiance from the physically-based model.
    *   $\hat{L}_i^{\text{cache}}$ is the radiance estimate from the cache (control variate).
    *   $\operatorname{stopgrad}(\cdot)$ denotes the stop-gradient operation, preventing gradients from flowing through the cache or the residual term in the numerator's second factor.
    
    *Note:* The standard Charbonnier loss is replaced by this variant (derived from RawNeRF [S1]) because the expected value of the radiance estimator lacks an analytic expression, making direct minimization impossible without the gradient trick [S1].
3.  **Optimization:**
    *   Update all parameters (physically-based model, environment map, fast cache, NeRF-cache, importance sampler).
    *   **Learning Rates:**
        *   Fast cache and occlusion-aware importance sampler: $1.5625 \times 10^{-4}$.
        *   All other parameters (physically-based model, environment map, NeRF-cache): $3.125 \times 10^{-5}$.
    *   **Iterations:**
        *   TensoIR-synthetic dataset: 40,000 iterations.
        *   Open Illumination dataset: 100,000 iterations.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_zip_nerf(dataset, config):
    """
    Main training loop for Zip-NeRF.
    config: dict containing dataset type, batch sizes, iteration counts, learning rates.
    """
    # Initialize models
    fast_cache = ZipNeRF()
    nerf_cache = ZipNeRF()
    phys_model = PhysicallyBasedModel()
    env_map = EnvironmentMap()
    importance_sampler = OcclusionAwareSampler()
    
    # Stage 1: Cache Training
    print("Starting Stage 1: Cache Training")
    cache_optimizer = Optimizer(fast_cache.parameters(), lr_schedule=linear_warmup_decay(0, 0.01, 10000, 0.001))
    
    for iteration in range(100000):
        # Sample batch of rays
        rays = sample_rays(batch_size=16384)
        
        # Predict radiance from cache
        cache_pred = fast_cache(rays)
        
        # Charbonnier loss
        loss = charbonnier_loss(cache_pred, rays.gt_color)
        
        # Backpropagate
        cache_optimizer.zero_grad()
        loss.backward()
        cache_optimizer.step()
        
        # Update learning rate based on schedule
        current_lr = get_lr(iteration, start=0, peak=0.01, peak_iter=10000, end=0.001, end_iter=100000)
        cache_optimizer.set_lr(current_lr)

    # Stage 2: Joint Training
    print("Starting Stage 2: Joint Training")
    
    # Determine iteration count based on dataset
    if dataset == "TensoIR-synthetic":
        total_iters = 40000
    else:
        total_iters = 100000
        
    # Optimizers with specific learning rates
    lr_fast = 1.5625e-4
    lr_other = 3.125e-5
    
    fast_cache_opt = Optimizer(fast_cache.parameters(), lr=lr_fast)
    sampler_opt = Optimizer(importance_sampler.parameters(), lr=lr_fast)
    
    # Group other parameters
    other_params = list(phys_model.parameters()) + list(env_map.parameters()) + list(nerf_cache.parameters())
    other_opt = Optimizer(other_params, lr=lr_other)
    
    for iteration in range(total_iters):
        # Sample primary rays
        primary_rays = sample_rays(batch_size=1024)
        
        # Generate secondary rays for fast cache (64 samples)
        fast_secondary = importance_sampler.sample_secondary(primary_rays, num_samples=64)
        L_fast = fast_cache(fast_secondary)
        
        # Generate secondary rays for NeRF-cache (16 samples)
        nerf_secondary = importance_sampler.sample_secondary(primary_rays, num_samples=16)
        L_nerf = nerf_cache(nerf_secondary)
        
        # Estimate residual and total radiance
        # Note: Exact combination formula for L_fast and L_nerf to get L_i is implied by the cache structure
        # but the loss function uses the final predicted L_i from the physically-based model
        L_pred = phys_model(primary_rays, env_map, L_fast, L_nerf)
        
        # Compute Photometric Loss with Gradient Trick
        # L_photometric = 2 * (I - L_pred) * stopgrad(I - L_pred) / stopgrad(L_cache)
        # In practice, L_cache is the control variate estimate. 
        # The source defines the loss using the cache estimate in the denominator.
        
        # Calculate the numerator term: (I - L_pred)
        diff = primary_rays.gt_color - L_pred
        
        # Apply stopgrad to the residual term in the numerator's second factor
        # and to the cache estimate in the denominator
        numerator = 2 * diff * stopgrad(diff)
        denominator = stopgrad(L_fast) # Using fast cache as the primary control variate estimate
        
        loss = numerator / (denominator + 1e-8) # Add epsilon for stability
        loss = loss.mean()
        
        # Backpropagate
        fast_cache_opt.zero_grad()
        sampler_opt.zero_grad()
        other_opt.zero_grad()
        
        loss.backward()
        
        fast_cache_opt.step()
        sampler_opt.step()
        other_opt.step()

    return phys_model, env_map, fast_cache, nerf_cache, importance_sampler
```

### Edge Cases and Pitfalls
*   **Gradient Estimation:** The loss cannot be minimized directly because the expected value of the radiance estimator lacks an analytic expression. Failure to apply the gradient trick (stop-gradient on specific terms) will result in incorrect gradients in expectation [S1].
*   **Precision:** The division by the stop-graded cache estimate requires care to avoid division by zero or instability; a small epsilon should be added [G].

## Subtopics

### [Fast Cache (Control Variate Radiance Cache)](9552-fast-cache-control-variate-radiance-cache.md)
A Zip-NeRF-based radiance cache used to estimate incoming radiance, serving as a control variate to reduce variance in the joint training phase.

## Operating characteristics
*   **Time Complexity:** Stage 1 takes 5.5 hours on a single NVIDIA A100 GPU for 100,000 iterations. Stage 2 takes 6.5 hours on a single NVIDIA A100 GPU for 100,000 iterations [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The method is designed to reduce bias in radiance cache-based inverse rendering [S1].
*   **Strengths:** Allows for efficient joint optimization of physically-based models and radiance caches by decoupling the gradient flow through the control variate.
*   **Limitations:** Requires two distinct training stages. The learning rate schedule is specific to the datasets tested (TensoIR-synthetic and Open Illumination).
*   **When to use:** When performing inverse rendering with radiance caches and needing to jointly optimize scene parameters and cache representations.
*   **When not to use:** When computational resources are extremely limited and two-stage training is infeasible, or when high-frequency details are not critical enough to warrant the complexity of cache-based control variates.

## Background
The Zip-NeRF training procedure was introduced in the paper "Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering" [S1]. It builds upon the Zip-NeRF representation and integrates concepts from RawNeRF's loss formulation to handle the stochastic nature of radiance estimation in inverse rendering [S1].

## Sources
[S1] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
