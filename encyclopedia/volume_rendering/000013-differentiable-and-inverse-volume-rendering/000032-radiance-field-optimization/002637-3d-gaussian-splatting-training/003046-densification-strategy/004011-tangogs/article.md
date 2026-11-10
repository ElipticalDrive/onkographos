# TangoGS

Also called: Gauss What You Need, Capture-Conditioned Learning Allowance

## Overview
TangoGS is a density controller for 3D Gaussian Splatting that automatically scales the number of Gaussians across different capture sizes without per-capture retuning [S1, unofficial]. It combines a capture-derived "learning allowance" (model sizing based on track-adjusted pixels) with training-based adaptation (quality feedback) to determine the final Gaussian population [S1, unofficial]. Unlike sibling methods that require manual hyperparameter tuning for each scene, TangoGS uses a shared configuration for all captures, matching the mean PSNR of the best-performing baselines with significantly fewer Gaussians [S1, unofficial].

## How it works
TangoGS operates within the differentiable training loop of 3D Gaussian Splatting, building on [Densification Strategy](6234-densification-strategy.md). It manages Gaussian population through three phases: allowance calculation, budgeted densification with soft pruning, and final selection.

### Inputs and Outputs
*   **Inputs:**
    *   $N_{\text{img}}$: Total capture image count (including held-out validation images) [S1, unofficial].
    *   $P$: Mean per-image PSNR over the just-completed training epoch [S1, unofficial].
    *   $N_{\text{pre}}$: Population of Gaussians entering the final selection event [S1, unofficial].
    *   Current Gaussian attributes (position, covariance, opacity, SH coefficients, Adam buffers) [S1, unofficial].
*   **Outputs:**
    *   $T_{\text{adapt}}$: Target Gaussian count for the current growth window [S1, unofficial].
    *   Final Gaussian set $G_{\text{final}}$ with size $N_G$ [S1, unofficial].

### Data Structures
*   **Gaussian Buffers:** Parameters and Adam optimizer buffers are stored in GPU arrays. To align with batch removal operations, these are clustered into Morton-ordered groups of 128 elements [S1, unofficial].
*   **Sensitivity Score:** A scalar value accumulated per Gaussian over an epoch, representing the opacity-weighted effect on training views [S1, unofficial].
*   **Culling Map:** A data structure that scatters visible cluster scores back to original Gaussian indices to enable per-Gaussian pruning decisions [S1, unofficial].

### Formulas and Constants
The method relies on specific constants derived from 13 standard benchmark scenes [S1, unofficial]:
*   $g \approx 0.063362106$: Effective Gaussian count scaling factor (Gaussians per track-adjusted pixel) [S1, unofficial].
*   $T_{\text{ref}} = 2,462,476$: Reference count, the rounded geometric mean of the 13 standard linear allowances [S1, unofficial].
*   $s = 1.7$: Sensitivity scaling factor for final selection [S1, unofficial].
*   $c = 0.0075$: Quality feedback coefficient [S1, unofficial].
*   $P^* = 20 \text{ dB}$: Threshold PSNR below which no removal occurs [S1, unofficial].

**1. Learning Allowance Calculation**
The target capacity is derived from the capture size. First, compute the linear allowance $T_{\text{lin}}$ using nested rounding to implement the scaling factor $g$ [S1, unofficial]:
$$ T_{\text{lin}} = \text{round} \left[ 0.90 \cdot \text{round}(0.07040234 \cdot N_{\text{img}}) \right] $$
Then, compute the capped allowance $T_{\text{cap}}$ using a square-root mapping to reduce spread between scenes [S1, unofficial]:
$$ T_{\text{cap}} = \text{round} \left[ T_{\text{ref}} \cdot \left( \frac{T_{\text{lin}}}{T_{\text{ref}}} \right)^{0.5} \right] $$

**2. Quality Feedback and Keep Ratio**
At each growth or pruning event, the keep ratio $\rho$ is computed from the preceding epoch's mean PSNR $P$ [S1, unofficial]:
$$ \rho = 0.7382 - 0.0075 \cdot P $$
This formula is a rounded approximation of $\rho = \frac{1}{s} - c(P - P^*)$. If $P \le P^*$, $\rho$ effectively caps such that no removal occurs [S1, unofficial].

**3. Densification Target**
The target count for the current growth window is [S1, unofficial]:
$$ T_{\text{adapt}} = \rho \cdot T_{\text{cap}} $$

**4. Final Selection**
After growth ends, the final retained count $N_G$ is determined by applying a keep fraction $f$ repeatedly [S1, unofficial]:
$$ f = \min(1, s \cdot \rho) $$
$$ N_G = N_{\text{pre}} \cdot f^h $$
where $h$ is the number of remaining events in a reference pruning schedule spaced 8% of total training iterations apart [S1, unofficial].

