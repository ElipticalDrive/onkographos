# Adam Optimisation with Adaptive Scaling

Also called: Fused Adam with Adaptive Scaling, PyTorch Adam with Adaptive Scaling

## Overview
Adam Optimisation with Adaptive Scaling is a parameterized training configuration for differentiable volume rendering (specifically Gaussian Splatting) that applies per-parameter learning rate multipliers and dimension-dependent scaling to the standard Adam optimizer [S1, unofficial]. It solves the problem of disparate convergence speeds between geometric parameters (positions, shapes) and photometric parameters (amplitudes, colors), preventing divergence in obscured regions and stabilizing high-dimensional fits [S1, unofficial][S2]. It sets itself apart from standard Adam by explicitly compensating for gradient dilution and using fused CUDA kernels to reduce memory overhead [S1, S3, unofficial].

## How it works
**Inputs and Outputs:**
- Inputs: Model parameters $\theta$ (positions, covariances/Cholesky factors, opacities, spherical harmonic coefficients), gradients $\nabla_\theta L$, current iteration $t$, hyperparameters.
- Outputs: Updated parameters $\theta_{t+1}$.

**Data Structures:**
- Adam moments: First moment $m_t$ and second moment $v_t$ for each parameter tensor.
- Parameter tensors: Stored in fixed-size pools to maintain constant tensor shapes during dynamic capacity changes (e.g., splitting/merging Gaussians) [S1, unofficial].

**Parameters:**
- Adam defaults: $\beta_1 = 0.9$, $\beta_2 = 0.999$, $\epsilon = 10^{-8}$ [S1, unofficial].
- Base learning rate: $\eta = 0.01$ [S1, unofficial].
- Reduce-on-plateau schedule: Patience 15 iterations, factor 0.9, threshold $10^{-3}$, floor $10^{-8}$ [S1, unofficial].
- Early stopping: Patience 500 iterations on regularized training loss [S1, unofficial].
- Iteration cap: 20,000 iterations [S1, unofficial].

**Method Steps:**

1. **Compute Gradients:** Calculate gradients $\nabla_\theta L$ for the current batch of views. If using fused optimization, this is combined with the Adam update in a single kernel [S3, unofficial].

2. **Apply Gradient Dilution Compensation:** Adjust the base learning rate based on the number of splats contributing to the residual. For progressive fitting residual passes, use $\eta = 0.03$ [S1, unofficial].

3. **Calculate Effective Learning Rate Scaling:**
   Determine the scaling factor based on the dimensionality $D$ of the spatial representation (e.g., $D=3$ for 3D Gaussians) [S1, unofficial]:
   $$ P_D = D + \frac{D(D+1)}{2} $$
   where $P_D$ is the number of position and Cholesky parameters per splat (5 for 2D, 9 for 3D) [S1, unofficial].
   The base scaling factor is:
   $$ S_{base} = \frac{P_D}{P^2} $$
   where $P$ is the total number of parameters per splat (typically 15 for 3D: 3 pos + 6 cov + 1 opac + 5 SH degree 0 + 12 SH degree 1? Note: Source defines $P_D$ specifically for pos/Cholesky. The formula $P_D/P^2$ implies normalization by total parameter count squared or similar. Source states "scaled by $P_D / P^2$". We follow source notation exactly) [S1, unofficial].
   
   If $D \ge 4$, apply an additional empirical factor:
   $$ S_{dim} = D^{0.8} $$
   $$ S_{total} = S_{base} \times S_{dim} $$
   For $D < 4$, $S_{dim} = 1$ [S1, unofficial].

4. **Apply Per-Parameter Multipliers:**
   Multiply the scaled learning rate by specific factors for different parameter groups [S1, unofficial]:
   - **Amplitudes (e.g., SH coefficients, Opacity):** $\eta_{amp} = 3 \times \eta_{scaled}$
   - **Positions:** $\eta_{pos} = 1.5 \times \eta_{scaled}$
   - **Other parameters (Shapes/Covariances):** $\eta_{other} = \eta_{scaled}$ (implied, as only amp and pos are explicitly boosted) [S1, unofficial].

5. **Update Adam Moments:**
   For each parameter group $g$:
   $$ m_t = \beta_1 m_{t-1} + (1 - \beta_1) \nabla_g L $$
   $$ v_t = \beta_2 v_{t-1} + (1 - \beta_2) (\nabla_g L)^2 $$
   Bias correction:
   $$ \hat{m}_t = \frac{m_t}{1 - \beta_1^t}, \quad \hat{v}_t = \frac{v_t}{1 - \beta_2^t} $$
   Parameter update:
   $$ \theta_g \leftarrow \theta_g - \eta_g \frac{\hat{m}_t}{\sqrt{\hat{v}_t} + \epsilon} $$

6. **Handle Dynamic Capacity (Fixed-Pool Relocation):**
   If splats are split or merged, relocate the optimizer states ($m, v$) to the new parameter indices in the fixed pool. Do not reset states to zero for existing splats to preserve optimization history [S1, unofficial].

7. **State Inheritance (On Splitting):**
   When a Gaussian splits into children, inherit Adam states from the parent (mother) using fading factors [S2]:
   $$ t_{child} = \alpha_t \times t_{mother} $$
   $$ (g_{child})_t = \alpha_g \times (g_{mother})_t $$
   $$ (g_{child}^2)_t = (\alpha_g)^2 \times (g_{mother}^2)_t $$
   where $\alpha_t$ is the lifespan fading factor and $\alpha_g$ is the gradient fading factor [S2].

