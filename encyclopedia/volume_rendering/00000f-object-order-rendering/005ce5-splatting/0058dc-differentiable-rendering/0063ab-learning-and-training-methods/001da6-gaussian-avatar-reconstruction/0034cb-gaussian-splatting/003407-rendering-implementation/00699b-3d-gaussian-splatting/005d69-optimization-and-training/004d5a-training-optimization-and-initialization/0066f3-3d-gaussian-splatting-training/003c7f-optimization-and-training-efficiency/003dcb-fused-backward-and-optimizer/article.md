# Fused Backward and Optimizer

Also called: Fused Adam Update, Fused optimizer-backward pass, Fused updates, Fusing Backward and Optimizer

## Overview
Fused Backward and Optimizer is a training optimization technique for 3D Gaussian Splatting that integrates the Adam optimizer step directly into the backward pass of the differentiable rasterizer [S1, unofficial]. It solves the bottleneck where parameter updates consume 40–60% of training time by eliminating separate optimizer kernels and reducing VRAM usage through the avoidance of intermediate parameter buffers [S1, unofficial]. It differs from sibling methods like [FastGS](9177-fastgs.md) by prioritizing memory efficiency and correctness over maximum speed, whereas [FastGS](9177-fastgs.md) and [CLM](7351-clm.md) focus on skipping computations for invisible Gaussians to achieve higher speedups at the cost of potential quality degradation or inconsistency [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ computed by the rasterizer's backward pass; current first and second moment estimates $m_t, v_t$ for Adam; hyperparameters $\alpha$ (learning rate), $\beta_1, \beta_2, \epsilon$.
- **Outputs:** Updated Gaussian parameters $\theta_{t+1}$; updated moments $m_{t+1}, v_{t+1}$.

**Data Structures:**
- **Gaussian Parameters:** Stored in contiguous GPU memory. Fields include position, covariance (6 values), opacity, and spherical harmonics coefficients.
- **Adam Moments:** Two buffers per parameter set, matching the size of the parameters, storing $m_t$ and $v_t$.
- **Gradients:** Buffer matching parameter size, populated by the rasterizer backward pass.

**Numbered Steps:**
1. **Gradient Computation:** The differentiable rasterizer computes pixel colors and gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ for all visible Gaussians. For Gaussians outside the viewing frustum, gradients are explicitly set to zero [S1, unofficial].
2. **Moment Loading:** Load current first and second moment estimates $m_t$ and $v_t$ from global memory into registers or shared memory for the active Gaussians [S1, unofficial].
3. **Fused Update Kernel:** Launch a single CUDA kernel that performs the Adam update logic for all parameters in parallel:
   - Compute biased first moment: $m_{t+1} = \beta_1 m_t + (1 - \beta_1) \frac{\partial \mathcal{L}}{\partial \theta}$ [S1, unofficial].
   - Compute biased second moment: $v_{t+1} = \beta_2 v_t + (1 - \beta_2) \left(\frac{\partial \mathcal{L}}{\partial \theta}\right)^2$ [S1, unofficial].
   - Compute bias-corrected estimates: $\hat{m}_{t+1} = \frac{m_{t+1}}{1 - \beta_1^{t+1}}$, $\hat{v}_{t+1} = \frac{v_{t+1}}{1 - \beta_2^{t+1}}$ [S1, unofficial].
   - Update parameters: $\theta_{t+1} = \theta_t - \alpha \frac{\hat{m}_{t+1}}{\sqrt{\hat{v}_{t+1}} + \epsilon}$ [S1, unofficial].
   - Write updated moments $m_{t+1}, v_{t+1}$ and parameters $\theta_{t+1}$ back to global memory [S1, unofficial].
4. **Memory Management:** No intermediate buffers for raw parameters are allocated during the update, as updates are written directly to the parameter storage [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fused_adam_backward(grads, params, m, v, alpha, beta1, beta2, eps, t):
    """
    Fuses Adam update into the backward pass.
    grads: Tensor of gradients dL/dtheta.
    params: Tensor of current parameters theta_t.
    m: Tensor of first moment estimates m_t.
    v: Tensor of second moment estimates v_t.
    alpha: Learning rate.
    beta1, beta2: Adam decay rates.
    eps: Adam epsilon.
    t: Current training step index (1-based).
    """
    # Compute bias correction terms
    bias1 = 1.0 - beta1 ** t
    bias2 = 1.0 - beta2 ** t
    
    # Update first moment
    m_new = beta1 * m + (1.0 - beta1) * grads
    
    # Update second moment
    v_new = beta2 * v + (1.0 - beta2) * (grads ** 2)
    
    # Bias-corrected moments
    m_hat = m_new / bias1
    v_hat = v_new / bias2
    
    # Parameter update
    params_new = params - alpha * (m_hat / (sqrt(v_hat) + eps))
    
    # Write back
    m[:] = m_new
    v[:] = v_new
    params[:] = params_new
```

**Parameters:**
- $\alpha$: Learning rate (typically $1.6 \times 10^{-2}$ for position, $1.0 \times 10^{-3}$ for other parameters in standard 3DGS, but specific values for the fused variant are not explicitly differentiated in the source) [S1, unofficial].
- $\beta_1, \beta_2$: Adam momentum coefficients (typically $0.9, 0.999$) [S1, unofficial].
- $\epsilon$: Adam epsilon (typically $10^{-15}$) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Gradients:** To maintain correctness with respect to the original Adam step, parameter updates must be performed even for parameters receiving a gradient of zero (e.g., Gaussians outside the viewing frustum) [S1, unofficial]. Skipping these updates breaks the moment estimation consistency.
- **Performance Trade-off:** While this approach reduces VRAM, it does not skip computations for invisible Gaussians, limiting the speedup compared to methods that do skip them [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of Gaussian parameters. The fused kernel reduces kernel launch overhead and memory bandwidth pressure by combining gradient accumulation and update into one pass.
- **Memory Complexity:** $O(N)$ for parameters and moments. VRAM is reduced because no additional buffers for intermediate parameter states are required during the update step [S1, unofficial].
- **Measured Performance:** On Mip-NeRF360 outdoor scenes with an RTX 4090, training time reduced from 5m31s to 5m04s, and VRAM usage decreased from 6.0GiB to 5.6GiB, with PSNR increasing slightly from 24.72 to 24.73 [S1, unofficial].
- **Strengths:** Significant VRAM reduction, especially beneficial for large scenes with many Gaussians. Maintains strict numerical equivalence with the standard Adam optimizer.
- **Limitations:** Attainable performance improvements are smaller compared to methods that skip updates for invisible Gaussians. After integration, parameter updates remain a significant bottleneck, consuming a large fraction of the remaining runtime [S1, unofficial].
- **When to Use:** When VRAM is the primary constraint or when strict adherence to the original 3DGS optimization trajectory is required without risking quality degradation from skipped updates.
- **When Not to Use:** When maximum training speed is the priority and slight quality inconsistencies are acceptable; in such cases, skipping updates for invisible Gaussians (as in [FastGS](9177-fastgs.md)) is preferred.

## Background
The method was introduced in "Faster-GS: Analyzing and Improving Gaussian Splatting Optimization" [S1, unofficial]. Profiling of the original 3DGS implementation revealed that the Adam update routine consumed 40–60% of total training time [S1, unofficial]. The authors proposed fusing the optimizer step into the rasterizer's backward pass to alleviate this bottleneck [S1, unofficial]. The approach is presented as a foundational optimization that can be combined with other techniques like skipping invisible Gaussian updates, though the latter is noted as an optional extension that may introduce inconsistencies [S1, unofficial].

## Sources
[S1] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