### Parameters
*   **Training Seeds:** 4114 and 4115 (fix training-view order and stochastic draws) [S1, unofficial].
*   **Iteration Counts:** 30,000 for standard scenes; 60,000 for large captures [S1, unofficial].
*   **Epoch Spacing Unit:** 150 epochs. Growth events are spaced 5 epochs apart for runs $\ge 150$ epochs; for shorter runs, spacing scales proportionally to a minimum of 1 epoch [S1, unofficial].
*   **SH-rest Learning-Rate Divisor:** 20 [S1, unofficial].
*   **Split/Clone Threshold:** 0.001 (maximum gradient magnitude) [S1, unofficial].
*   **Opacity Decay:** Factor 0.5 per step, floor 1/128 [S1, unofficial].

### Numbered Steps

1.  **Initialization:** Calculate $T_{\text{lin}}$ and $T_{\text{cap}}$ from $N_{\text{img}}$ using the formulas above. Initialize Gaussian population. Set densification start at 2% of total iterations and end at 80% [S1, unofficial].
2.  **Training Loop:** For each iteration:
    *   Render view and compute photometric loss.
    *   Backpropagate loss.
    *   Accumulate sensitivity scores for all active Gaussians using LiteGS's kernel (substituting a unit upstream image multiplier) [S1, unofficial].
3.  **Epoch End (Calibration):**
    *   Compute mean per-image PSNR $P$ over the training views of the just-completed epoch [S1, unofficial].
    *   Recalibrate quality feedback using $P$ [S1, unofficial].
4.  **Growth Event (Triggered periodically):**
    *   Compute $T_{\text{adapt}} = \rho(P) \cdot T_{\text{cap}}$ [S1, unofficial].
    *   Perform budgeted densification: Add Gaussians to ramp linearly from the current count to $T_{\text{adapt}}$ over the growth window [S1, unofficial].
    *   Newly created Gaussians are exempt from immediate pruning [S1, unofficial].
5.  **Soft Prune (Triggered periodically, starts at ~9% training):**
    *   Rank Gaussians present *before* the current growth event using their accumulated sensitivity scores [S1, unofficial].
    *   Compute keep fraction $f = \min(1, s \cdot \rho)$ [S1, unofficial].
    *   Keep the top fraction $\rho$ (or $f$ for final selection) of Gaussians.
    *   Remove the rest. Removal counts are rounded down to the nearest multiple of 128 [S1, unofficial].
    *   Guard: If removal would exceed 90% of the population, abort the event [S1, unofficial].
