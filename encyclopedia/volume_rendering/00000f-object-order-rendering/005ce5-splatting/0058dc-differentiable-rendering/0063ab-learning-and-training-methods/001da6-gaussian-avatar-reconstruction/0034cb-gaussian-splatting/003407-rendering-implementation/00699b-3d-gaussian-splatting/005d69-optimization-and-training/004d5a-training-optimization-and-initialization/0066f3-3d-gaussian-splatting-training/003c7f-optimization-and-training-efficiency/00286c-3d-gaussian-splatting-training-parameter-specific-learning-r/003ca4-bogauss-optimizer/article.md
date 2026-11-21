# BOGausS Optimizer
Also called: Unbiased Sparse Adam Optimizer

## Overview
BOGausS (Better Optimized Gaussian Splatting) is a modified Stochastic Gradient Descent optimizer designed to ensure unbiased state estimation for sparsely-viewed and split Gaussians in 3D Gaussian Splatting training [S1]. It addresses the performance degradation caused by biased moment estimates in standard Adam when Gaussians are occluded or rarely updated [S1]. It sets itself apart by using Gaussian-dependent time steps and precision-aware scaling factors derived from reconstruction confidence intervals, rather than global iteration counts and uniform learning rates [S1].

## How it works
**Inputs and Outputs:**
- Input: Gradient $\nabla_\theta \mathcal{L}_t$ for parameter $\theta$ at update step $t$; current state variables $\widehat{g}_{t-1}$, $\widehat{g}_{t-1}^2$, $t_{prev}$; learning rate $\eta_\theta$; Adam parameters $\beta_1, \beta_2$ (denoted $\beta_s^\star$); scaling factor $\Delta_\theta$ [S1].
- Output: Updated parameter $\theta_{t+1}$; updated state variables $\widehat{g}_t, \widehat{g}_t^2, t$ [S1].

**Data Structures:**
Each Gaussian primitive maintains optimizer state variables independent of the global iteration counter [S1]:
- $t$: Scalar integer, representing the Gaussian's specific lifespan (number of times it has been updated). Initialized to 0 [S1].
- $\widehat{g}$: First moment estimate (mean of gradients). Vector of same shape as $\theta$. Initialized to 0 [S1].
- $\widehat{g}^2$: Second moment estimate (mean of squared gradients). Vector of same shape as $\theta$. Initialized to 0 [S1].

**Parameters:**
- $\eta_\theta$: Learning rate for parameter $\theta$ [S1].
- $\beta_1, \beta_2$: Exponential decay rates for moment estimates (standard Adam values, e.g., 0.9, 0.999) [S1].
- $\epsilon$: Small constant for numerical stability (standard Adam value, e.g., $10^{-8}$) [G].
- $\Delta_\theta$: Scaling factor. For localization and size parameters, $\Delta_\theta$ is the confidence interval $\Delta_i$ derived from reconstruction confidence metrics [S1]. For all other parameters (e.g., opacity, spherical harmonics coefficients), $\Delta_\theta = 1$ [S1].
- $\alpha_t$: Inheritance factor for time during splitting [S1].
- $\alpha_g$: Inheritance factor for gradient states during splitting [S1].

**Algorithm Steps:**
1. **Compute Adaptive Coefficients:** Calculate the time-dependent coefficients $\alpha_{1,t}$ and $\alpha_{2,t}$ based on the Gaussian's updated lifespan $t \ge 1$:
   $$ \alpha_{s,t} = \max(1/t, 1 - \beta_s^\star) $$
   where $s \in \{1, 2\}$ [S1]. The coefficient uses the step count including the current update to avoid division by zero when $t=0$ [S1, G].
2. **Update First Moment:**
   $$ \widehat{g}_t = \widehat{g}_{t-1} + \alpha_{1,t}(\nabla_\theta \mathcal{L}_t - \widehat{g}_{t-1}) $$
   This is an exponential moving average update where the step size adapts to the Gaussian's age [S1]. (Note: The source contains a typographical error $\widehat{g}_{-1}$; it is corrected here to $\widehat{g}_{t-1}$ to match the recursive definition.)
3. **Update Second Moment:**
   $$ \widehat{g}_t^2 = \widehat{g}_{t-1}^2 + \alpha_{2,t}((\nabla_\theta \mathcal{L}_t)^2 - \widehat{g}_{t-1}^2) $$
   Similarly, this maintains an unbiased estimate of the second moment [S1].
4. **Parameter Update:**
   $$ \theta_{t+1} = \theta_t + \eta_\theta \Delta_\theta \frac{\widehat{g}_t}{\widehat{g}_t^2 + \epsilon} $$
   The update is scaled by $\Delta_\theta$ to account for the sensitivity of the parameter to changes [S1].
