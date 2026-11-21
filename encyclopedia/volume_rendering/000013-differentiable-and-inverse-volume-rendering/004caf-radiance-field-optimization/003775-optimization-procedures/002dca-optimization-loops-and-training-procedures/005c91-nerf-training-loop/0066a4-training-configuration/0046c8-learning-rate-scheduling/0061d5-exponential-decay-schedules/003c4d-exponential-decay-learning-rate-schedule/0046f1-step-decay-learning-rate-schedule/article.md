# Step Decay Learning Rate Schedule
Also called: StepSchedule
## Overview
A step-wise decay schedule that reduces a learning rate by a fixed multiplicative factor at regular intervals until a maximum number of decays is reached [S2, unofficial]. It differs from the parent [Exponential Decay Learning Rate Schedule](9145-exponential-decay-learning-rate-schedule.md) by applying discrete, uniform reductions at specific step boundaries rather than a continuous exponential curve [S2, unofficial].
## How it works
**Inputs:** Current training step (integer), initial learning rate, decay interval, decay factor, maximum number of decays, and optionally a final learning rate value.
**Outputs:** The current learning rate value.

**Data Structures:**
The schedule is stateless regarding the learning rate value itself, relying only on the current step index and configuration constants.

**Parameters:**
- `initial_value`: The starting learning rate. Typical value: 0.01 [S1].
- `decay_interval`: The number of steps between each decay event [S2, unofficial].
- `decay_factor`: The multiplicative factor applied per decay. Typical value: 0.33 [S1].
- `max_decays`: The maximum number of decays allowed [S2, unofficial].
- `final_value`: The learning rate after `max_decays` have occurred. If not provided, it is computed as `initial_value * decay_factor**max_decays` [S2, unofficial].

**Algorithm:**
1. Calculate the decay phase: `phase = step // decay_interval`.
2. If `phase >= max_decays`, return `final_value` [S2, unofficial].
3. Otherwise, return `initial_value * decay_factor**phase` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def step_decay_learning_rate(step, initial_value, decay_interval, decay_factor, max_decays, final_value=None):
    """
    Computes the learning rate for a given step using step decay.
    
    Args:
        step: Current training step (integer).
        initial_value: Starting learning rate.
        decay_interval: Steps between decays.
        decay_factor: Multiplicative factor per decay.
        max_decays: Maximum number of decays.
        final_value: Optional explicit final value.
        
    Returns:
        Current learning rate.
    """
    # Compute the number of full decay intervals completed
    phase = step // decay_interval
    
    # Determine the final value if not explicitly provided
    if final_value is None:
        final_value = initial_value * (decay_factor ** max_decays)
        
    # If the phase exceeds the maximum allowed decays, clamp to final value
    if phase >= max_decays:
        return final_value
    
    # Apply the decay factor for the current phase
    return initial_value * (decay_factor ** phase)
```

**Edge Cases and Pitfalls:**
- **Warm-up:** In specific implementations (e.g., Instant-NGP-based NeRF), a linear warm-up phase is applied for the first 1K iterations before the step decay begins [S1]. The core step decay formula does not account for warm-up; this must be handled as a separate pre-processing step or conditional logic before applying the decay [S1].
- **Boundary Conditions:** The decay occurs at multiples of `decay_interval`. Steps strictly less than `decay_interval` result in no decay (phase 0) [S2, unofficial].
- **Integer Division:** The use of integer division (`//`) ensures that the decay is discrete. Floating point inaccuracies in step counting should be avoided [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(1)$ per step. The calculation involves a single division, a comparison, and a power operation [S2, unofficial].
- Memory: $O(1)$. No state is stored between steps [S2, unofficial].

**Measured Performance:**
- In Instant-NGP-based NeRF training, this schedule is used with an initial learning rate of 0.01 over 20K total iterations [S1].
- Specific decay points are at iterations 9K, 12K, 15K, 17K, and 19K, using a decay factor of 0.33 [S1]. This implies a variable `decay_interval` or specific hard-coded steps rather than a uniform interval in that specific implementation, which contradicts the generic "regular intervals" definition in [S2, unofficial]. The sources disagree on whether the intervals are uniform or fixed specific points. [S1] lists specific iterations, while [S2, unofficial] defines a uniform `decay_interval`. Implementers should verify if the specific application requires uniform intervals or fixed checkpoints [S1][S2, unofficial].

**Strengths and Limitations:**
- **Strengths:** Simple to implement and debug; provides clear, discrete reductions in learning rate [S2, unofficial].
- **Limitations:** Less smooth than exponential decay; may cause instability if the decay factor is too aggressive or intervals are too short [S2, unofficial].

**When to Use:**
- When discrete, predictable reductions in learning rate are desired [S2, unofficial].
- When the training process benefits from specific checkpoint-based adjustments rather than continuous decay [S1].

## Background
**Origin:**
The step decay schedule is a standard optimization technique in deep learning. In the context of NeRF, it is used in the Nerfies framework [S2, unofficial] and Instant-NGP-based NeRF variants [S1].

**History and Development:**
- The Nerfies implementation defines the schedule with parameters `decay_interval`, `decay_factor`, and `max_decays` [S2, unofficial].
- Instant-NGP-based NeRF implementations use a variant with specific decay points (9K, 12K, 15K, 17K, 19K) and a warm-up phase [S1].

**Where it is used:**
- Nerfies training scripts [S2, unofficial].
- Instant-NGP-based NeRF compression studies [S1].

## Sources
[S1] How Far can we Compress Instant-NGP-Based NeRF?, https://doi.org/10.1109/cvpr52733.2024.01921, fetched 2026-10-26
[S2] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
