# Zip-NeRF Training Procedure

Also called: Joint training for Zip-NeRF, Training Procedure, Zip-NeRF-based radiance cache training

## Overview
The Zip-NeRF Training Procedure is a two-stage optimization method for inverse rendering that jointly trains a physically-based material model, environment map, and radiance caches (Zip-NeRF and Fast Cache) to estimate incoming radiance. It solves the problem of high-variance, high-cost radiance estimation in inverse rendering by decoupling the training of the radiance field representation from the material optimization. It sets itself apart from standard NeRF training by using a specialized photometric loss with a gradient trick to handle the non-analytic expectation of the radiance estimator, and by employing a two-stage schedule that first stabilizes the cache before joint optimization [S1].

## How it works
The method consists of two sequential stages: Cache Training and Joint Training.

### Inputs and Outputs
**Inputs:**
*   Dataset of multi-view images (e.g., TensoIR-synthetic, Open Illumination).
*   Camera poses and intrinsics.
*   Initial Zip-NeRF structure (sparse hash grid + MLP) and Fast Cache structure (low-res hash grid + small MLP).
*   Physically-based material model parameters $\Phi$.
*   Environment map parameters.

**Outputs:**
*   Optimized Zip-NeRF radiance cache.
*   Optimized Fast Cache.
*   Optimized material parameters $\Phi$.
*   Optimized environment map.

### Data Structures
*   **Zip-NeRF Cache:** A sparse hash grid storing features, decoded by an MLP to predict radiance. Accessed via spatial constraints defined in [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md).
*   **Fast Cache:** A lightweight hash grid and small MLP predicting incoming radiance $L_{fast}$ [S1].
*   **Occlusion-aware Importance Sampler:** Samples directions for cache evaluation.

### Numbered Steps

#### Stage 1: Cache Training
1.  **Initialize:** Set up Zip-NeRF cache.
2.  **Sample Rays:** Sample a batch of $N_{rays} = 16384$ primary rays from the training dataset.
3.  **Evaluate Cache:** For each ray, query the Zip-NeRF cache to estimate incoming radiance $\hat{L}_{cache}$.
4.  **Compute Loss:** Calculate the Charbonnier loss between the predicted cache radiance and the ground truth incoming radiance (derived from the dataset or a proxy).
    $$ \mathcal{L}_{cache} = \sum_{i=1}^{N_{rays}} \sqrt{(\hat{L}_{cache}^{(i)} - L_{gt}^{(i)})^2 + \epsilon} $$
    where $\epsilon$ is a small constant (standard Charbonnier definition) [S1].
5.  **Update:** Optimize Zip-NeRF cache parameters using the computed loss.
6.  **Schedule:** Repeat for 100,000 iterations.
    *   **Learning Rate:** Warm up from 0 to 0.01 over the first 10,000 iterations.
    *   **Decay:** Linearly decrease from 0.01 to 0.001 over the remaining 90,000 iterations [S1].

#### Stage 2: Joint Training
1.  **Initialize:** Load optimized Zip-NeRF cache from Stage 1. Initialize material parameters $\Phi$, environment map, and Fast Cache.
2.  **Sample Rays:** Sample a batch of $N_{rays} = 1024$ primary rays.
3.  **Estimate Incoming Radiance:** For each primary ray, estimate incoming radiance $\hat{L}_i$ using a combination of caches:
    *   Use 64 secondary rays to estimate $L_{fast}$ from the Fast Cache.
    *   Use 16 secondary rays to estimate $\Delta L_o$ from the NeRF-cache (Zip-NeRF) [S1].
    *   Combine these estimates to form the final radiance estimator $\hat{L}_i(\mathbf{o}, \omega_o; \Phi)$.
