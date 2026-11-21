# Adaptive View and Pixel Sampling Scheduling

Also called: Adaptive Scheduling of View Budget and Pixel Sampling, Sampled view number scheduling, Tile-wise pixel sampling rate scheduling, query sampling schedule

## Overview
A scheduling mechanism that jointly adjusts the tile-wise pixel sampling rate ($r_{tile}$) and the number of sampled views ($K$) over training iterations to allocate more computation during difficult stages and reduce it as optimization converges [S1, unofficial]. It sets itself apart from static sampling strategies by dynamically scaling the view budget and pixel density based on reconstruction error and training progress, achieving a trade-off between training speed and reconstruction quality [S1, unofficial].

## How it works
**Inputs and outputs:**
*   **Inputs:** Current iteration $t$, initial mean view error $\bar{E}_{init}$, current mean view error $\bar{E}_t$, densification/pruning cutoff iteration $\tau_{until}$, tile sampling scale $\lambda$, minimum/maximum view budgets $K_{min}, K_{max}$, transition speed $\alpha$, and minimum/maximum sampling rates $r_{min}, r_{max}$.
*   **Outputs:** Tile-wise pixel sampling rate $r_{tile}^{(t)}$ and sampled view number $K_t$ for the current iteration.

**Data structures:**
No persistent data structures are required beyond scalar values for error tracking and iteration counters. The scheduler computes values on-the-fly.

**Parameters:**
*   $\lambda$: Tile sampling scale. Typical value not given in the sources.
*   $K_{min}, K_{max}$: Minimum and maximum number of sampled views. The source uses a range of $10 \to 3$ (i.e., $K_{max}=10, K_{min}=3$) [S1, unofficial].
*   $\alpha$: Controls the transition speed of the view budget scheduling. Value not given in the sources.
*   $\tau_{until}$: The last iteration for densification and pruning. Value not given in the sources.
*   $\bar{E}_{init}$: Initial mean view error. Computed at the start of training.
*   $\bar{E}_t$: Current mean view error at iteration $t$.

**Steps:**
1.  **Compute Error Ratio ($r_t$):** Calculate the normalized error ratio by clipping the square root of the ratio of current error to initial error.
    $$ r_t = \text{clip}\left(\sqrt{\frac{\bar{E}_t}{\bar{E}_{init} + \epsilon}}, 0, 1\right) $$
    where $\epsilon$ is a small constant for numerical stability [S1, unofficial].

2.  **Compute Sinusoidal Modulation ($\phi(r_t)$):** Apply a sinusoidal function to amplify sampling variation in the low-error regime.
    $$ \phi(r_t) = \sin\left(\frac{\pi}{2} r_t\right) $$
    This modulation is chosen because reconstruction error drops rapidly in early training and remains in a narrow range later, making sinusoidal modulation more sensitive to subtle differences than linear or exponential variants [S1, unofficial].

3.  **Compute Progress Coefficient ($\rho_t$):** Determine the annealing factor based on training progress relative to the densification phase.
    $$ \rho_t = 1 - \text{clip}\left(\frac{t}{\tau_{until}}, 0, 1\right) $$
    This coefficient gradually reduces sampling intensity after the densification and pruning phase is complete [S1, unofficial].

4.  **Compute Tile-wise Pixel Sampling Rate ($r_{tile}^{(t)}$):**
    $$ r_{tile}^{(t)} = r_{min} + (r_{max} - r_{min}) \left(1 - \rho_t \phi(r_t)\right) $$
    where $r_{min} = \lambda / K_{max}$ and $r_{max} = \lambda / K_{min}$ [S1, unofficial].

