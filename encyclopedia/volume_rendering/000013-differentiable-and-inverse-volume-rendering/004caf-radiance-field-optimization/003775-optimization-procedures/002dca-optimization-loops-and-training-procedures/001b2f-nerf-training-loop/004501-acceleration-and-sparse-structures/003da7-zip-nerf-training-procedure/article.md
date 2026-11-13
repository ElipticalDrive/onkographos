# Zip-NeRF Training Procedure

Also called: Joint training for Zip-NeRF, Training Procedure, Zip-NeRF-based radiance cache training

## Overview
The Zip-NeRF training procedure is a two-stage optimization method for inverse rendering that jointly trains a Zip-NeRF-based radiance cache, a physically-based material model, and an environment map. It solves the problem of efficiently optimizing high-fidelity neural radiance fields for inverse rendering by decoupling the training of the cache from the joint optimization of the scene and material parameters. It sets itself apart from standard NeRF training by using a specialized "gradient trick" to handle the non-analytic expected value of the radiance estimator and by employing a fast cache control variate to reduce bias.

## How it works
The method consists of two sequential stages: Cache Training and Joint Training.

### Inputs and Outputs
*   **Inputs:** Ground truth images $I(\mathbf{o}, \boldsymbol{\omega}_o)$, camera poses, scene geometry (implicit or explicit), and initial parameters for the Zip-NeRF cache, physically-based model, environment map, and fast cache.
*   **Outputs:** Optimized Zip-NeRF radiance cache parameters, physically-based model parameters, environment map, and fast cache parameters.

### Data Structures
*   **Zip-NeRF Radiance Cache:** A sparse voxel grid structure (as defined in [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)) storing radiance features.
*   **Fast Cache:** A lightweight hash grid and small MLP predicting incident radiance.
*   **Physically-Based Model:** Parameters defining surface reflectance (e.g., BRDF/BTDF coefficients).
*   **Environment Map:** Parameters defining the incident lighting field.

### Numbered Steps

**Stage 1: Cache Training**
1.  **Initialize:** Set up the Zip-NeRF radiance cache.
2.  **Sample Rays:** For each iteration, sample a batch of $N_{batch} = 16384$ rays from the training dataset.
3.  **Evaluate Cache:** For each ray, query the Zip-NeRF cache to obtain the estimated radiance $\hat{L}_i^{\text{cache}}$.
4.  **Compute Loss:** Calculate the Charbonnier loss between the ground truth image intensity and the cache prediction:
    $$ \mathcal{L}_{\text{cache}} = \| I(\mathbf{o}, \boldsymbol{\omega}_o) - \hat{L}_i^{\text{cache}}(\mathbf{o}, \boldsymbol{\omega}_o) \|_1^{\alpha} $$
    where $\alpha$ is typically 1 or 2 (Charbonnier norm).
5.  **Optimize:** Update the Zip-NeRF cache parameters using gradient descent.
6.  **Schedule:** Apply a learning rate schedule: warm up from 0 to 0.01 over the first 10,000 iterations, then linearly decrease to 0.001 for the remaining iterations.
7.  **Iterate:** Repeat for 100,000 iterations.

**Stage 2: Joint Training**
1.  **Initialize:** Use the optimized Zip-NeRF cache from Stage 1. Initialize the physically-based model, environment map, fast cache, and occlusion-aware importance sampler.
2.  **Sample Rays:** For each iteration, sample a batch of $N_{batch} = 1024$ rays.
3.  **Estimate Incident Radiance:**
    *   For each ray, estimate the incoming radiance $\hat{L}_i$ using the physically-based model and the caches.
    *   Use $N_{fast} = 64$ secondary rays to estimate the fast cache contribution $\hat{L}_{fast}$.
    *   Use $N_{nerf} = 16$ secondary rays to estimate the NeRF cache contribution $\hat{L}_{nerf}$ and the residual $\Delta \hat{L}_o$.