8. **Out-of-Core Management (If Applicable):**
   If using out-of-core optimization, discard Adam moments for blocks evicted from VRAM. Re-admitted blocks cold-start their moments [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adam_optimize_with_adaptive_scaling(
    params: dict, 
    grads: dict, 
    state: dict, 
    iteration: int, 
    dim: int = 3
) -> dict:
    """
    params: dict of parameter tensors (positions, amplitudes, etc.)
    grads: dict of gradient tensors
    state: dict containing 'm' (first moment), 'v' (second moment), 'lr' (current lr)
    iteration: current training iteration
    dim: spatial dimensionality (default 3)
    """
    beta1 = 0.9
    beta2 = 0.999
    eps = 1e-8
    base_lr = 0.01
    
    # 1. Gradient Dilution Compensation (simplified: assume residual pass check is external)
    # In standard mode: lr = base_lr. In residual pass: lr = 0.03.
    # We assume 'state' holds the current effective base lr.
    current_lr = state['lr']
    
    # 2. Effective Rate Scaling
    # P_D = D + D*(D+1)/2
    P_D = dim + (dim * (dim + 1)) / 2
    # P is total parameters per splat. Source uses P^2 in denominator.
    # For 3D Gaussian: 3 pos + 6 cov + 1 opac + 5 SH0 + 12 SH1 = 27? 
    # Source does not define P explicitly, only P_D. We use P_D/P^2 as stated.
    # Assuming P is known from model definition.
    P = 27 # Example value for 3DGS, not given in source. [G]
    scale_factor = P_D / (P * P)
    
    if dim >= 4:
        scale_factor *= (dim ** 0.8)
        
    scaled_lr = current_lr * scale_factor
    
    # 3. Per-parameter updates
    for name, param in params.items():
        grad = grads[name]
        
        # Initialize moments if new
        if name not in state['m']:
            state['m'][name] = torch.zeros_like(param)
            state['v'][name] = torch.zeros_like(param)
            
        m = state['m'][name]
        v = state['v'][name]
        
        # Bias correction
        bias_correction1 = 1 - beta1 ** iteration
        bias_correction2 = 1 - beta2 ** iteration
        
        # Update moments
        m = beta1 * m + (1 - beta1) * grad
        v = beta2 * v + (1 - beta2) * (grad * grad)
        
        m_hat = m / bias_correction1
        v_hat = v / bias_correction2
        
        # Determine learning rate multiplier
        if 'amplitude' in name or 'sh' in name or 'opacity' in name:
            lr = scaled_lr * 3.0
        elif 'position' in name:
            lr = scaled_lr * 1.5
        else:
            lr = scaled_lr
            
        # Update parameter
        param = param - lr * (m_hat / (torch.sqrt(v_hat) + eps))
        
        # Store back
        state['m'][name] = m
        state['v'][name] = v
        params[name] = param
        
    return params, state
```

## Subtopics

### [Weighted Adam Optimizer](9555-weighted-adam-optimizer.md)
An optimization update rule that modifies the standard Adam algorithm by weighting the gradient and momentum terms with a scalar weight $\bar{w}_i \in [0,1]$ representing the average visibility or informativeness of the gradient for iteration $i$. This prevents divergence when primitives are obscured or have many keyframes [9555-weighted-adam-optimizer.md].

### [Custom Adam Implementation](9579-custom-adam-implementation.md)
A custom Adam optimizer implementation that precisely matches PyTorch's behavior while optimizing away unnecessary overhead, exploiting CUDA kernel fusion, fast math operations, and fused-multiply additions [9579-custom-adam-implementation.md].

## Operating characteristics
- **Time Complexity:** $O(N)$ per iteration, where $N$ is the number of parameters. Fused kernels reduce overhead [S3, unofficial].
- **Memory Complexity:** $O(N)$ for storing first and second moments. Fixed-pool relocation keeps tensor shapes constant, avoiding reallocation costs [S1, unofficial]. Out-of-core variants trade memory for I/O latency [S4, unofficial].
- **Performance:** Stabilizes convergence in high-dimensional spaces ($D \ge 4$) via the $D^{0.8}$ factor [S1, unofficial]. Prevents divergence in obscured regions via gradient weighting [9555-weighted-adam-optimizer.md].
- **Strengths:** Handles disparate convergence rates (amplitudes vs. positions); robust to dynamic topology changes; memory-efficient via fusion and out-of-core strategies.
- **Limitations:** Requires careful tuning of fading factors ($\alpha_t, \alpha_g$) for state inheritance [S2]; high block churn in out-of-core settings can degrade convergence due to cold starts [S4, unofficial].

## Background
Originated in the context of Gaussian Splatting for volume rendering and microscopy data visualization [S1, unofficial]. Developed to address the instability of standard Adam when applied to the highly non-convex, dynamic optimization landscape of 3D Gaussians [S1, unofficial][S2]. Extended by subsequent works to include fused kernels for performance [S3, unofficial] and out-of-core strategies for billion-scale primitives [S4, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S2] BOGausS: Better Optimized Gaussian Splatting, https://doi.org/10.23919/eusipco63237.2025.11226626, fetched 2026-10-26
[S3] VkSplat: High-Performance 3DGS Training in Vulkan Compute, https://doi.org/10.2312/egs.20261024, fetched 2026-10-29 (unofficial)
[S4] TideGS: Scalable Training of Over One Billion 3D Gaussian Splatting Primitives via Out-of-Core Optimization, https://doi.org/10.48550/arxiv.2605.20150, fetched 2026-10-26 (unofficial)
