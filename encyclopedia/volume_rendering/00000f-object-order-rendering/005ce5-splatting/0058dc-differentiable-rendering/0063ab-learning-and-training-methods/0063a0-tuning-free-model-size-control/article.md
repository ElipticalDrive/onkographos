# Tuning-Free Model-Size Control

Also called: One Setting, Any Scene

## Overview
Tuning-Free Model-Size Control is a method for enforcing a strict primitive count budget in 3D Gaussian Splatting during training, ensuring the final model contains exactly the user-specified number of primitives regardless of scene complexity [S1, unofficial]. Unlike MaskGaussian, which allows the model size to vary significantly (up to 7.8x) across different scenes, this method delivers the exact requested size in every instance by using a controller that approaches the budget from above and applies a terminal projection [S1, unofficial]. It produces models that are 2.2x smaller than MaskGaussian and 1.40x faster to render, with paired quality differences bounded to [-0.076, +0.085] dB [S1, unofficial].

## How it works
The method replaces the standard adaptive density control (which grows primitives based on gradient thresholds without a stopping criterion) with a closed-loop controller that treats the model size as a constrained optimization variable [S1, unofficial]. It operates on the population of 3D Gaussians, managing their count via multiplicative dual ascent.

**Inputs and Outputs:**
*   **Input:** The target budget $B$ (the exact number of primitives desired).
*   **Output:** A trained 3D Gaussian Splatting model containing exactly $B$ primitives.

**Data Structures:**
The method operates on the standard 3D Gaussian Splatting representation: a set of $N$ primitives, each defined by position $\mathbf{t}$, covariance $\Sigma$ (or rotation $\mathbf{q}$ and scale $\mathbf{s}$), opacity $\alpha$, and spherical harmonics coefficients $\mathbf{c}$ [S1, unofficial]. The controller maintains a scalar dual variable $\lambda$ (the Lagrange multiplier for the size constraint) and internal state variables for the controller logic.

**Parameters:**
The controller uses a fixed set of constants that are applied unchanged across all experiments (13 scenes, 3 datasets, 2 seeds) [S1, unofficial]:
*   Gain
*   Step bound
*   Dead band
*   Release ratio
*   Initial multiplier
*   Setpoint margin
*   Projection iteration count

The user supplies only the budget $B$.

**Algorithm Steps:**
1.  **Initialization:** Initialize the dual variable $\lambda$ using the `initial multiplier` and set the controller state. Set the current primitive count $N_{curr} = N_{initial}$.
2.  **Setpoint Definition:** Define a setpoint $S$ for the controller. The setpoint is held above the target budget $B$ by a `setpoint margin` ($S = B + \text{margin}$). This ensures the population approaches the budget from the "recoverable side" (overshoot), avoiding the failure mode where masking prevents recovery from undershoot [S1, unofficial].
3.  **Training Loop:** For each training step:
    a. Render the image and compute the loss (photometric + perceptual) [G].
    b. Compute the error $e = N_{curr} - S$.
    c. Update the dual variable $\lambda$ using **multiplicative dual ascent on a log-domain error**. The update incorporates an **asymmetric release mechanism** and is bounded by the `step bound` and `dead band` [S1, unofficial].
    d. Apply the dual variable $\lambda$ to influence the density control:
        *   If $\lambda$ indicates the population is too large, primitives are marked for deletion.
        *   If $\lambda$ indicates the population is too small, primitives are cloned or split (standard 3D-GS growth) [S1, unofficial].
    e. **Deletion Floor:** Enforce a deletion floor to prevent the removal of primitives below a certain threshold, ensuring stability [S1, unofficial].
    f. Update $N_{curr}$ based on the growth/deletion operations.
