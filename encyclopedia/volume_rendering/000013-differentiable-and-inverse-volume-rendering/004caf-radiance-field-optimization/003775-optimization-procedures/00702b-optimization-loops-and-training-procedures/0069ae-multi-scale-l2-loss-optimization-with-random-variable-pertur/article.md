# Multi-scale L2 Loss Optimization with Random Variable Perturbation

Also called: Multi-scale L2 Loss Optimization, Optimization, Ours, Pyramid L2 Loss, Resolution Alternating Loss

## Overview
This method optimizes shader parameters to match a target image by minimizing a multi-scale L2 loss while using random variable perturbation to escape local minima [S1]. It is specifically designed for differentiable rendering pipelines where parameters (such as Dirac parameters) define geometry or appearance, and the rendering output is compared against a ground-truth image [S1]. Unlike methods that rely on a single resolution or fixed parameter ordering, this approach alternates between low-resolution and full-resolution loss objectives and perturbs non-ray-marching parameters to smooth the optimization landscape, particularly for discontinuous or binary targets [S1]. It differs from [NeRF Training Loop](970-nerf-training-loop.md) by focusing on explicit shader parameter optimization rather than neural field weights, and from [Optimization Configuration (Adam + Exponential Decay + Selective L2)](2133-optimization-configuration-adam-exponential-decay-selective-.md) by explicitly introducing stochastic perturbation to handle flat landscapes caused by sparse gradient signals [S1].

## How it works
The method optimizes a set of shader parameters $\theta$ to minimize the difference between a rendered image $\hat{I}$ and a target image $I_{target}$. The optimization is performed using a differentiable rendering operator that supports both forward and backward passes [S1].

### Inputs and Outputs
- **Inputs:**
  - Target image $I_{target}$ (RGB, resized from source).
  - Initial shader program with parameterized primitives (e.g., rings, knots).
  - Initial parameter values $\theta_0$ (randomly initialized).
- **Outputs:**
  - Optimized shader program with optimized parameters $\theta^*$.
  - GLSL code with unused computations removed.

### Data Structures
- **Parameter Vector $\theta$:** Contains all shader parameters.
  - **Dirac Parameters:** Parameters not dependent on ray marching (e.g., position, radius, color, stroke width). These are subject to random perturbation.
  - **Ray-Marching Parameters:** Parameters dependent on the ray path (e.g., Z-ordering that varies per pixel). These are *not* perturbed.
- **Random Variable Scales $\sigma$:** A vector of scales corresponding to each Dirac parameter. These are optimized alongside $\theta$.
- **Resolution Pyramid:** A list of resolutions $R_0, R_1, \dots, R_k$ where $R_0$ is the lowest resolution and $R_k$ is the target rendering resolution.

### Algorithm Steps
1. **Initialization:**
   - Initialize $\theta$ with 100 random restarts [S1].
   - Initialize random variable scales $\sigma$ to small positive values.

2. **Optimization Loop (per restart):**
   - Run for 2000 iterations [S1].
   - Within each restart, alternate the loss objective 5 times [S1].
   - **Loss Alternation:**
     - For each alternation step $j \in \{0, \dots, 4\}$:
       - Define current resolution $R = R_j$.
       - Compute the multi-scale L2 loss $\mathcal{L}$:
         $$
         \mathcal{L} = \sum_{r=0}^{j} \text{L2}(\text{Downsample}(I_{render}, R_r), \text{Downsample}(I_{target}, R_r))
         $$
         where $\text{Downsample}(I, R)$ reduces the image to resolution $R$. The sum includes all resolutions from the lowest ($R_0$) up to the current resolution $R_j$ [S1].
       - Perform gradient descent step on $\theta$ and $\sigma$ using $\mathcal{L}$ [S1].
   - **Random Variable Perturbation:**
     - For every Dirac parameter $\theta_i$ (not dependent on ray marching), add a random variable $\epsilon_i$:
       $$
       \theta_i^{perturbed} = \theta_i + \epsilon_i, \quad \epsilon_i \sim \mathcal{U}(-\sigma_i, \sigma_i)
       $$
       where $\mathcal{U}$ is the uniform distribution and $\sigma_i$ is the optimized scale for parameter $i$ [S1].
     - The scales $\sigma_i$ are updated via gradient descent to minimize the loss. Upon convergence, $\sigma_i$ typically approaches zero [S1].

