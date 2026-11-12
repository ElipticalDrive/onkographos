# Annealing Schedules

Also called: Constant Schedule, Exponential Schedule, Linear Schedule, Parameter Annealing Framework

## Overview
Annealing Schedules provide a unified framework for decaying or maintaining hyperparameter values, primarily learning rates, during the training of neural radiance fields. Unlike specialized schedules that implement only one decay strategy, this framework supports constant, linear, and exponential interpolation between initial and final values [S1, unofficial]. It is distinct from [Exponential Decay Learning Rate Schedule](9145-exponential-decay-learning-rate-schedule.md) by using interpolation between bounds rather than a fixed decay rate, and from [Linear Learning Rate Schedule](11850-linear-learning-rate-schedule.md) by offering multiple decay modes within a single class [S1, unofficial].

## How it works
The method computes a scalar value based on the current training step, the total number of steps, and configured bounds.

**Inputs and Outputs:**
- `step`: Current training step (integer).
- `num_steps`: Total number of training steps (integer).
- `initial_value`: Starting value of the parameter (float).
- `final_value`: Ending value of the parameter (float).
- `schedule_type`: String identifier for the schedule ('constant', 'linear', 'exponential').
- Output: The computed parameter value (float32).

**Data Structures:**
No complex data structures are required. The implementation relies on vectorized operations over the step index.

**Steps:**
1. Calculate the progress ratio $\alpha$ as the minimum of the normalized step and 1.0:
   $$ \alpha = \min\left(\frac{\text{step}}{\text{num\_steps}}, 1.0\right) $$
2. Apply the specific formula based on `schedule_type`:
   - **Constant**: Return `initial_value` (or a specified constant `value`).
   - **Linear**: Interpolate between `initial_value` and `final_value`:
     $$ \text{value} = (1.0 - \alpha) \cdot \text{initial\_value} + \alpha \cdot \text{final\_value} $$
   - **Exponential**: Decay exponentially from `initial_value` to `final_value`:
     $$ \text{base} = \frac{\text{final\_value}}{\text{initial\_value}} $$
     $$ \text{value} = \text{initial\_value} \cdot (\text{base})^{\alpha} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import jax.numpy as jnp

def anneal_schedule(step, num_steps, initial_value, final_value, schedule_type='linear'):
    """
    Compute the annealed parameter value.
    
    Args:
        step: Current training step (int or jnp.ndarray).
        num_steps: Total number of training steps (int).
        initial_value: Starting value (float).
        final_value: Ending value (float).
        schedule_type: 'constant', 'linear', or 'exponential'.
        
    Returns:
        The computed parameter value (jnp.ndarray).
    """
    # Calculate progress ratio alpha, capped at 1.0
    alpha = jnp.minimum(step / num_steps, 1.0)
    
    if schedule_type == 'constant':
        # Constant schedule ignores step, returns initial_value (or a fixed value)
        # Note: The source code uses a specific 'value' attribute for constant, 
        # but typically this implies returning the initial or a fixed constant.
        # Here we assume it returns the initial_value if no separate constant is provided,
        # or a specific constant if defined. Based on source: `self.value`.
        # For this generic implementation, we return initial_value as the constant baseline.
        return jnp.full_like(step, initial_value, dtype=jnp.float32)
        
    elif schedule_type == 'linear':
        # Linear interpolation between initial and final
        return (1.0 - alpha) * initial_value + alpha * final_value
        
    elif schedule_type == 'exponential':
        # Exponential interpolation
        # Handle division by zero if initial_value is 0
        if initial_value == 0:
            return jnp.full_like(step, final_value, dtype=jnp.float32)
        
        base = final_value / initial_value
        return initial_value * (base ** alpha)
        
    else:
        raise ValueError(f"Unknown schedule type: {schedule_type}")
```

**Parameters:**
- `initial_value`: Starting value for the schedule. Typical values depend on the hyperparameter (e.g., $10^{-3}$ for learning rates) [S1, unofficial].
- `final_value`: Ending value. Often set to a small fraction of `initial_value` (e.g., $10^{-6}$) [S1, unofficial].
- `num_steps`: Total training steps. Determined by dataset size and batch size [S1, unofficial].

**Edge Cases:**
- If `initial_value` is 0, the exponential base calculation results in division by zero. The implementation must handle this, typically by returning `final_value` or `initial_value` directly [G].
- If `step` exceeds `num_steps`, $\alpha$ is capped at 1.0 to prevent extrapolation [S1, unofficial].

## Subtopics

### Constant Schedule
Maintains a fixed value throughout training. Useful for parameters that should not decay. [Constant Schedule](11850-constant-schedule.md)

### Linear Schedule
Decays linearly from initial to final. Simple and predictable. [Linear Learning Rate Schedule](11850-linear-learning-rate-schedule.md)

### Exponential Schedule
Decays exponentially. Provides faster initial decay and slower final decay. [Exponential Decay Learning Rate Schedule](9145-exponential-decay-learning-rate-schedule.md)

## Operating characteristics
- **Time Complexity**: $O(1)$ per step. The computation involves basic arithmetic operations.
- **Memory Complexity**: $O(1)$ if scalar, or $O(N)$ if applied to a batch of steps, where $N$ is the batch size.
- **Strengths**: Simple to implement, flexible, and covers common decay patterns.
- **Limitations**: Does not account for loss landscape geometry (unlike adaptive methods). The exponential schedule can be sensitive to the choice of `final_value` if `initial_value` is small.
- **When to use**: When a simple, predictable decay of hyperparameters is desired. Particularly useful in NeRF training where learning rates often decay over time [S1, unofficial].

## Background
The framework is implemented in the Nerfies repository, which provides code for "Deformable Neural Radiance Fields" [S1, unofficial]. It was developed by Keunhong Park et al. at Google and the University of Washington [S1, unofficial]. The implementation uses JAX for efficient computation [S1, unofficial].

## Sources
[S1] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