4.  **Compute Photometric Loss:** Calculate the loss between the ground truth image $I(\mathbf{o}, \omega_o)$ and the estimated radiance $\hat{L}_i$. Because the expected value $\mathbb{E}[\hat{L}_i]$ lacks an analytic expression, apply the **gradient trick** [S1]:
    $$ \mathcal{L}_{photometric} = \frac{2 \left( I(\mathbf{o}, \omega_o) - \hat{L}_i(\mathbf{o}, \omega_o; \Phi) \right) \operatorname{stopgrad}\left( I(\mathbf{o}, \omega_o) - \hat{L}_i(\mathbf{o}, \omega_o; \Phi) \right)}{\operatorname{stopgrad}\left( \hat{L}_i^{cache}(\mathbf{o}, \omega_o; \Phi) \right)} $$
    where $\operatorname{stopgrad}(\cdot)$ denotes the stop-gradient operation, preventing gradients from flowing through the cache estimation during the material update [S1].
5.  **Update:** Optimize material parameters $\Phi$, environment map, Fast Cache, and Occlusion-aware Importance Sampler.
    *   **Learning Rate for Fast Cache & Sampler:** $1.5625 \times 10^{-4}$ [S1].
    *   **Learning Rate for Others (Material, Env Map):** $3.125 \times 10^{-5}$ [S1].
6.  **Iterate:** Repeat for 40,000 iterations (TensoIR-synthetic) or 100,000 iterations (Open Illumination) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_zip_nrf_stage1(dataset, zip_nrf_cache, iterations=100000, batch_size=16384):
    """
    Stage 1: Optimize Zip-NeRF radiance cache.
    """
    optimizer = Adam(zip_nrf_cache.parameters(), lr=0.0)
    
    for step in range(iterations):
        # Learning rate schedule
        if step < 10000:
            lr = 0.01 * (step / 10000.0)
        else:
            lr = 0.01 - (0.009 * (step - 10000) / 90000.0)
        
        for param_group in optimizer.param_groups:
            param_group['lr'] = lr
            
        # Sample batch of rays
        rays = sample_rays(dataset, batch_size)
        
        # Evaluate Zip-NeRF cache
        # Assumes zip_nrf_cache.forward(rays) returns estimated incoming radiance
        L_cache = zip_nrf_cache.forward(rays)
        
        # Get ground truth incoming radiance (proxy or pre-computed)
        L_gt = get_ground_truth_incoming_radiance(rays)
        
        # Charbonnier Loss
        loss = torch.sqrt((L_cache - L_gt)**2 + 1e-6).mean()
        
        # Backprop and update
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

def train_zip_nrf_stage2(dataset, zip_nrf_cache, fast_cache, material_model, env_map, 
                         iterations, batch_size=1024, num_fast_samples=64, num_nerf_samples=16):
    """
    Stage 2: Joint training of material, environment, and caches.
    """
    # Optimizers with different learning rates
    lr_fast = 1.5625e-4
    lr_other = 3.125e-5
    
    optimizer_fast = Adam(fast_cache.parameters() + material_model.sampler.parameters(), lr=lr_fast)
    optimizer_other = Adam(material_model.parameters() + env_map.parameters(), lr=lr_other)
    
    for step in range(iterations):
        # Sample batch of primary rays
        rays = sample_rays(dataset, batch_size)
        
        # Estimate incoming radiance using caches
        # 1. Fast Cache estimation (64 samples)
        L_fast = fast_cache.estimate_incoming_radiance(rays, num_samples=num_fast_samples)
        
        # 2. Zip-NeRF Cache estimation (16 samples)
        # Assumes zip_nrf_cache provides a delta or refined estimate
        L_nerf = zip_nrf_cache.estimate_incoming_radiance(rays, num_samples=num_nerf_samples)
        
        # Combine estimates to form final estimator L_hat_i
        # Implementation detail: Combination logic depends on specific cache fusion strategy
        L_hat_i = combine_caches(L_fast, L_nerf)
        
        # Get ground truth image color
        I_gt = get_ground_truth_image_color(rays)
        
        # Photometric Loss with Gradient Trick
        # Term 1: (I_gt - L_hat_i)
        diff = I_gt - L_hat_i
        
        # Term 2: stopgrad(I_gt - L_hat_i)
        diff_stopgrad = diff.detach()
        
        # Term 3: stopgrad(L_hat_i_cache)
        # L_hat_i_cache is the cache estimate used for normalization/stabilization
        L_cache_stopgrad = L_hat_i.detach()
        
        # Loss formula
        loss = (2.0 * diff * diff_stopgrad / L_cache_stopgrad).mean()
        
        # Backprop and update
        optimizer_fast.zero_grad()
        optimizer_other.zero_grad()
        
        loss.backward()
        
        optimizer_fast.step()
        optimizer_other.step()
