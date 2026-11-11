# Recursive Control Variates for Inverse Rendering

Also called: CV-PS, CV-PSS, RCV

## Overview
Recursive Control Variates (RCV) is a gradient estimation technique for physically based differentiable inverse rendering that reduces variance and bias by leveraging information from previous optimization steps [S1]. It exploits the high correlation between consecutive renderings in gradient descent, where scene parameters change minimally, to recursively compound variance reduction across the optimization sequence [S1]. Unlike standard variance reduction methods that maintain a full covariance matrix or require independent re-sampling, RCV uses only the immediate previous estimate, approximating the optimal weighting in steady state with constant memory overhead [S1]. It is particularly effective for reducing bias introduced by non-linear loss functions applied to noisy inputs, preventing optimization drift [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current scene parameters $\theta_n$, reference image $I_{ref}$, previous control variate estimate $\langle F_{\theta} \rangle_{CV}^{n-1}$, previous primal estimate $\langle F_{\theta} \rangle_{n-1}$, and running statistics (mean, variance, covariance) estimated via exponentially weighted Welford’s algorithm.
*   **Outputs:** Gradient estimate $\langle \partial_\theta L(F_\theta, I_{ref}) \rangle$ and updated control variate estimate $\langle F_{\theta} \rangle_{CV}^{n}$.

**Data Structures:**
*   **Running Statistics:** Three scalar values per color channel (or per pixel if per-pixel statistics are used, though the source implies global or channel-wise for the weight $\alpha_n$) stored for the estimator sequence:
    *   $M_n$: Exponentially weighted mean of $\langle F \rangle$.
    *   $S_n$: Exponentially weighted sum of squared differences (variance numerator).
    *   $C_n$: Exponentially weighted covariance numerator between current and previous estimates.
    *   *Note:* The source specifies estimating statistics using an exponentially weighted variant of Welford’s algorithm [S1].

**Algorithm Steps:**

1.  **Initialization:**
    For the first iteration ($n=0$), initialize the control variate as the primal rendering:
    $$ \langle F_{\theta} \rangle_{CV}^{0} := \langle F_{\theta} \rangle_{0} $$
    Initialize running statistics $M_0, S_0, C_0$ based on $\langle F_{\theta} \rangle_{0}$ [S1].

2.  **Primal Rendering (Current Step):**
    Render the scene with current parameters $\theta_n$ using a set of Monte Carlo samples $\mathcal{S}_n$ to obtain the noisy primal estimate $\langle F_{\theta} \rangle_{n}$ [S1].

3.  **Re-evaluation (Control Variate Source):**
    Evaluate the *previous* scene state (parameters $\theta_{n-1}$) using the *current* sample set $\mathcal{S}_n$ to obtain $\langle F_{\theta} \rangle_{n}^{prev}$. This requires a sample reuse strategy:
    *   **CV-PSS (Primary Sample Space):** Reset the random number generator state to match the start of the previous iteration's second pass [S1].
    *   **CV-PS (Path Space):** Reuse the sampled light paths. Use BSDF adapters to evaluate the BSDF with parameters $\theta_{n-1}$ on the paths generated for $\theta_n$. For participating media, sample free-flight distances using a majorant that bounds both $\theta_n$ and $\theta_{n-1}$ [S1].
    *   *Note:* The source indicates CV-PS is preferred for lower overhead (1.6x vs 2x) and better correlation [S1].

4.  **Update Statistics:**
    Update the running statistics using the newly obtained $\langle F_{\theta} \rangle_{n}$ and the re-evaluated $\langle F_{\theta} \rangle_{n}^{prev}$. The statistics must be updated *after* computing the control variate to ensure statistical independence of the weight $\alpha_n$ from the current primal estimate, mitigating bias [S1].
    
    Using exponentially weighted Welford’s algorithm with decay factor $\lambda$ (typically close to 1, e.g., 0.99, to emphasize recent history):
    *   Update mean $M_n$ of $\langle F_{\theta} \rangle_{n}$.
    *   Update variance $Var[\langle F_{\theta} \rangle_{n}]$ using $S_n$.
    *   Update covariance $Cov[\langle F_{\theta} \rangle_{n}, \langle F_{\theta} \rangle_{n}^{prev}]$ using $C_n$.
    *   *Note:* The variance of the previous control variate $Var[\langle F_{\theta} \rangle_{CV}^{n-1}]$ is tracked via a recurrence relation (Equation 10 in source) rather than direct estimation, as it is correlated with previous steps [S1].

5.  **Compute Control Weight:**
    Calculate the optimal control weight $\alpha_n$ using the estimated statistics:
    $$ \alpha_n = \frac{Cov[\langle F_{\theta} \rangle_{n}, \langle F_{\theta} \rangle_{n}^{prev}]}{Var[\langle F_{\theta} \rangle_{n}^{prev}] + Var[\langle F_{\theta} \rangle_{CV}^{n-1}]} $$
    Clamp $\alpha_n$ to the range $[0, 1]$ to prevent negative correlation exploitation and aid convergence at the start of optimization [S1].

6.  **Compute Control Variate Estimate:**
    Combine the current primal estimate with the previous control variate:
    $$ \langle F_{\theta} \rangle_{CV}^{n} = \langle F_{\theta} \rangle_{n} + \alpha_n \left( \langle F_{\theta} \rangle_{CV}^{n-1} - \langle F_{\theta} \rangle_{n}^{prev} \right) $$
    Here, $\langle F_{\theta} \rangle_{n}^{prev}$ is the re-evaluation of the previous step's parameters using current samples, acting as the correlated sample for the difference term [S1].

7.  **Gradient Estimation:**
    Compute the gradient of the loss $L$ with respect to parameters $\theta$. To handle the bias from multiplying the gradient of the forward function by the derivative of the loss, use Path Replay Back-propagation (PRB) for the forward gradient and the control variate for the loss derivative [S1]:
    $$ \langle \partial_\theta L(F_\theta, I_{ref}) \rangle = \langle \partial_\theta F_\theta \rangle_{PRB} \cdot \partial L(\langle F_{\theta} \rangle_{CV}^{n}, I_{ref}) $$
    $\langle \partial_\theta F_\theta \rangle_{PRB}$ is the gradient of the forward rendering function estimated using PRB, ensuring independence from the control variate term [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def recursive_control_variates_step(scene_params_prev, scene_params_curr, 
                                    control_variate_prev, stats_prev, 
                                    reference_image, loss_fn, 
                                    sample_generator, majorant):
    """
    One step of Recursive Control Variates for Inverse Rendering.
    
    Args:
        scene_params_prev: Parameters theta_{n-1}
        scene_params_curr: Parameters theta_n
        control_variate_prev: <F_theta>_CV^{n-1}
        stats_prev: Dict containing M, S, C (exponentially weighted stats)
        reference_image: I_ref
        loss_fn: Loss function L
        sample_generator: RNG state for CV-PSS or path history for CV-PS
        majorant: Majorant density for media sampling (bounds both states)
        
    Returns:
        gradient: Estimated gradient
        control_variate_curr: <F_theta>_CV^{n}
        stats_curr: Updated statistics
    """
    
    # 1. Primal Rendering with current parameters
    # Uses sample set S_n
    primal_curr = render(scene_params_curr, sample_generator) 
    
    # 2. Re-evaluation of previous parameters with current samples
    # For CV-PSS: Reset RNG state to match start of prev iteration's second pass
    # For CV-PS: Reuse paths, evaluate BSDF/Media with scene_params_prev
    # Media sampling uses majorant to bound both states
    primal_prev_on_curr_samples = render(scene_params_prev, sample_generator, 
                                         reuse_paths=True, majorant=majorant)
    
    # 3. Compute Control Weight Alpha
    # Extract stats from stats_prev
    cov = stats_prev['cov'] # Cov[<F>_n, <F>_{n-1}^{prev}]
    var_prev_primal = stats_prev['var_prev_primal'] # Var[<F>_{n-1}^{prev}]
    var_prev_cv = stats_prev['var_prev_cv'] # Var[<F>_CV^{n-1}]
    
    numerator = cov
    denominator = var_prev_primal + var_prev_cv
    
    # Avoid division by zero
    if denominator < 1e-8:
        alpha = 0.0
    else:
        alpha = numerator / denominator
        
    # Clamp alpha to [0, 1]
    alpha = max(0.0, min(1.0, alpha))
    
    # 4. Compute Control Variate Estimate
    # <F>_CV^n = <F>_n + alpha * (<F>_CV^{n-1} - <F>_{n-1}^{prev})
    control_variate_curr = primal_curr + alpha * (control_variate_prev - primal_prev_on_curr_samples)
    
    # 5. Update Statistics (Must happen AFTER computing control_variate_curr)
    # Update mean, variance, covariance using exponentially weighted Welford's
    # Note: The variance of the CV is updated via recurrence, not direct Welford
    # Var[<F>_CV^n] = Var[<F>_n] + alpha^2 * Var[<F>_{n-1}^{prev}] 
    #                - 2*alpha*Cov[<F>_n, <F>_{n-1}^{prev}] + alpha^2 * Var[<F>_CV^{n-1}]
    
    var_curr_primal = estimate_variance(primal_curr, stats_prev['var_prev_primal']) # Simplified: use Welford update
    cov_curr = estimate_covariance(primal_curr, primal_prev_on_curr_samples, stats_prev['cov'])
    
    # Recurrence for CV variance
    var_curr_cv = var_curr_primal + (alpha**2 * var_prev_primal) - (2 * alpha * cov_curr) + (alpha**2 * var_prev_cv)
    
    stats_curr = {
        'cov': cov_curr,
        'var_prev_primal': var_curr_primal, # This becomes the "prev" for next step
        'var_prev_cv': var_curr_cv
    }
    # Note: Mean update is implicit in the Welford logic for the specific estimator, 
    # but the source emphasizes the covariance/variance tracking for alpha.
    
    # 6. Gradient Estimation
    # Use PRB for the forward gradient to ensure independence
    grad_forward = compute_gradient_prb(scene_params_curr, sample_generator)
    
    # Use Control Variate for the loss derivative
    # Loss derivative is evaluated on the denoised/low-variance estimate
    loss_derivative = loss_fn(control_variate_curr, reference_image)
    
    gradient = grad_forward * loss_derivative
    
    return gradient, control_variate_curr, stats_curr
```

**Parameters:**
*   **Sample count per pixel:** Low sample counts (e.g., 16) are sufficient in the primal rendering phase because the control variate reduces noise. Prior work often required tens to hundreds of samples [S1].
*   **Control weight $\alpha_n$:** Clamped to $[0, 1]$ [S1].
*   **Exponential weighting factor:** Not explicitly given, but "exponentially weighted variant of Welford’s algorithm" is used [S1].

**Edge Cases and Pitfalls:**
*   **Bias from Correlated Weight:** The weight $\alpha_n$ is correlated with previous estimates. To mitigate bias, update running statistics *after* evaluating $\langle F_{\theta} \rangle_{CV}$ [S1].
*   **Bias from Non-linear Loss:** For non-L2 losses, $E[f(X)] \neq f(E[X])$ causes bias. The control variate reduces this bias by providing a low-variance estimate, but L2 loss is the only one yielding unbiased derivatives in this context without additional debiasing [S1].
*   **Bias from Multiplication:** The product in the gradient estimator is unbiased only if factors are independent. PRB is used for the forward gradient to ensure this [S1].
*   **Geometric Optimization:** Moving/deforming surfaces are not supported. CV-PSS loses correlation; CV-PS would require "shift mappings" [S1].
*   **Denoising:** Applying denoisers to primal images adds bias. Denoising the control variate estimator is preferred [S1].

## Subtopics

### [Heuristic Control Weights for Recursive Control Variates](11578-heuristic-control-weights-for-recursive-control-variates.md)
A variant that uses heuristic methods to determine control weights instead of the statistically derived recurrence, potentially simplifying implementation at the cost of optimality.

## Operating characteristics
*   **Time Complexity:** The method adds a re-evaluation step per iteration. CV-PSS doubles the primal rendering cost (approx. 1/3 of total PRB step cost). CV-PS reduces this overhead to 1.6x and allows fusing computation into a single GPU kernel [S1].
*   **Memory Complexity:** Constant memory overhead, storing only the previous control variate estimate and running statistics (mean, variance, covariance) per channel/pixel [S1]. This is significantly lower than methods maintaining a full covariance matrix of all previous renderings [S1].
*   **Measured Performance:** Primal rendering efficiency improves by a factor of up to 10 compared to prior work, leading to significant speedup at equal quality or higher quality at equal time [S1].
*   **Scaling:** When optimization reaches steady state, the recursive control variate becomes equivalent to rendering with a correspondingly larger sample count, implying unbounded noise reduction potential without proportional linear cost increase [S1].
*   **Strengths:** Reduces variance and bias; allows low sample counts; works with non-L2 losses (by reducing bias); compatible with PRB and DRT [S1].
*   **Limitations:** Does not support geometric optimization (moving surfaces); CV-PSS has higher overhead and loses correlation for parameters affecting path geometry (roughness, IOR); bias remains due to correlation between weight and previous CV [S1].
*   **When to use:** Inverse rendering with textured surfaces and homogeneous/heterogeneous volumes where scene parameters change slowly between iterations.
*   **When not to use:** Geometric optimization with deforming surfaces; scenarios where high correlation between steps is not present.

## Background
Recursive Control Variates for Inverse Rendering was introduced by Baptiste Nicolet et al. in 2023 [S1]. It builds on the theory of control variates [Fieller and Hartley 1954] and previous work on variance reduction in rendering [Lafortune and Willems 1995a,b; Rousselle et al. 2016]. The method was implemented in Mitsuba 3, combined with Path Replay Back-propagation (PRB) for surfaces/homogeneous volumes and Differential Ratio Tracking (DRT) for heterogeneous volumes [S1]. It addresses the issue of noisy gradients in differentiable rendering, which often leads to drift in optimization, by exploiting the redundancy in consecutive gradient descent steps [S1].

## Sources
[S1] Recursive Control Variates for Inverse Rendering, https://research.nvidia.com/labs/rtr/publication/nicolet2023recursive/, fetched 2026-11-08
