# Recursive Control Variates for Inverse Rendering

Also called: CV-PS, CV-PSS, RCV, Recursive Control Variates

## Overview
Recursive Control Variates (RCV) is a gradient estimation technique for physically based differentiable inverse rendering that reduces variance and bias by leveraging information from previous optimization steps [S1]. It exploits the high correlation between consecutive renderings in gradient descent, where scene parameters change minimally, to recursively compound variance reduction across the optimization sequence [S1]. Unlike SAGA and SVRG which maintain a running approximation of the full gradient for machine learning pipelines, RCV handles variance from numerical integration over continuous domains, and unlike the optimal strategy of using all previous renderings as individual control variates, RCV uses only the immediate previous estimate to avoid growing covariance matrices [S1]. It effectively mitigates bias introduced by differentiating non-linear loss functions with respect to noisy inputs, preventing optimization drift [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current scene parameters $\theta_n$, reference image $I_{ref}$, previous control variate estimate $\langle F_{\theta} \rangle_{CV}^{n-1}$, and running statistics (mean, variance, covariance) estimated via exponentially weighted Welford’s algorithm [S1].
*   **Outputs:** Gradient estimate $\langle \partial_\theta L(F_\theta, I_{ref}) \rangle$ and updated control variate estimate $\langle F_{\theta} \rangle_{CV}^{n}$ [S1].

**Data Structures:**
*   **Running Statistics:** Per color channel: exponentially weighted mean $M_n$, variance numerator $S_n$, and covariance numerator $C_n$ [S1]. The variance of the control variate $\text{Var}[\langle F_{\theta} \rangle_{CV}]$ is tracked via a recurrence relation rather than direct estimation [S1].

**Algorithm Steps:**
1.  **Initialization:** For $n=0$, set $\langle F_{\theta} \rangle_{CV}^{0} := \langle F_{\theta} \rangle_{0}$ and initialize running statistics [S1].
2.  **Primal Rendering:** Render with $\theta_n$ using sample set $\mathcal{S}_n$ to obtain $\langle F_{\theta} \rangle_{n}$ [S1].
3.  **Re-evaluation:** Evaluate previous parameters $\theta_{n-1}$ using $\mathcal{S}_n$ to obtain $\langle F_{\theta} \rangle_{n}^{prev}$ [S1]. This uses either CV-PSS (reset RNG state to match previous iteration's second pass) or CV-PS (reuse light paths with BSDF adapters and a majorant for media) [S1].
4.  **Compute Control Weight:** Calculate $\alpha_n = \text{Cov}[\langle F_{\theta} \rangle_{n}, \langle F_{\theta} \rangle_{n}^{prev}] / (\text{Var}[\langle F_{\theta} \rangle_{n}^{prev}] + \text{Var}[\langle F_{\theta} \rangle_{CV}^{n-1}])$ [S1]. Clamp $\alpha_n \in [0, 1]$ [S1].
5.  **Compute Control Variate:** $\langle F_{\theta} \rangle_{CV}^{n} = \langle F_{\theta} \rangle_{n} + \alpha_n (\langle F_{\theta} \rangle_{CV}^{n-1} - \langle F_{\theta} \rangle_{n}^{prev})$ [S1].
6.  **Update Statistics:** Update running statistics using $\langle F_{\theta} \rangle_{n}$ and $\langle F_{\theta} \rangle_{n}^{prev}$ *after* computing $\langle F_{\theta} \rangle_{CV}^{n}$ to ensure $\alpha_n$ is statistically independent of $\langle F_{\theta} \rangle_{n-1}^{n-1}$, mitigating bias [S1]. Track $\text{Var}[\langle F_{\theta} \rangle_{CV}^{n}]$ via recurrence [S1].
7.  **Gradient Estimation:** Compute $\langle \partial_\theta L(F_\theta, I_{ref}) \rangle = \langle \partial_\theta F_\theta \rangle_{PRB} \cdot \partial L(\langle F_{\theta} \rangle_{CV}^{n}, I_{ref})$ [S1]. PRB estimates the forward gradient to maintain independence from the control variate term [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def recursive_control_variates_step(params_prev, params_curr, cv_prev, stats, ref_img, loss_fn, rng_state, majorant):
    # 1. Primal rendering with current parameters theta_n
    # Standard path tracing evaluation using sample set S_n
    primal_curr = render(params_curr, rng_state)
    
    # 2. Re-evaluate previous parameters theta_{n-1} with current samples S_n
    # CV-PSS: reset RNG state; CV-PS: reuse paths with BSDF adapters & majorant
    primal_prev_on_curr = render(params_prev, rng_state, reuse_paths=True, majorant=majorant)
    
    # 3. Compute control weight alpha_n
    cov = stats['cov']
    var_prev_primal = stats['var_prev_primal']
    var_prev_cv = stats['var_prev_cv']
    
    denom = var_prev_primal + var_prev_cv
    alpha = cov / denom if denom > 1e-8 else 0.0
    alpha = max(0.0, min(1.0, alpha))  # Clamp to [0, 1]
    
    # 4. Compute control variate estimate <F>_CV^n
    cv_curr = primal_curr + alpha * (cv_prev - primal_prev_on_curr)
    
    # 5. Update statistics (after computing cv_curr to avoid bias)
    # Exponentially weighted Welford update for mean and variance/covariance numerators [G]
    lambda_w = 0.99  # Exponential decay factor
    mean_prev = stats['mean']
    mean_curr = lambda_w * mean_prev + (1 - lambda_w) * primal_curr
    stats['mean'] = mean_curr
    
    diff_curr = primal_curr - mean_prev
    diff_prev = primal_prev_on_curr - mean_prev
    stats['var_prev_primal'] = lambda_w * stats['var_prev_primal'] + (1 - lambda_w) * diff_curr**2
    stats['cov'] = lambda_w * stats['cov'] + (1 - lambda_w) * diff_curr * diff_prev
    
    # Recurrence for CV variance: Var[<F>_CV^n] = Var[<F>_n] + alpha^2*Var[<F>_{n-1}^prev] - 2*alpha*Cov + alpha^2*Var[<F>_CV^{n-1}]
    var_curr_cv = stats['var_prev_primal'] + alpha**2 * stats['var_prev_primal'] - 2 * alpha * stats['cov'] + alpha**2 * var_prev_cv
    stats['var_prev_cv'] = var_curr_cv
    
    # 6. Gradient estimation
    # PRB gradient computation of forward rendering function [G]
    grad_forward = compute_gradient_prb(params_curr, rng_state)
    loss_deriv = loss_fn(cv_curr, ref_img)
    gradient = grad_forward * loss_deriv
    
    return gradient, cv_curr, stats
```

**Parameters:**
*   **Sample count per pixel:** The method allows using only a few samples per pixel in the primal rendering phase, as the control variate reduces the resulting noise [S1]. Prior work often required tens to hundreds of samples to alleviate bias [S1].
*   **Control weight $\alpha_n$:** Clamped to the range $[0, 1]$ to prevent negative correlation exploitation and aid convergence at the start of optimization [S1].
*   **Color channels:** Control weights are constructed separately per color channel [S1].

**Edge Cases and Pitfalls:**
*   **Bias from Correlated Weight:** The weight $\alpha_n$ is correlated with previous estimates, introducing bias. To mitigate this, update the running statistics *after* evaluating $\langle F_{\theta} \rangle_{CV}$ [S1].
*   **Bias from Non-linear Loss:** For non-L2 losses, $E[f(X)] \neq f(E[X])$ causes bias in the loss derivative [S1]. L2 loss is the only loss function that leads to unbiased derivative estimates in this context [S1].
*   **Bias from Multiplication:** The product in the gradient estimator is only unbiased if both factors are independent estimators. This is handled by estimating the first factor using PRB and the second using the control variate [S1].
*   **Geometric Optimization:** Moving or deforming surfaces are not supported. CV-PSS would lose correlation; CV-PS would require 'shift mappings' to keep path vertices valid [S1].
*   **Denoising:** Applying a denoiser to primal images adds bias that impedes optimization. Denoising the control variate estimator instead reduces bias and yields the best results [S1].

## Subtopics

### [Heuristic Control Weights for Recursive Control Variates](11578-heuristic-control-weights-for-recursive-control-variates.md)
A method for dynamically adjusting the influence of previous control variates to minimize bias and variance in gradient estimates. It addresses the trade-off between reusing historical rendering information and maintaining statistical independence from the current optimization step. It sets itself apart from static weighting schemes by adapting $\alpha_n$ based on running (co-)variance statistics, though it requires careful ordering of updates to avoid introducing bias.

## Operating characteristics
*   **Time Complexity:** The method adds a re-evaluation step per iteration. CV-PSS doubles the primal rendering cost (approximately 1/3 of the total PRB step cost) [S1]. CV-PS reduces primal rendering overheads from 2× to 1.6× and allows fusing computation into a single GPU kernel [S1].
*   **Memory Complexity:** Constant memory overhead, storing only the previous control variate estimate and running statistics per channel [S1]. This avoids the ever-growing covariance matrix required by the optimal strategy of using all previous renderings [S1].
*   **Measured Performance:** Primal rendering efficiency improves by a factor of up to 10 compared to prior work, leading to either significant speedup at equal reconstruction quality or higher quality after equal time [S1].
*   **Scaling:** When optimization reaches steady state, the recursive control variate becomes equivalent to rendering with a correspondingly larger sample count, implying unbounded noise reduction potential without proportional linear cost increase [S1].
*   **Strengths:** Reduces variance and bias; allows low sample counts; works with non-L2 losses by reducing bias; compatible with PRB for textured surfaces/homogeneous volumes and DRT for heterogeneous volumes [S1].
*   **Limitations:** Does not support geometric optimization; CV-PSS suffers from loss of correlation when parameters affect path geometry (roughness, IOR, extinction); bias remains due to correlation between weight and previous CV [S1].
*   **When to use:** Inverse rendering with textured surfaces and homogeneous/heterogeneous volumes where scene parameters change slowly between iterations [S1].
*   **When not to use:** Geometric optimization with deforming surfaces; scenarios where high correlation between steps is not present [S1].

## Background
Introduced by Baptiste Nicolet et al. in 2023 [S1]. It builds on the theory of control variates [Fieller and Hartley 1954] and previous work on variance reduction in rendering [Lafortune and Willems 1995a,b; Rousselle et al. 2016] [S1]. Implemented in Mitsuba 3, combined with PRB for textured surfaces/homogeneous volumes and DRT for heterogeneous volumes [S1]. It addresses the issue of noisy gradients in differentiable rendering, which often leads to drift in optimization, by exploiting the redundancy in consecutive gradient descent steps [S1].

## Sources
[S1] Recursive Control Variates for Inverse Rendering, https://research.nvidia.com/labs/rtr/publication/nicolet2023recursive/, fetched 2026-11-08
