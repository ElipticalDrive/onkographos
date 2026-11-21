# Cosine Learning Rate Schedule

Also called: CosineEasingSchedule

## Overview
A learning rate schedule that transitions from an initial value to a final value using a cosine function to ease the change [S1, unofficial]. It differs from the parent [Linear Learning Rate Schedule](11850-linear-learning-rate-schedule.md) by providing a non-linear, smooth decay curve rather than a linear interpolation, which can help stabilize optimization in radiance-field training by reducing the learning rate more gradually in the later stages [S1, unofficial].

## How it works
**Inputs and Outputs:**
- `step`: Current training step (integer).
- `num_steps`: Total number of training steps (integer).
- `initial_value`: Starting learning rate (float).
- `final_value`: Target learning rate at `num_steps` (float).
- Output: The computed learning rate for the current step (float).

**Data Structures:**
No complex data structures are required; the calculation is stateless and depends only on the current step and configuration parameters.

**Steps:**
1. Calculate the normalized progress `alpha` as the ratio of `step` to `num_steps`, clamped to a maximum of 1.0.
2. Calculate the total scale of change as `final_value - initial_value`.
3. Clamp `alpha` to the range [0.0, 1.0] to handle edge cases where `step` might be negative or exceed `num_steps` due to implementation details.
4. Compute the cosine factor using the formula: $0.5 \cdot (1 + \cos(\pi x + \pi))$, where $x$ is the clamped `alpha`.
5. Return the result: `initial_value + scale * cosine_factor`.

**Formula:**
$$v = v_{init} + (v_{final} - v_{init}) \cdot 0.5 \cdot (1 + \cos(\pi x + \pi))$$

Where:
- $v$ is the resulting learning rate.
- $v_{init}$ is `initial_value`.
- $v_{final}$ is `final_value`.
- $x = \min(\max(step/num\_steps, 0), 1)$.

**Parameters:**
- `initial_value`: Typically the peak learning rate used in the training setup.
- `final_value`: Often set to a small value (e.g., $10^{-6}$) or zero to converge the model.
- `num_steps`: The total duration of the training loop.

**Edge Cases and Pitfalls:**
- If `step` is negative, the clamping ensures $x=0$, returning `initial_value`.
- If `step` exceeds `num_steps`, the clamping ensures $x=1$, returning `final_value`.
- Precision issues with floating-point arithmetic in the cosine calculation are generally negligible for standard training scales.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def cosine_learning_rate(step, num_steps, initial_value, final_value):
    # Calculate normalized progress
    alpha = step / num_steps if num_steps > 0 else 0.0
    
    # Clamp alpha to [0.0, 1.0]
    if alpha < 0.0:
        alpha = 0.0
    elif alpha > 1.0:
        alpha = 1.0
        
    # Calculate scale
    scale = final_value - initial_value
    
    # Compute cosine factor
    # Note: cos(pi * x + pi) is equivalent to -cos(pi * x)
    cosine_factor = 0.5 * (1 + math.cos(math.pi * alpha + math.pi))
    
    # Return final learning rate
    return initial_value + scale * cosine_factor
```

## Subtopics

### [Delayed Cosine Schedule](11854-delayed-cosine-schedule.md)
A schedule wrapper that delays the start of a base learning rate schedule using a sinusoidal rate multiplier. It sets apart from standard cosine schedules by introducing a warm-up phase where the learning rate is suppressed, easing into the base schedule's trajectory. This prevents early instability in radiance-field optimization by avoiding large gradient updates during the initial training steps.

## Operating characteristics
- **Time Complexity:** $O(1)$ per step, as it involves basic arithmetic and a single trigonometric function evaluation.
- **Memory Complexity:** $O(1)$, requiring no additional storage beyond the current step and configuration values.
- **Strengths:** Provides a smooth, non-linear decay that can help the optimizer settle into minima more effectively than linear schedules. The cosine shape naturally slows down the learning rate as training progresses.
- **Limitations:** May converge slower than aggressive schedules if the initial learning rate is not well-tuned. The smooth decay might not be optimal for all types of loss landscapes in radiance-field optimization.
- **When to use:** Suitable for stable training scenarios where a gradual reduction in learning rate is beneficial. Often used in conjunction with warm-up phases (e.g., [Delayed Cosine Schedule](11854-delayed-cosine-schedule.md)).
- **When not to use:** If rapid convergence is required in the early stages, a linear or step schedule might be more appropriate.

## Background
The cosine learning rate schedule is implemented in the Nerfies project, a JAX-based framework for neural radiance fields [S1, unofficial]. It is part of the training configuration for optimizing radiance fields, providing a smooth transition in learning rates during the training loop.

## Sources
[S1] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