```

### Parameters
*   **Stage 1 Batch Size:** 16384 rays [S1].
*   **Stage 1 Iterations:** 100,000 [S1].
*   **Stage 1 Learning Rate:** Warmup 0 to 0.01 (10k iters), then linear decay to 0.001 (90k iters) [S1].
*   **Stage 2 Batch Size:** 1024 rays [S1].
*   **Stage 2 Iterations:** 40,000 (TensoIR-synthetic), 100,000 (Open Illumination) [S1].
*   **Stage 2 Learning Rate (Fast Cache & Sampler):** $1.5625 \times 10^{-4}$ [S1].
*   **Stage 2 Learning Rate (Material & Env Map):** $3.125 \times 10^{-5}$ [S1].
*   **Fast Cache Samples:** 64 secondary rays [S1].
*   **NeRF Cache Samples:** 16 secondary rays [S1].

### Edge Cases and Pitfalls
*   **Non-analytic Expectation:** The expected value of the radiance estimator $\mathbb{E}[\hat{L}_i]$ cannot be computed analytically. Direct minimization of the squared error between $I$ and $\mathbb{E}[\hat{L}_i]$ is impossible. The gradient trick must be used to obtain correct gradients in expectation [S1].
*   **Precision:** The use of `stopgrad` on the cache estimates is critical to prevent unstable gradients from the Monte Carlo sampling process from affecting the cache training during joint optimization [S1].

## Subtopics

### [Fast Cache (Control Variate Radiance Cache)](9552-fast-cache-control-variate-radiance-cache.md)
A computationally efficient approximation of incoming radiance using a lightweight hash grid and small MLP, serving as a control variate to correct bias in the NeRF cache.

### [NeRF-Casting Reflection Model](11495-nerf-casting-reflection-model.md)
A rendering procedure for specular reflections that casts reflected rays through a volume representation and uses a lightweight decoder, differing by explicitly approximating the reflection cone via a von Mises-Fisher distribution.

### [Conical Reflected Features](11502-conical-reflected-features.md)
Estimates expected volume-rendered features over a vMF distribution of reflected rays, avoiding Monte Carlo integration costs by modeling angular spread via conical sampling.

## Operating characteristics
*   **Time Complexity:** Stage 1 takes 5.5 hours on a single A100 for 100,000 iterations. Stage 2 takes 6.5 hours on a single A100 for 100,000 iterations [S1].
*   **Memory Complexity:** Depends on the size of the Zip-NeRF hash grid and the Fast Cache. Zip-NeRF uses a sparse structure, reducing memory compared to dense grids. Fast Cache uses a fixed, small number of samples (8 for control variate, 64 for estimation in joint training) [S1].
*   **Strengths:** Reduces bias in radiance caching by using a control variate (Fast Cache). Joint training allows for consistent optimization of material and environment.
*   **Limitations:** Requires significant GPU time (12+ hours total for full training). Relies on the quality of the initial Zip-NeRF cache.
*   **When to Use:** When high-fidelity inverse rendering with accurate specular reflections is required, and computational resources allow for multi-stage optimization.
*   **When Not to Use:** When real-time performance is required during training, or when hardware is limited (e.g., no A100-class GPU).

## Background
The Zip-NeRF Training Procedure was introduced in "Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering" [S1]. It builds upon the Zip-NeRF acceleration structure and addresses the bias issues in radiance caching for inverse rendering. The method integrates concepts from RawNeRF's loss function and gradient trick [S1].

## Sources
[S1] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