3. **Post-Optimization (Dead Code Elimination):**
   - The compiler outputs the shader program with optimized parameters to a GLSL backend [S1].
   - During optimization, computations for extra primitives (added to avoid local minima) are wrapped in an `EliminateIfUnused()` annotation [S1].
   - Upon convergence, the compiler progressively removes computations within `EliminateIfUnused()` if they do not increase the optimization error [S1].
   - This effectively prunes primitives that are pushed outside the image or occluded by background color [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_shader(target_image, initial_params, num_restarts=100, iterations_per_restart=2000, num_alternations=5):
    best_params = None
    best_loss = infinity
    
    for restart in range(num_restarts):
        # Initialize parameters and random variable scales
        theta = random_initialize(initial_params)
        sigma = random_initialize_scales(theta)
        
        # Define resolution pyramid (e.g., [H/8, H/4, H/2, H])
        resolutions = get_resolution_pyramid(target_image.shape)
        
        for iteration in range(iterations_per_restart):
            # Alternate loss objectives
            for alt_step in range(num_alternations):
                # Current resolution index
                r_idx = alt_step
                current_res = resolutions[r_idx]
                
                # Compute multi-scale L2 loss up to current resolution
                loss = 0.0
                for r in range(r_idx + 1):
                    res = resolutions[r]
                    # Downsample target and rendered image to resolution res
                    target_ds = downsample(target_image, res)
                    # Render with perturbed parameters
                    perturbed_theta = apply_random_perturbation(theta, sigma)
                    rendered_ds = downsample(render(perturbed_theta, res), res)
                    loss += l2_loss(rendered_ds, target_ds)
                
                # Compute gradients and update parameters
                gradients_theta, gradients_sigma = compute_gradients(loss, theta, sigma)
                theta = update(theta, gradients_theta)
                sigma = update(sigma, gradients_sigma)
        
        # Evaluate final loss without perturbation
        final_loss = l2_loss(render(theta), target_image)
        
        if final_loss < best_loss:
            best_loss = final_loss
            best_params = theta
            
    # Post-processing: Dead code elimination
    optimized_shader = compile_shader(best_params)
    optimized_shader = eliminate_unused_primitives(optimized_shader, target_image)
    
    return optimized_shader
```

### Parameters
- **Number of restarts:** 100 [S1].
- **Iterations per restart:** 2000 [S1].
- **Loss alternation repetitions:** 5 times per restart [S1].
- **Random variable distribution:** Uniform $\mathcal{U}(-\sigma_i, \sigma_i)$ [S1].

### Edge Cases and Pitfalls
- **Local Minima:** The optimization landscape can be flat, especially for binary or high-contrast targets (e.g., Celtic Knot) where gradients are sparse and noisy [S1]. The random variable perturbation alleviates this by generating a "fuzzy" rendering, increasing the number of pixels with differently branched neighbors and allowing more frequent sampling of discontinuities [S1].
- **Convergence of Scales:** The scales of the random variables $\sigma_i$ should converge close to zero. If they do not, the perturbation may prevent precise convergence [S1].
- **Flat Landscape:** In cases with poor alignment, the gradient contribution is noisy, leading to a flat landscape. The method's success relies on the random variables to escape this flatness [S1].
- **Ray-Marching Parameters:** Random perturbation is *not* applied to parameters dependent on ray marching (e.g., parametric Z-ordering) to maintain geometric consistency [S1].

## Operating characteristics
- **Time Complexity:** The method is computationally intensive due to multiple restarts and iterations. However, it is significantly faster than baseline methods like FD* and SPSA* [S1].
  - **Olympic Rings (10 rings):** Median success time 1.4s, Expected time to success 5.0s [S1].
  - **Celtic Knot:** Median 2.3s, Expected 17.3s [S1].
  - **SIGGRAPH Logo:** Median 2.6s, Expected 6.1s [S1].
  - **TF RayMarch:** Median 0.7s, Expected 40.3s [S1].
  - **TF RayCast:** Median 4.3s, Expected 13.9s [S1].
- **Memory Complexity:** Not explicitly given, but the method uses standard differentiable rendering memory overhead [S1].
- **Scaling:** The method scales with the complexity of the shader. For example, the 5-ring Olympic Rings variant has a median success time 0.5x that of the 10-ring variant, but converges less often (Expected time 1.6x higher) due to faster runtime but lower convergence rate [S1].
- **Strengths:**
  - Effectively escapes local minima in flat optimization landscapes [S1].
  - Fast convergence compared to gradient-free baselines (FD*, SPSA*) [S1].
  - Produces clean GLSL code via dead code elimination [S1].
- **Limitations:**
  - Requires multiple restarts (100) to ensure convergence [S1].
  - Performance degrades on targets with sparse gradients (e.g., black and white images) without the random variable perturbation [S1].
  - The ablation without random variables (O/wo) fails to converge on challenging targets like Celtic Knot [S1].

## Background
The method was introduced in the paper "A δ" [S1]. It is designed for differentiable rendering pipelines where users define parametric shapes (e.g., rings, knots) and optimize their parameters to match target images. The method addresses the challenge of optimizing discrete or discontinuous structures where traditional gradient-based methods struggle due to sparse gradients and local minima [S1]. It is particularly useful for artistic and illustrative workflows where manual animation is difficult due to the high dimensionality of parameters [S1]. The method allows for parametric Z-ordering, unlike DVG which requires a single Z value per shape [S1].

## Sources
[S1] A δ, https://doi.org/10.1145/3528223.3530125, fetched 2026-10-30