6.  **Final Selection (At 80% training):**
    *   Perform a single hard prune using $N_G = N_{\text{pre}} \cdot f^h$ [S1, unofficial].
    *   Optimize the retained population for the remainder of training [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calculate_allowance(n_img, t_ref=2462476):
    """Calculate TangoGS learning allowance."""
    # Nested rounding for g ~ 0.063362106
    t_lin = math.round(0.90 * math.round(0.07040234 * n_img))
    # Square root mapping
    t_cap = math.round(t_ref * (t_lin / t_ref) ** 0.5)
    return t_lin, t_cap

def compute_keep_ratio(p, s=1.7, c=0.0075, p_star=20.0):
    """Compute keep ratio from PSNR."""
    # Approximation: rho = 0.7382 - 0.0075 * P
    rho = 0.7382 - c * p
    return rho

def get_final_keep_fraction(rho, s=1.7):
    """Compute f for final selection."""
    return min(1.0, s * rho)

def tango_gs_training_loop(gaussians, n_img, total_iterations, n_epochs):
    """
    Simplified TangoGS training loop.
    """
    t_lin, t_cap = calculate_allowance(n_img)
    
    # Schedule parameters
    densify_start = 0.02 * total_iterations
    densify_end = 0.80 * total_iterations
    prune_start = 0.09 * total_iterations
    
    # Epoch spacing logic
    if n_epochs >= 150:
        growth_spacing_epochs = 5
    else:
        # Scale proportionally, min 1
        growth_spacing_epochs = max(1, 5 * (n_epochs / 150))
        
    current_epoch = 0
    sensitivity_scores = {g.id: 0.0 for g in gaussians}
    
    for iteration in range(total_iterations):
        # 1. Standard 3DGS step
        view = get_training_view(iteration)
        loss = render_and_compute_loss(gaussians, view)
        loss.backward()
        
        # 2. Accumulate sensitivity scores (LiteGS kernel)
        # In practice, this is done via a custom backward pass
        # Here we assume scores are accumulated in the gradient step
        accumulate_sensitivity_scores(gaussians)
        
        # 3. Check for epoch end
        if iteration % (total_iterations / n_epochs) < 1:
            current_epoch += 1
            
            # Compute mean PSNR P
            p = compute_mean_psnr(gaussians)
            
            # 4. Check for Growth Event
            if densify_start <= iteration <= densify_end:
                if current_epoch % growth_spacing_epochs == 0:
                    rho = compute_keep_ratio(p)
                    t_adapt = rho * t_cap
                    
                    # Budgeted densification: ramp to t_adapt
                    current_count = len(gaussians)
                    if current_count < t_adapt:
                        densify_gaussians(gaussians, t_adapt - current_count)
                        
                    # 5. Soft Prune (if after prune_start)
                    if iteration > prune_start:
                        # Only prune Gaussians present BEFORE growth
                        pre_growth_ids = set(g.id for g in gaussians)
                        
                        # Rank by sensitivity
                        ranked_gaussians = sort_by_sensitivity(gaussians, pre_growth_ids)
                        
                        # Calculate removal count
                        keep_fraction = rho # For soft prune, use rho directly as per text "keeps top fraction rho"
                        # Note: Text says "keeps the top fraction rho" for soft prune
                        # and "f = min(1, s*rho)" for final selection.
                        
                        n_keep = math.floor(len(ranked_gaussians) * keep_fraction)
                        n_remove = len(ranked_gaussians) - n_keep
                        
                        # Guard against catastrophic collapse
                        if n_remove > 0.9 * len(gaussians):
                            continue # Abort prune
                            
                        # Remove bottom n_remove
                        # Round removal count to multiple of 128
                        n_remove = (n_remove // 128) * 128
                        
                        remove_gaussians(gaussians, ranked_gaussians[n_keep:n_keep+n_remove])
                        
            # 6. Final Selection
            if iteration == densify_end:
                rho = compute_keep_ratio(p)
                f = get_final_keep_fraction(rho)
                
                # h is number of remaining events in reference schedule
                # Reference schedule: pruning events spaced 8% of total iterations
                # h = remaining_events
                # For simplicity, assume h is pre-calculated or derived from schedule
                h = calculate_h(total_iterations) 
                
                n_pre = len(gaussians)
                n_final = math.floor(n_pre * (f ** h))
                
                # Hard prune
                ranked_gaussians = sort_by_sensitivity(gaussians)
                remove_gaussians(gaussians, ranked_gaussians[n_final:])
                
                # Optimize remaining
                optimize_gaussians(gaussians)
```

### Edge Cases and Pitfalls
*   **Catastrophic Collapse:** If a pruning event would remove more than 90% of the Gaussian population, the event is aborted [S1, unofficial].
*   **Short Runs:** For training runs with fewer than 150 epochs, growth event spacing scales down to a minimum of 1 epoch to ensure sufficient growth events [S1, unofficial].
*   **New Gaussians:** Newly created Gaussians during densification are exempt from pruning until they have trained for a period [S1, unofficial].
*   **Batch Alignment:** Removal counts are rounded down to the nearest multiple of 128 to maintain Morton-order group alignment [S1, unofficial].

## Subtopics

### [Capture-Conditioned Learning Allowance](10214-capture-conditioned-learning-allowance.md)
Determines the target Gaussian population size based on capture characteristics (track-adjusted pixels) to replace fixed defaults.

## Operating characteristics
*   **Time Complexity:** Training time is comparable to standard 3DGS, with additional overhead for sensitivity score accumulation and periodic densification/pruning. The sensitivity scoring uses LiteGS's kernel, avoiding additional renders [S1, unofficial].
*   **Memory Complexity:** Memory usage scales with the number of Gaussians. TangoGS typically uses 48% fewer Gaussians than baselines on standard scenes, reducing memory footprint [S1, unofficial]. Buffer capacity must be pre-allocated to handle growth events.
*   **Performance:** On 13 standard benchmark scenes, TangoGS matches the mean PSNR of the best-performing baseline (LeGS) with 48% fewer Gaussians [S1, unofficial]. On 8 large captures, it automatically scales to larger models without retuning [S1, unofficial].
*   **Strengths:** No per-capture tuning required; shared configuration across all 21 tested captures; robust scaling from small to large scenes; reduced model size.
*   **Limitations:** Requires accurate track-adjusted pixel count ($N_{\text{img}}$) from the capture metadata; sensitivity scoring relies on LiteGS implementation details; opacity decay mechanism differs from standard resets.

## Background
TangoGS was introduced in the paper "Gauss What You Need: Compact Gaussian Splatting Across Scene Scales" [S1, unofficial]. It addresses the limitation of standard 3DGS, which requires manual tuning of densification parameters for different scene scales. By decoupling model sizing (from capture geometry) from model quality (from training feedback), it provides a unified approach for diverse captures. The method builds on LiteGS for rendering and optimization [S1, unofficial].

## Sources
[S1] Gauss What You Need: Compact Gaussian Splatting Across Scene Scales, https://arxiv.org/abs/2609.31248, fetched 2026-10-29 (unofficial)