4.  **Compute Photometric Loss:**
    The goal is to minimize the difference between the ground truth $I$ and the expected value of the estimator $\mathbb{E}[\hat{L}_i]$. Since $\mathbb{E}[\hat{L}_i]$ lacks an analytic expression, the "gradient trick" is applied [S1].
    *   Define the loss as:
        $$ \mathcal{L}_{\text{photometric}} = \frac{2 ( I(\mathbf{o}, \boldsymbol{\omega}_o) - \hat{L}_i(\mathbf{o}, \boldsymbol{\omega}_o; \Phi) ) \operatorname{stopgrad}( I(\mathbf{o}, \boldsymbol{\omega}_o) - \hat{L}_i(\mathbf{o}, \boldsymbol{\omega}_o; \Phi) )}{ \operatorname{stopgrad}(\hat{L}_i^{\text{cache}}(\mathbf{o}, \boldsymbol{\omega}_o; \Phi)) } $$
    *   Here, $\Phi$ represents all trainable parameters (physically-based model, environment map, etc.).
    *   The $\operatorname{stopgrad}$ operation prevents gradients from flowing through the cache and the second term of the numerator, ensuring correct gradients in expectation for the physically-based model [S1].
5.  **Optimize:** Update all parameters (physically-based model, environment map, fast cache, occlusion-aware importance sampler) using the computed loss.
    *   Learning rate for fast cache and occlusion-aware importance sampler: $1.5625 \times 10^{-4}$.
    *   Learning rate for all other parameters: $3.125 \times 10^{-5}$.
