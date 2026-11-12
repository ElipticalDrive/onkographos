# Stepsize Annealing

Also called: Progressive Supersampling, Training Step Size Annealing

## Overview
Step size annealing is a training acceleration strategy for differentiable volume rendering and neural radiance fields (NeRF) that varies the sampling rate (inverse of step size) across training epochs. It solves the problem of high computational cost during early training stages by starting with coarse sampling (large step size) and progressively refining to fine sampling (small step size) as the model converges [S1, unofficial]. It sets itself apart from static sampling strategies by dynamically adjusting the resolution of the volume exploration based on training progress, achieving comparable visual quality to fixed high-resolution training with significantly reduced time [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Current epoch $e$, total epochs $E$, lower sampling rate bound $s_l$, upper sampling rate bound $s_h$.
- **Output:** Sampling rate $s(e)$ for the current epoch.

**Data Structures:**
No persistent data structures are required; the method is stateless with respect to the model weights, relying only on epoch counters and hyperparameters.

**Parameters:**
- $s_l$: Lower bound of the sampling rate range. Typical value: 0.1 [S1, unofficial].
- $s_h$: Upper bound of the sampling rate range. Typical value: 2.0 [S1, unofficial].
- $E$: Total number of training epochs.

**Formula:**
The sampling rate $s(e)$ at epoch $e$ is calculated using a quadratic interpolation:
$$ s(e) = s_l \left(1 - \left(\frac{e}{E}\right)^2\right) + \left(\frac{e}{E}\right)^2 s_h $$
where:
- $s(e)$ is the sampling rate at epoch $e$.
- $s_l$ is the lower bound.
- $s_h$ is the upper bound.
- $e$ is the current epoch index (0-based or 1-based, consistent with $E$).
- $E$ is the total number of epochs.

**Steps:**
1. Initialize training with epoch counter $e = 0$ and total epochs $E$.
2. For each epoch $e$ from 0 to $E-1$:
   a. Calculate the normalized progress factor $\alpha = (e/E)^2$.
   b. Compute the sampling rate $s(e) = s_l (1 - \alpha) + s_h \alpha$.
   c. Use $s(e)$ to determine the step size for ray marching during this epoch's training iterations. Note that step size $\Delta t$ is inversely proportional to sampling rate $s$ (e.g., $\Delta t \propto 1/s$).
   d. Perform forward and backward passes to optimize the volume rendering model.
3. End training when $e = E$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sampling_rate(epoch, total_epochs, sl, sh):
    """
    Compute the sampling rate for a given epoch using quadratic annealing.
    
    Args:
        epoch (int): Current epoch index (0-based).
        total_epochs (int): Total number of epochs.
        sl (float): Lower bound of sampling rate.
        sh (float): Upper bound of sampling rate.
        
    Returns:
        float: Sampling rate for the current epoch.
    """
    if total_epochs <= 0:
        return sh  # Fallback to upper bound if no epochs defined
        
    # Calculate normalized progress squared
    progress_ratio = epoch / total_epochs
    alpha = progress_ratio * progress_ratio
    
    # Linear interpolation weighted by alpha
    sampling_rate = sl * (1.0 - alpha) + sh * alpha
    
    return sampling_rate
```

**Edge Cases and Pitfalls:**
- **Division by Zero:** If $E=0$, the formula is undefined. A guard should return $s_h$ or handle the initialization separately [G].
- **Epoch Indexing:** The formula assumes $e$ ranges from 0 to $E-1$. If 1-based indexing is used ($e=1 \dots E$), the formula should be adjusted to $((e-1)/E)^2$ to ensure the first epoch uses $s_l$ and the last uses $s_h$ [G].
- **Precision:** Floating point precision is sufficient for standard epoch counts. No special handling is required.

## Subtopics

### Max Time-Step Scaling
A strategy to keep training stable under large maximum time steps $T$ by starting with a reduced maximum time step to avoid peak memory cost from dense early samples, then restoring $T$ once sampling becomes stable [S2, unofficial]. At the beginning of each training stage, the maximum time step is set to 2 for the initial epoch, then restored to 8 (or $T$) [S2, unofficial]. This preserves full temporal capacity while removing the initial memory bottleneck caused by unfiltered sampling points in early training [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- **Time:** Reduces training time by an average of 33.4% compared to training with a constant small step size (e.g., $s=2.0$) [S1, unofficial].
- **Memory:** Max Time-Step Scaling variant reduces peak memory cost during early training stages by limiting the maximum time step [S2, unofficial].

**Measured Performance:**
- On datasets with isotropic voxel resolution of $256 \times 256 \times 256$, step size annealing with $s_l=0.1, s_h=2.0$ produces visual results comparable to fixed $s=2.0$ but with 33.4% less training time [S1, unofficial].
- Compared to fixed small step sizes, it achieves similar performance with accelerated training [S1, unofficial].

**Scaling:**
- Performance gains are observed in differentiable volume rendering models (e.g., DeepDVR) and spike-based NeRFs (PATA) [S1, unofficial][S2, unofficial].
- Benefits diminish when increasing sampling rate beyond 1.0 in terms of perceptual quality, making annealing efficient [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** Fast convergence at low sampling rates in early epochs; refinement at high sampling rates in later epochs; reduced training time; stable memory usage with Max Time-Step Scaling.
- **Limitations:** Requires careful tuning of $s_l$ and $s_h$; effectiveness may depend on the specific rendering architecture and dataset [S1, unofficial].

**When to Use:**
- Use when training time is a bottleneck and visual quality is comparable across a range of sampling rates.
- Use when memory constraints are tight during early training stages (with Max Time-Step Scaling).

**When Not to Use:**
- Use when real-time inference requires fixed, predictable sampling rates and training time is not a concern.
- Use when the rendering model is highly sensitive to early-stage sampling artifacts that annealing might introduce.

## Background
Step size annealing was introduced as a training strategy for differentiable volume rendering models to accelerate training by varying the sampling rate across epochs [S1, unofficial]. It was proposed in the context of Deep Direct Volume Rendering (DeepDVR) to optimize transfer functions and rendering parameters end-to-end [S1, unofficial]. The method leverages the observation that low sampling rates provide better convergence efficiency in early epochs, while high sampling rates are needed for refinement [S1, unofficial]. Max Time-Step Scaling was later introduced in the context of Adaptive Time-step Training for Spike-Based Neural Radiance Fields (PATA) to handle memory bottlenecks in early training [S2, unofficial].

## Sources
[S1] Deep Direct Volume Rendering: Learning Visual Feature Mappings From Exemplary Images, https://doi.org/10.48550/arxiv.2106.05429, fetched 2026-10-26 (unofficial)
[S2] Adaptive Time-step Training for Enhancing Spike-Based Neural Radiance Fields, https://arxiv.org/abs/2507.23033v2, fetched 2026-10-27 (unofficial)