4.  **Terminal Projection:** After the main training loop converges, perform a **single terminal projection** onto the budget $B$. This step adjusts the primitive set to contain exactly $B$ primitives, correcting any residual drift [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tuning_free_model_size_control(gaussians, target_budget, params):
    """
    gaussians: List of Gaussian primitives
    target_budget: Integer, the exact number of primitives desired
    params: Dictionary containing gain, step_bound, dead_band, release_ratio,
            initial_multiplier, setpoint_margin, projection_iterations
    """
    # 1. Initialization
    # Dual variable lambda controls the pressure to reduce/increase size
    lambda_val = params['initial_multiplier']
    
    # Setpoint is held above the budget to approach from the recoverable side
    setpoint = target_budget + params['setpoint_margin']
    
    # Current count
    current_count = len(gaussians)
    
    # 2. Training Loop
    for step in training_steps:
        # Render and compute loss (standard differentiable rendering)
        # ... (standard 3D-GS training step) ...
        
        # Compute error relative to setpoint
        error = current_count - setpoint
        
        # Multiplicative dual ascent on log-domain error with asymmetric release
        # The update is asymmetric: different behavior for positive vs negative error
        # bounded by step_bound and dead_band
        if abs(error) > params['dead_band']:
            # Asymmetric release logic
            if error > 0: # Overshoot: need to delete
                # Increase lambda to penalize size
                lambda_val *= (1 + params['gain'] * error)
                lambda_val = min(lambda_val, params['step_bound'])
            else: # Undershoot: need to grow
                # Decrease lambda or allow growth
                lambda_val /= (1 + params['release_ratio'] * abs(error))
                lambda_val = max(lambda_val, 1.0 / params['step_bound'])
        
        # Apply density control influenced by lambda
        # If lambda is high, prune primitives; if low, clone/split
        # Deletion floor prevents removal below a threshold
        gaussians, current_count = apply_density_control(
            gaussians, lambda_val, params['deletion_floor']
        )
        
    # 3. Terminal Projection
    # Ensure exactly target_budget primitives
    if current_count != target_budget:
        gaussians = project_to_budget(gaussians, target_budget)
        
    return gaussians

def apply_density_control(gaussians, lambda_val, deletion_floor):
    """
    Applies growth or deletion based on lambda.
    If lambda > 1, pressure to delete. If lambda < 1, pressure to grow.
    """
    # Standard 3D-GS gradient-based growth
    # ... (clone/split logic based on gradients) ...
    
    # Deletion logic influenced by lambda
    # Primitives with low contribution are candidates for deletion
    candidates = get_deletion_candidates(gaussians)
    
    # Enforce deletion floor
    min_count = deletion_floor
    if len(gaussians) - len(candidates) < min_count:
        candidates = candidates[:len(gaussians) - min_count]
        
    # Remove candidates
    for g in candidates:
        gaussians.remove(g)
        
    return gaussians, len(gaussians)

def project_to_budget(gaussians, target):
    """
    Single terminal projection to ensure exact count.
    """
    current = len(gaussians)
    if current > target:
        # Remove excess primitives, e.g., those with lowest opacity/contribution
        gaussians = remove_least_contribution(gaussians, current - target)
    elif current < target:
        # Clone or add primitives to reach target
        gaussians = add_primitives(gaussians, target - current)
    return gaussians
```

**Edge Cases and Pitfalls:**
*   **Overshoot Recovery:** Naive dual ascent fails if the population crosses the budget early because masking is a "one-way door" (primitives cannot be easily recovered once masked/deleted). The setpoint held above the budget ensures the population settles from the recoverable side [S1, unofficial].
*   **Undershoot:** Without the terminal projection, undershoots of up to 36% were observed in naive controllers [S1, unofficial]. The terminal projection removes this failure entirely.
*   **Negative Results:** Weighting sparsity penalty by screen footprint and annealing Gumbel temperature, which help when size is free, collapse once size is held fixed [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The controller adds minimal overhead to the standard 3D-GS training loop, primarily involving the calculation of the dual variable update and the final projection. The projection is a single pass at the end.
*   **Memory Complexity:** The memory usage is strictly bounded by the target budget $B$, unlike standard 3D-GS which can grow to millions of primitives. Models are 2.2x smaller than MaskGaussian and 5.0x smaller than standard 3D-GS [S1, unofficial].
*   **Performance:** Models are 1.40x faster to render than MaskGaussian [S1, unofficial].
*   **Strengths:** Delivers exactly the requested number of primitives in every scene and seed. No per-scene tuning is required.
*   **Limitations:** The quality difference is bounded but not zero ([-0.076, +0.085] dB vs MaskGaussian). The method relies on a terminal projection, which may require a final pass over the data.
*   **When to use:** When deployment constraints require a strict, predictable model size. When MaskGaussian's variability is unacceptable.

## Background
The method was introduced in the paper "One Setting, Any Scene: Tuning-Free Model-Size Control for 3D Gaussian Splatting" [S1, unofficial]. It addresses the issue that 3D Gaussian Splatting's adaptive density control is driven by a photometric criterion that does not know when to stop, leading to models with several million primitives [S1, unofficial]. Previous methods like MaskGaussian allowed size to vary significantly. This work proposes a controller that is correct under the asymmetry of soft masking, using multiplicative dual ascent on a log-domain error [S1, unofficial].

## Sources
[S1] One Setting, Any Scene: Tuning-Free Model-Size Control for 3D Gaussian Splatting, https://doi.org/10.21203/rs.3.rs-10987108/v1, fetched 2026-10-26 (unofficial)