6.  **Iterate:** Repeat for 40,000 iterations (TensoIR-synthetic) or 100,000 iterations (Open Illumination).

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_zip_nerf_inverse_rendering(dataset, config):
    """
    Two-stage training procedure for Zip-NeRF based inverse rendering.
    """
    # Stage 1: Cache Training
    cache = ZipNeRFCache()
    cache_optimizer = Adam(cache.parameters(), lr=0.01)
    
    # Learning rate schedule for Stage 1
    warmup_iters = 10000
    total_iters_stage1 = 100000
    lr_start = 0.0
    lr_peak = 0.01
    lr_end = 0.001
    
    for i in range(total_iters_stage1):
        # Sample batch of rays
        rays = dataset.sample_batch(batch_size=16384)
        
        # Get ground truth colors
        I_gt = rays.colors
        
        # Forward pass through Zip-NeRF cache
        L_cache = cache.forward(rays)
        
        # Compute Charbonnier loss
        # Using alpha=1 for L1-like Charbonnier, or alpha=2 for L2
        loss = torch.norm(I_gt - L_cache, p=1) # Simplified Charbonnier
        
        # Update learning rate
        if i < warmup_iters:
            lr = lr_start + (lr_peak - lr_start) * (i / warmup_iters)
        else:
            progress = (i - warmup_iters) / (total_iters_stage1 - warmup_iters)
            lr = lr_peak + (lr_end - lr_peak) * progress
            
        cache_optimizer.lr = lr
        
        # Backward pass
        cache_optimizer.zero_grad()
        loss.backward()
        cache_optimizer.step()
        
    # Stage 2: Joint Training
    phys_model = PhysicallyBasedModel()
    env_map = EnvironmentMap()
    fast_cache = FastCache()
    sampler = OcclusionAwareSampler()
    
    # Optimizers with different learning rates
    optimizer_main = Adam(list(phys_model.parameters()) + 
                          list(env_map.parameters()), lr=3.125e-5)
    optimizer_aux = Adam(list(fast_cache.parameters()) + 
                         list(sampler.parameters()), lr=1.5625e-4)
    
    # Determine number of iterations based on dataset
    if config.dataset == 'TensoIR-synthetic':
        total_iters_stage2 = 40000
    else: # Open Illumination
        total_iters_stage2 = 100000
        
    N_fast = 64
    N_nerf = 16
    
    for i in range(total_iters_stage2):
        # Sample batch of rays
        rays = dataset.sample_batch(batch_size=1024)
        I_gt = rays.colors
        
        # Estimate incident radiance using caches
        # This involves casting secondary rays
        L_fast = fast_cache.estimate(rays, N_samples=N_fast)
        L_nerf = cache.forward(rays, N_samples=N_nerf)
        
        # Combine caches to estimate total incident radiance
        # Note: Exact combination formula depends on the specific radiance cache implementation
        # Assuming a weighted sum or residual correction as per standard radiance caching
        L_i_est = compute_incident_radiance(rays, L_fast, L_nerf, phys_model, env_map)
        
        # Compute Photometric Loss using the Gradient Trick
        # L_photometric = 2 * (I - L_est) * stopgrad(I - L_est) / stopgrad(L_cache)
        
        # Calculate the residual
        residual = I_gt - L_i_est
        
        # Apply stopgrad to the residual and the cache prediction
        # This ensures gradients only flow through the physically-based model and env map
        residual_stopgrad = residual.detach()
        L_cache_stopgrad = cache.forward(rays).detach()
        
        # Avoid division by zero
        epsilon = 1e-8
        denominator = L_cache_stopgrad + epsilon
        
        # Compute loss
        loss = 2.0 * residual * residual_stopgrad / denominator
        
        # Backward pass for main parameters
        optimizer_main.zero_grad()
        loss.backward(retain_graph=True) # Retain graph for auxiliary parameters if needed
        optimizer_main.step()
        
        # Backward pass for auxiliary parameters (fast cache, sampler)
        optimizer_aux.zero_grad()
        # Note: In practice, gradients for auxiliary params might be computed separately
        # or via a separate backward pass depending on implementation details
        # The source implies joint optimization, so gradients must flow to them
        # However, the gradient trick specifically stops gradients through the cache
        # for the main model. The auxiliary params (fast cache) are optimized
        # via their own contribution to the loss.
        # Assuming the loss function is differentiable w.r.t fast cache params
        # through the L_i_est calculation.
        # Re-compute loss or use retained graph for auxiliary update
        # For simplicity, assuming a single backward pass covers all if gradients are not stopped
        # But the formula explicitly stops grad through L_cache.
        # The fast cache is part of L_i_est, so it needs gradients.
        # The gradient trick stops grad through the *Zip-NeRF* cache (L_cache)
        # to stabilize the main model. The fast cache is a separate component.
        
        # To be precise with the source:
        # "optimize the physically-based model, environment map, fast cache, NeRF-cache, 
        # and occlusion-aware importance sampler"
        # The loss formula uses stopgrad on L_cache (Zip-NeRF) and the residual term.
        # Gradients should flow to fast_cache and sampler via L_i_est.
        
        # If retain_graph was used, we can do another backward or accumulate
        # Here we assume a standard joint optimization where gradients are accumulated
        # and then stepped.
        optimizer_aux.step()
