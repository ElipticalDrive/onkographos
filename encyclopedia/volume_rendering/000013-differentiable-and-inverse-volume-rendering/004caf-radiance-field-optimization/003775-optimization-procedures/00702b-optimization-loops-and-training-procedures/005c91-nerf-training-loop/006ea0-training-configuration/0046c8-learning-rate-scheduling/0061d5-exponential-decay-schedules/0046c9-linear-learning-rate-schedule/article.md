# Linear Learning Rate Schedule

Also called: Linear Interpolation Schedule, LinearSchedule

## Overview
A linear interpolation schedule that transitions a learning rate from an initial value to a final value over a specified number of steps [S1, unofficial]. It differs from its sibling [Exponential Decay Learning Rate Schedule](9145-exponential-decay-learning-rate-schedule.md) by using a constant rate of change rather than a multiplicative decay, and from [Cosine Decay Learning Rate Schedule](14118-cosine-decay-learning-rate-schedule.md) by lacking the smooth, non-linear curvature that reduces the learning rate more aggressively in the middle of the schedule [S1, unofficial].

## How it works
**Inputs and outputs:**
- Input: current training step $t$, total steps $T$, initial learning rate $lr_{init}$, final learning rate $lr_{final}$.
- Output: current learning rate $lr_t$.

**Data structures:**
No persistent data structures are required; the schedule is stateless with respect to the optimizer's internal state, depending only on the step counter.

**Steps:**
1. Compute the progress ratio $\alpha$ as the minimum of the normalized step count and 1.0:
   $$ \alpha = \min\left(\frac{t}{T}, 1.0\right) $$
2. Compute the learning rate via linear interpolation:
   $$ lr_t = (1 - \alpha) \cdot lr_{init} + \alpha \cdot lr_{final} $$

**Parameters:**
- $lr_{init}$: Starting value of the schedule [S1, unofficial].
- $lr_{final}$: Target value at completion [S1, unofficial].
- $T$ (or $num\_steps$): Total number of steps for the transition [S1, unofficial].

**Edge cases and pitfalls:**
- If $t > T$, the schedule clamps to $lr_{final}$ [S1, unofficial].
- Some implementations define the schedule as log-linear interpolation, equivalent to exponential decay, rather than linear interpolation of the rate itself [S2, unofficial]. In such cases, the formula becomes $lr_t = lr_{init} \cdot (lr_{final} / lr_{init})^{t/T}$ [S2, unofficial].
- An optional initial delay phase can be applied where the learning rate is scaled by a multiplier $lr_{delay\_mult}$ for the first $lr_{delay\_steps}$ steps, then eases back to the normal rate [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_schedule(step, num_steps, initial_value, final_value, 
                    lr_delay_steps=0, lr_delay_mult=1.0):
    """
    Compute learning rate using linear interpolation.
    
    Args:
        step: Current training step (integer).
        num_steps: Total number of steps for the transition (integer).
        initial_value: Starting learning rate (float).
        final_value: Target learning rate at completion (float).
        lr_delay_steps: Number of steps to apply delayed initial learning rate (integer).
        lr_delay_mult: Multiplier for the initial learning rate during delay phase (float).
        
    Returns:
        Current learning rate (float).
    """
    # Handle delay phase
    if step < lr_delay_steps:
        return initial_value * lr_delay_mult
    
    # Compute alpha
    alpha = min(step / num_steps, 1.0)
    
    # Linear interpolation
    lr = (1.0 - alpha) * initial_value + alpha * final_value
    
    return lr
```

## Subtopics

### [Cosine Learning Rate Schedule](11851-cosine-learning-rate-schedule.md)
A learning rate schedule that eases slowly using a cosine function to transition between initial and final values. It differs from the parent [Linear Learning Rate Schedule](11850-linear-learning-rate-schedule.md) by using a smooth, non-linear decay curve rather than a linear interpolation, which can help stabilize training in radiance-field optimization by reducing the learning rate more aggressively in the middle of the schedule.

### Linear Stage Transition
A linear interpolation method to avoid abrupt transitions between training stages by blending the previous coarser representation with the current less-merged one [S3]. Transitions between stages are smoothed using linear interpolation over 2k iterations [S3]. The interpolated decoded quantity $z$ is computed as:
$$ z^{(s)} = (1-\lambda) z^{(s-1)} + \lambda \bar{z}^{(s)} $$
where $\lambda \in [0,1]$ is the transition coefficient [S3]. This procedure does not introduce new candidates but progressively relaxes the merging applied to the same underlying set of predictions [S3].

## Operating characteristics
Not given in the sources.

## Background
The linear interpolation schedule is implemented in the training code for Nerfies, a system for deformable neural radiance fields [S1, unofficial]. It is also referenced in the context of density-bounded ray weights [S2, unofficial]. The concept of linear stage transition is used in GlobalSplat, an efficient feed-forward 3D Gaussian Splatting method, to smooth transitions between training stages [S3].

## Sources
[S1] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
[S2] AlexSheldrick/density_bounded_ray_weights, https://github.com/AlexSheldrick/density_bounded_ray_weights, fetched 2026-10-27 (unofficial)
[S3] GlobalSplat: Efficient Feed-Forward 3D Gaussian Splatting via Global Scene Tokens, https://doi.org/10.1007/978-3-032-37595-7_10, fetched 2026-10-29
