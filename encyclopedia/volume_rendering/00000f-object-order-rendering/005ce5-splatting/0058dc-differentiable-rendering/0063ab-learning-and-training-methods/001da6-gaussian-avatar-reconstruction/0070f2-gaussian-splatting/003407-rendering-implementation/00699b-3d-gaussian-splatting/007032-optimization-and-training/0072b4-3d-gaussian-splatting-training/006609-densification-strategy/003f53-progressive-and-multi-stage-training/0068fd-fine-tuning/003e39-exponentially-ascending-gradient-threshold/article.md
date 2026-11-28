# Exponentially Ascending Gradient Threshold

Also called: Exponential grad thresh

## Overview
A densification control mechanism for 3D Gaussian Splatting that uses an exponentially ascending gradient threshold $T_i$ to determine when to split or clone Gaussians [S1]. It solves the trade-off between slow convergence in early training (due to few Gaussians) and overfitting in late training (due to excessive densification) by allowing aggressive densification early and restricting it later [S1]. It sets itself apart from static threshold methods by dynamically adjusting the sensitivity of densification based on the training progress [S1].

## How it works
**Inputs and Outputs:**
- Input: Current iteration index $i$, total iterations $i_{\max}$, initial threshold $T_s$, final threshold $T_f$, and the modified gradient magnitude $\tau'_k$ for each Gaussian $g_k$.
- Output: A boolean decision for each Gaussian indicating whether to split, clone, or do nothing.

**Data Structures:**
- $T_i$: Scalar threshold value for iteration $i$.
- $\tau'_k$: Scalar modified gradient magnitude for Gaussian $k$.
- $T_s$: Initial threshold (default 0.0001).
- $T_f$: Final threshold (default 0.0004).
- $i$: Current iteration index (integer).
- $i_{\max}$: Total number of training iterations (integer).

**Steps:**
1. Calculate the normalized iteration progress: $p = \frac{i}{i_{\max}}$.
2. Compute the logarithmic interpolation between the natural logs of the initial and final thresholds:
   $$ \ln(T_i) = \ln(T_s) \cdot (1 - p) + \ln(T_f) \cdot p $$
3. Exponentiate the result to obtain the current threshold:
   $$ T_i = \exp \left( \ln(T_s) \cdot \left(1 - \frac{i}{i_{\max}}\right) + \ln(T_f) \cdot \frac{i}{i_{\max}} \right) $$
4. For each Gaussian $g_k$ in the scene:
   a. Retrieve its modified gradient magnitude $\tau'_k$.
   b. Compare $\tau'_k$ against $T_i$.
   c. If $\tau'_k \ge T_i$, consider densification (split or clone based on size constraints).
   d. If $\tau'_k < T_i$, skip densification for this Gaussian.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calculate_exponential_threshold(i, i_max, T_s=0.0001, T_f=0.0004):
    """
    Calculates the exponentially ascending gradient threshold for iteration i.
    
    Args:
        i (int): Current iteration index.
        i_max (int): Total number of training iterations.
        T_s (float): Initial threshold value.
        T_f (float): Final threshold value.
        
    Returns:
        float: The threshold T_i for the current iteration.
    """
    if i_max == 0:
        return T_f
    
    # Normalized progress
    progress = i / i_max
    
    # Log-linear interpolation followed by exponentiation
    # Formula: T_i = exp( ln(T_s) * (1 - progress) + ln(T_f) * progress )
    log_T_i = math.log(T_s) * (1 - progress) + math.log(T_f) * progress
    T_i = math.exp(log_T_i)
    
    return T_i