5. **Increment Lifespan:**
   $$ t \leftarrow t + 1 $$
   The Gaussian's specific time counter is incremented only if it was updated in this pass [S1].

**Edge Cases and Pitfalls:**
- **Initialization:** $\widehat{g}_0$ and $\widehat{g}_0^2$ must be exactly 0. $t$ must be 0 for new Gaussians [S1].
- **Splitting:** When a Gaussian splits, the child Gaussians inherit state. The time $t$ and gradient states are scaled by inheritance factors $\alpha_t$ and $\alpha_g$ to maintain unbiasedness across the split [S1].
- **Parameterization:** Size parameters $\sigma_a$ must be linearly parameterized ($\theta_\sigma = \sigma$). Using logarithmic parameterization ($\theta_\sigma = \log \sigma$) is incompatible with the precision-aware update $\Delta_\theta$ and leads to instability (exploding sizes) in outdoor scenes [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bogauss_update(param, grad, state, config):
    """
    param: current parameter value (tensor)
    grad: gradient of loss w.r.t param (tensor)
    state: dict containing 't', 'g', 'g2' for this specific Gaussian/param group
    config: dict containing 'eta', 'beta1', 'beta2', 'eps', 'delta'
    """
    t = state['t']
    g = state['g']
    g2 = state['g2']
    
    # Increment t for the current update step (k = t + 1)
    k = t + 1
    
    # Calculate adaptive coefficients using updated lifespan to avoid 1/0
    # alpha_s_t = max(1/k, 1 - beta_s)
    alpha1 = max(1.0 / k, 1.0 - config['beta1'])
    alpha2 = max(1.0 / k, 1.0 - config['beta2'])
    
    # Update first moment estimate
    # g_t = g_{t-1} + alpha1 * (grad - g_{t-1})
    g = g + alpha1 * (grad - g)
    
    # Update second moment estimate
    # g2_t = g2_{t-1} + alpha2 * (grad^2 - g2_{t-1})
    g2 = g2 + alpha2 * (grad * grad - g2)
    
    # Parameter update
    # theta_{t+1} = theta_t + eta * delta * g_t / (g2_t + eps)
    update = config['eta'] * config['delta'] * g / (g2 + config['eps'])
    param = param + update
    
    # Update state
    state['t'] = k
    state['g'] = g
    state['g2'] = g2
    
    return param, state
```

## Subtopics

### [SparseGaussianAdam](13467-sparsegaussianadam.md)
SparseGaussianAdam is a sparse variant of the Adam optimizer designed specifically for 3D Gaussian Splatting training. It addresses the computational and memory inefficiency of standard Adam by applying parameter updates only to Gaussians that are visible in the current rendering pass, thereby skipping updates for occluded or out-of-view primitives. It builds on [BOGausS Optimizer](9298-bogauss-optimizer.md) by integrating visibility-aware state management into the Adam update logic.

## Operating characteristics
- **Time Complexity:** Per-update complexity is $O(1)$ per active Gaussian parameter, similar to Adam [G].
- **Memory Complexity:** $O(N \cdot D)$ where $N$ is the number of Gaussians and $D$ is the dimensionality of the parameters being optimized [S1]. BOGausS requires maintaining per-Gaussian state variables ($t, \widehat{g}, \widehat{g}^2$), increasing memory overhead compared to a global optimizer state [S1].
- **Strengths:** Prevents performance decrease and improves convergence for sparsely viewed Gaussians by ensuring unbiased state estimation [S1]. Adapts to scene content and camera setup via confidence intervals [S1]. Reduces artifacts like exploding sizes in outdoor scenes by using linear parameterization and precision-aware updates [S1].
- **Limitations:** Higher memory footprint due to per-Gaussian state tracking [S1]. Requires careful handling of state inheritance during Gaussian splitting [S1]. Not given in the sources: specific scaling with resolution or number of Gaussians beyond the linear memory cost.

## Background
BOGausS was proposed in the paper "BOGausS: Better Optimized Gaussian Splatting" [S1]. It builds on the observation that standard Adam's biased moment estimates degrade performance when Gaussians are occluded or rarely updated (sparse viewing) [S1]. The method draws inspiration from Variational Bayesian Inference to define confidence metrics on geometry reconstruction, which are then used to adapt the optimizer [S1]. It also introduces a density-preserving Gaussian splitting mechanism that allows smooth loss transition [S1].

## Sources
[S1] BOGausS: Better Optimized Gaussian Splatting, https://doi.org/10.23919/eusipco63237.2025.11226626, fetched 2026-10-26
