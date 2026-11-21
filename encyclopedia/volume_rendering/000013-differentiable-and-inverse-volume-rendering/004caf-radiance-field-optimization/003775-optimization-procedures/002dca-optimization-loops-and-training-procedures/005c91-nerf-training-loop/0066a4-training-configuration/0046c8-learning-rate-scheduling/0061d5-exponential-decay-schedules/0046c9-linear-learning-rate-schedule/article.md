# Linear Learning Rate Schedule

Also called: Linear Interpolation Schedule, LinearSchedule

## Overview
A learning rate schedule that transitions from an initial value to a final value over a specified number of steps using linear interpolation [S1, unofficial]. It provides a deterministic, monotonic decay path for optimization parameters in radiance-field training [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**: `step` (current optimization step index, integer $\ge 0$), `initial_value` (starting learning rate, float), `final_value` (target learning rate at completion, float), `num_steps` (total steps for the transition, integer) [S1, unofficial].
*   **Output**: The computed learning rate for the current step (float) [S1, unofficial].

**Data Structures**
No persistent data structures are required; the state is purely a function of the current step index and the fixed parameters [G].

**Parameters**
*   `initial_value`: The learning rate at step 0 [S1, unofficial].
*   `final_value`: The learning rate at step `num_steps` [S1, unofficial].
*   `num_steps`: The total number of steps over which the interpolation occurs [S1, unofficial].

**Algorithm**
1.  Compute the progress ratio $\alpha$ as the fraction of steps completed, capped at 1.0:
    $$ \alpha = \min\left(\frac{\text{step}}{\text{num\_steps}}, 1.0\right) $$
2.  Compute the learning rate $lr$ by linearly interpolating between `initial_value` and `final_value` using $\alpha$:
    $$ lr = (1.0 - \alpha) \cdot \text{initial\_value} + \alpha \cdot \text{final\_value} $$

**Edge Cases and Pitfalls**
*   **Division by Zero**: If `num_steps` is 0, the division in step 1 is undefined. The implementation must ensure `num_steps > 0` or handle this case explicitly (e.g., returning `initial_value`) [G].
*   **Step Bounds**: If `step` exceeds `num_steps`, $\alpha$ is clamped to 1.0, ensuring the learning rate remains at `final_value` rather than extrapolating [S1, unofficial].
*   **Precision**: Using floating-point arithmetic for `step / num_steps` is standard. For very large `num_steps`, precision loss is negligible for typical learning rate magnitudes [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_schedule(step: int, initial_value: float, final_value: float, num_steps: int) -> float:
    """
    Compute the learning rate using a linear schedule.
    
    Args:
        step: Current optimization step index.
        initial_value: Starting learning rate.
        final_value: Target learning rate at completion.
        num_steps: Total number of steps for the transition.
        
    Returns:
        The computed learning rate.
    """
    if num_steps <= 0:
        # Guard against division by zero; behavior is undefined in sources for num_steps=0 [G]
        return initial_value
        
    # Compute progress ratio, capped at 1.0
    alpha = min(step / num_steps, 1.0)
    
    # Linear interpolation
    lr = (1.0 - alpha) * initial_value + alpha * final_value
    
    return lr
```

## Subtopics

### [Cosine Learning Rate Schedule](11851-cosine-learning-rate-schedule.md)
A learning rate schedule that transitions from an initial value to a final value using a cosine function to ease the change. It differs from the parent [Linear Learning Rate Schedule](11850-linear-learning-rate-schedule.md) by providing a non-linear, smooth decay curve rather than a linear interpolation, which can help stabilize optimization in radiance-field training by reducing the learning rate more gradually in the later stages.

### Linear Stage Transition
A linear interpolation method to avoid abrupt transitions between training stages by blending the previous coarser representation with the current less-merged one [S3]. Transitions between stages are smoothed using linear interpolation over 2k iterations [S3]. It interpolates the decoded quantity $z$ as $z^{(s)} = (1-\lambda) z^{(s-1)} + \lambda \bar{z}^{(s)}$, where $\lambda \in [0,1]$ is the transition coefficient [S3].

## Operating characteristics
*   **Time Complexity**: $O(1)$ per step [G]. The calculation involves a single division, a minimum, and two multiplications and an addition [G].
*   **Memory Complexity**: $O(1)$. No history or state needs to be stored beyond the current step counter [G].
*   **Strengths**: Simple to implement and understand [G]. Provides a predictable, linear decay path [G]. Useful when a steady, constant reduction in learning rate is desired without the compounding effect of exponential decay [G].
*   **Limitations**: The rate of change is constant, which may not be optimal for all loss landscapes [G]. It lacks the smoothness of cosine schedules or the adaptive nature of exponential decay [G].
*   **When to use**: When a simple, deterministic decay is sufficient and computational overhead of more complex schedules is undesirable [G].
*   **When not to use**: When the optimization landscape requires a smoother transition (consider Cosine) or a faster initial decay followed by a slower tail (consider Exponential Decay) [G].

## Background
It is implemented in the Nerfies library [S1, unofficial].

## Sources
[S1] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
[S2] AlexSheldrick/density_bounded_ray_weights, https://github.com/AlexSheldrick/density_bounded_ray_weights, fetched 2026-10-27 (unofficial)
[S3] GlobalSplat: Efficient Feed-Forward 3D Gaussian Splatting via Global Scene Tokens, https://doi.org/10.1007/978-3-032-37595-7_10, fetched 2026-10-29
