# Cosine Learning Rate Schedule
Also called: CosineEasingSchedule
## Overview
A learning rate schedule that eases slowly using a cosine function to transition between initial and final values [S1, unofficial]. It differs from the parent [Linear Learning Rate Schedule](11850-linear-learning-rate-schedule.md) by using a smooth, non-linear decay curve rather than a linear interpolation, which can help stabilize training in radiance-field optimization by reducing the learning rate more aggressively in the middle of the schedule [S1, unofficial].
## How it works
**Inputs and Outputs:**
- `step`: Current training step (integer).
- `num_steps`: Total number of training steps (integer).
- `initial_value`: Starting learning rate (float).
- `final_value`: Target learning rate at `num_steps` (float).
- Output: The computed learning rate for the current step (float).

**Data Structures:**
No complex data structures are required; the computation is stateless and depends only on the current step and configuration parameters.

**Steps:**
1. Compute the normalized progress `alpha` as the ratio of the current step to the total steps, clamped to a maximum of 1.0:
   $$ \alpha = \min\left(\frac{\text{step}}{\text{num\_steps}}, 1.0\right) $$
2. Calculate the total scale of the transition:
   $$ \text{scale} = \text{final\_value} - \text{initial\_value} $$
3. Clamp `alpha` to the range $[0.0, 1.0]$ to handle edge cases where `step` might be negative or exceed `num_steps` due to implementation details:
   $$ x = \min(\max(\alpha, 0.0), 1.0) $$
4. Compute the cosine factor. The term $\cos(\pi x + \pi)$ ensures that at $x=0$, the factor is $-1$ (yielding `initial_value`), and at $x=1$, the factor is $1$ (yielding `final_value`):
   $$ \text{factor} = 0.5 \cdot (1 + \cos(\pi x + \pi)) $$
5. Return the interpolated value:
   $$ \text{lr} = \text{initial\_value} + \text{scale} \cdot \text{factor} $$

**Formula:**
$$ v = v_{init} + (v_{final} - v_{init}) \cdot 0.5 \cdot (1 + \cos(\pi x + \pi)) $$
where $x = \min(\max(\text{step}/\text{num\_steps}, 0), 1)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def cosine_learning_rate(step, num_steps, initial_value, final_value):
    # Step 1: Compute normalized progress
    alpha = min(step / num_steps, 1.0)
    
    # Step 2: Compute scale
    scale = final_value - initial_value
    
    # Step 3: Clamp x
    x = min(max(alpha, 0.0), 1.0)
    
    # Step 4 & 5: Compute and return learning rate
    # cos(pi * x + pi) is equivalent to -cos(pi * x)
    # 0.5 * (1 + cos(pi * x + pi)) transitions from 0 to 1 as x goes 0 to 1
    return initial_value + scale * 0.5 * (1 + math.cos(math.pi * x + math.pi))
```

**Parameters:**
- `initial_value`: Typically the base learning rate used in the optimizer (e.g., $10^{-3}$ or $10^{-4}$).
- `final_value`: Often set to a small value like $10^{-6}$ or $0$ to ensure convergence.
- `num_steps`: Total number of optimization steps, determined by the dataset size and batch count.

**Edge Cases and Pitfalls:**
- If `num_steps` is 0, division by zero occurs. Ensure `num_steps > 0` [G].
- If `step` is negative, the clamping in step 3 ensures `x` starts at 0, preventing invalid cosine arguments or out-of-bounds interpolation [S1, unofficial].
- Precision issues with floating-point arithmetic are minimal given the smooth nature of the cosine function, but standard IEEE 754 double precision is recommended for stability [G].

## Subtopics

### [Delayed Cosine Schedule](11854-delayed-cosine-schedule.md)
A schedule wrapper that delays the start of a base learning rate schedule using a sinusoidal rate multiplier. It multiplies the output of a base schedule by a delay rate that transitions from a minimum value to 1.0 over a specified number of steps.

## Operating characteristics
- **Time Complexity:** $O(1)$ per step, as it involves a constant number of arithmetic operations and a single trigonometric evaluation.
- **Memory Complexity:** $O(1)$, requiring no additional storage beyond the current step and configuration parameters.
- **Strengths:** Provides a smooth transition that can help avoid sharp changes in gradient magnitude, potentially leading to more stable convergence in complex radiance-field optimizations [S1, unofficial].
- **Limitations:** The smooth decay might be too slow for some tasks that benefit from a sharper initial reduction in learning rate, compared to step decay or linear schedules [S1, unofficial].
- **When to use:** Suitable for training NeRFs and radiance fields where smooth optimization dynamics are preferred [S1, unofficial].
- **When not to use:** If rapid convergence is needed in the early stages, a linear or step schedule might be more appropriate.

## Background
The Cosine Learning Rate Schedule is implemented in the Nerfies project, a research codebase for neural radiance fields [S1, unofficial]. It is part of the training configuration for optimizing radiance fields, building on the principles of exponential decay schedules and linear learning rate schedules [S1, unofficial]. The specific implementation is found in the `schedules` module of the Nerfies repository [S1, unofficial].

## Sources
[S1] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