def densification_check(gaussian, i, i_max, T_s=0.0001, T_f=0.0004):
    """
    Determines if a Gaussian should be densified based on the exponential threshold.
    
    Args:
        gaussian: Object containing 'modified_gradient_magnitude' (tau'_k).
        i (int): Current iteration index.
        i_max (int): Total number of training iterations.
        T_s (float): Initial threshold.
        T_f (float): Final threshold.
        
    Returns:
        bool: True if densification (split/clone) should be considered.
    """
    T_i = calculate_exponential_threshold(i, i_max, T_s, T_f)
    
    # Compare modified gradient magnitude against the dynamic threshold
    if gaussian.modified_gradient_magnitude >= T_i:
        return True
    else:
        return False
```

**Parameters:**
- $T_s$: Initial gradient threshold value. Typical value: 0.0001 [S1].
- $T_f$: Final gradient threshold value. Typical value: 0.0004 [S1].
- $i_{\max}$: Total number of training iterations [S1].

**Edge Cases and Pitfalls:**
- Precision: The use of logarithms and exponentials requires careful handling of floating-point precision, especially if $T_s$ or $T_f$ are very small [S1].
- Boundary Conditions: At $i=0$, $T_i = T_s$. At $i=i_{\max}$, $T_i = T_f$ [S1].
- Degenerate Input: If $i_{\max}$ is zero, the threshold logic is undefined; however, in practice, training iterations are always positive [G].

## Subtopics

### Variational Densify Threshold
A densification strategy that uses accumulated gradient norms instead of a single gradient norm to detect blurred areas and dynamically adjusts the threshold for cloning/splitting Gaussians [S2]. It addresses the pitfall where single gradient norms fail to detect overly blurred areas because gradient signals from different pixels may cancel out, resulting in low overall magnitude despite poor reconstruction [S2]. It computes the gradient norm for each pixel influenced by a Gaussian during the backward pass, accumulates these norms to get $S_{grad}$, and compares $S_{grad}$ against a threshold $\lambda_\alpha$ [S2]. The optimal $\lambda_\alpha$ found is 0.00045, compared to the vanilla 3DGS value of 0.0002 [S2]. Experiments on scenes (flowers, garden, treehill, counter, drjohnson) showed optimal performance around $\lambda_\alpha = 0.00045$, with PSNR, SSIM, and LPIPS evaluated at intervals of 0.00005 from 0.0002 to 0.0006 [S2].

## Operating characteristics
- **Time Complexity:** The calculation of $T_i$ is $O(1)$ per iteration per Gaussian, adding negligible overhead to the training loop [S1].
- **Memory Complexity:** No additional memory structures are required beyond the scalar threshold value [S1].
- **Measured Performance:** In ablation studies on 13 scenes, deactivating the exponential gradient threshold led to more Gaussian splats being generated but showed a smaller difference in PSNR compared to other components like extent correction [S1]. The method allows the number of Gaussians to rise much quicker during early training steps and decline towards the end, stabilizing after 15,000 iterations to a similar number as standard 3DGS [S1].
- **Strengths:** Enables fast convergence early in training by allowing many Gaussians to densify, and prevents overfitting later by restricting densification to only those with very high positional gradients [S1].
- **Limitations:** Does not significantly improve PSNR on its own compared to other densification improvements; its primary benefit is controlling the number of Gaussians [S1].
- **When to use:** When the goal is to control the growth rate of Gaussian primitives to balance reconstruction speed and final scene compactness [S1].
- **When not to use:** When computational resources are extremely limited and the overhead of dynamic threshold calculation is prohibitive (though minimal), or when other densification strategies (like Variational Densify Threshold) are preferred for handling blurred areas [S2].

## Background
Originated in the paper "Improving Adaptive Density Control for 3D Gaussian Splatting" [S1]. The method was developed to address the limitations of static gradient thresholds in 3D Gaussian Splatting, where a fixed threshold either hinders convergence early on or causes overfitting late in training [S1]. It builds on the standard 3DGS densification algorithm by introducing a time-dependent threshold schedule [S1].

## Sources
[S1] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
[S2] Minigs: efficient 3D Gaussian splatting with full factors weighted pruning for scene representation, https://doi.org/10.1007/s00530-025-02104-w, fetched 2026-10-26