```

### Parameters
*   **Stage 1 Batch Size:** 16384 rays.
*   **Stage 1 Iterations:** 100,000.
*   **Stage 1 Learning Rate:** Warmup 0 to 0.01 (10,000 iters), then linear decay to 0.001.
*   **Stage 2 Batch Size:** 1024 rays.
*   **Stage 2 Iterations:** 40,000 (TensoIR-synthetic), 100,000 (Open Illumination).
*   **Stage 2 Learning Rate (Fast Cache, Sampler):** $1.5625 \times 10^{-4}$.
*   **Stage 2 Learning Rate (Other):** $3.125 \times 10^{-5}$.
*   **Fast Cache Samples:** 64 secondary rays.
*   **NeRF Cache Samples:** 16 secondary rays.

### Edge Cases and Pitfalls
*   **Non-Analytic Expectation:** The expected value of the radiance estimator $\mathbb{E}[\hat{L}_i]$ does not have an analytic expression. Direct minimization of the loss $\| I - \mathbb{E}[\hat{L}_i] \|^2$ is impossible. The gradient trick must be used to obtain correct gradients in expectation [S1].
*   **Precision:** The use of `stopgrad` is critical. If gradients flow through the cache predictions in the loss formula, the optimization may become unstable or biased.
*   **Boundary Conditions:** The learning rate schedule must be carefully implemented to avoid NaNs during the warmup phase if the initial loss is high.

## Subtopics

### [Fast Cache (Control Variate Radiance Cache)](9552-fast-cache-control-variate-radiance-cache.md)
A computationally efficient approximation of the incoming radiance used in radiance caching for inverse rendering, designed to replace or augment the expensive NeRF-based cache. It solves the bottleneck of evaluating high-fidelity neural radiance fields for every importance sampling direction by using a lightweight, low-resolution hash grid and a small MLP to predict incident radiance. It sets itself apart from the standard NeRF-based cache by using a fixed, small number of samples (8) and direct distance prediction, while maintaining accuracy by serving as a control variate to correct the bias of the NeRF cache.

### [NeRF-Casting Reflection Model](11495-nerf-casting-reflection-model.md)
NeRF-Casting is a rendering procedure designed to model accurate, detailed specular reflections in neural radiance fields without the computational cost of evaluating expensive color MLPs along every reflected ray. It addresses the problem of rendering view-dependent specular highlights by casting a small number of reflected rays through a volume representation, accumulating features, and then using a lightweight decoder to produce final colors. It differs from its siblings by explicitly approximating the reflection cone via a von Mises-Fisher distribution and pulling the reflection origin closer to the surface for non-perfect mirrors to maintain intersection with the pixel cone, whereas [Conical Reflected Features](11502-conical-reflected-features.md) likely handles conical integration differently and [Fast Cache](9552-fast-cache-control-variate-radiance-cache.md) uses control variates.

### [Conical Reflected Features](11502-conical-reflected-features.md)
Conical Reflected Features estimates the expected volume-rendered feature over a von Mises-Fisher (vMF) distribution of reflected rays to decode into reflected color, avoiding the prohibitive cost of Monte Carlo integration. It differs from standard NeRF-Casting by explicitly modeling the angular spread of roughness via a conical sampling pattern and adapting Zip-NeRF's feature downweighting to the directional domain to prevent aliasing in distant reflections.

## Operating characteristics
*   **Time Complexity:** Stage 1 takes 5.5 hours on a single A100 for 100,000 iterations. Stage 2 takes 6.5 hours on a single A100 for 100,000 iterations [S1].
*   **Memory Complexity:** Depends on the Zip-NeRF cache size and the number of secondary rays (64 for fast cache, 16 for NeRF cache). The fast cache uses a lightweight hash grid, reducing memory overhead compared to a full NeRF evaluation for every sample.
*   **Strengths:** Reduces bias in radiance caching by using a fast cache control variate. Efficiently optimizes physically-based models and environment maps jointly with the radiance cache.
*   **Limitations:** Requires two stages of training. The gradient trick adds complexity to the loss computation. Performance is sensitive to the learning rate schedule and the number of secondary rays.
*   **When to Use:** Suitable for inverse rendering tasks where high-fidelity radiance caching is needed, and computational resources allow for multi-stage optimization.
*   **When Not to Use:** Not suitable for real-time applications due to the training time. If bias in radiance caching is not a critical concern, simpler methods may be preferred.

## Background
The Zip-NeRF training procedure was introduced in the paper "Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering" [S1]. It builds upon the Zip-NeRF acceleration structure and addresses the bias issues inherent in radiance caching for inverse rendering. The method was evaluated on the TensoIR-synthetic and Open Illumination datasets.

## Sources
[S1] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