5.  **Compute Sampled View Number ($K_t$):**
    $$ K_t = \text{round}\left(K_{min} + (K_{max} - K_{min}) e^{-\alpha(1 - \rho_t \phi(r_t))}\right) $$
    This formula exponentially decays the view budget as the term $(1 - \rho_t \phi(r_t))$ increases, shifting from global multi-view consistency to local refinement [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_scheduling_params(
    t: int,
    tau_until: int,
    E_t: float,
    E_init: float,
    lambda_scale: float,
    K_min: int,
    K_max: int,
    alpha: float,
    epsilon: float = 1e-6
) -> tuple[float, int]:
    """
    Computes the tile-wise pixel sampling rate and sampled view number
    for the current training iteration.
    
    Args:
        t: Current training iteration.
        tau_until: Iteration after which densification/pruning stops.
        E_t: Current mean view error.
        E_init: Initial mean view error.
        lambda_scale: Tile sampling scale.
        K_min: Minimum number of sampled views.
        K_max: Maximum number of sampled views.
        alpha: Transition speed control for view budget.
        epsilon: Small constant for numerical stability.
        
    Returns:
        (r_tile, K_t): Tile-wise pixel sampling rate and sampled view number.
    """
    # Step 1: Compute Error Ratio
    # Clip the square root of the error ratio to [0, 1]
    error_ratio_raw = math.sqrt(E_t / (E_init + epsilon))
    r_t = max(0.0, min(1.0, error_ratio_raw))
    
    # Step 2: Compute Sinusoidal Modulation
    phi_r_t = math.sin(math.pi / 2 * r_t)
    
    # Step 3: Compute Progress Coefficient
    # Clip t / tau_until to [0, 1], then subtract from 1
    progress_ratio = max(0.0, min(1.0, t / tau_until))
    rho_t = 1.0 - progress_ratio
    
    # Step 4: Compute Sampling Rate Bounds
    r_min = lambda_scale / K_max
    r_max = lambda_scale / K_min
    
    # Step 5: Compute Tile-wise Pixel Sampling Rate
    # r_tile = r_min + (r_max - r_min) * (1 - rho_t * phi(r_t))
    modulation_factor = 1.0 - rho_t * phi_r_t
    r_tile = r_min + (r_max - r_min) * modulation_factor
    
    # Step 6: Compute Sampled View Number
    # K_t = round(K_min + (K_max - K_min) * exp(-alpha * modulation_factor))
    view_budget = K_min + (K_max - K_min) * math.exp(-alpha * modulation_factor)
    K_t = round(view_budget)
    
    return r_tile, K_t
```

**Edge cases and pitfalls:**
*   **Very small view budget:** Setting $K=1$ degrades performance due to insufficient multi-view constraints [S1, unofficial].
*   **Large fixed budget:** Using a large fixed budget (e.g., $K=10$) increases training time without quality gains [S1, unofficial].
*   **Division by zero:** Ensure $\bar{E}_{init} + \epsilon > 0$ to prevent division by zero in the error ratio calculation [G].
*   **Negative sampling rate:** The clipping in $r_t$ and $\rho_t$ ensures the modulation factor stays within bounds, but implementation should verify $r_{tile} \ge 0$ [G].

## Operating characteristics
*   **Time Complexity:** The scheduling computation itself is $O(1)$ per iteration. However, the method reduces the total training time by reducing the number of sampled views and pixels in later stages. The source reports a reduction in training time from 81s (linear schedule) to 77s (sinusoidal schedule) while improving PSNR [S1, unofficial].
*   **Memory Complexity:** No additional memory overhead compared to static sampling, as it only computes scalar values.
*   **Measured Performance:** On the tested scenes, the sinusoidal schedule achieved a PSNR of 27.57, SSIM of 0.794, and LPIPS of 0.256, outperforming linear (PSNR 27.35) and exponential (PSNR 27.27) schedules [S1, unofficial].
*   **Scaling:** The method scales by reducing computation as the model converges, making it more efficient for long training runs.
*   **Strengths:** Improves sensitivity to subtle reconstruction differences in the low-error regime; achieves best trade-off between speed and quality by gradually shifting from global multi-view consistency to local refinement [S1, unofficial].
*   **Limitations:** Requires tracking initial and current mean view errors; sensitive to the choice of $\alpha$ and $\tau_{until}$ [S1, unofficial].
*   **When to use:** When training 3D Gaussian Splatting models and seeking to accelerate convergence without sacrificing quality, particularly when reconstruction error dynamics are non-linear [S1, unofficial].
*   **When not to use:** When computational resources are extremely limited in the early stages and multi-view consistency is critical (avoid $K=1$) [S1, unofficial].

## Background
The method was introduced in the paper "TurboGS: Accelerating 3D Gaussian Splatting via Error-Guided Sparse Pixel Sampling and Optimization" [S1, unofficial]. It addresses the inefficiency of fixed sampling rates in 3D Gaussian Splatting training, where early iterations require dense sampling for global consistency and later iterations benefit from sparse sampling for local refinement. The sinusoidal modulation was found to outperform linear and exponential variants based on ablation studies [S1, unofficial].

## Sources
[S1] TurboGS: Accelerating 3D Gaussian Splatting via Error-Guided Sparse Pixel Sampling and Optimization, https://doi.org/10.48550/arXiv.2606.15924, fetched 2026-10-29 (unofficial)
