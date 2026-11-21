# Learning Rate Schedule with Delay

Also called: Learning Rate Scheduler with Delay, lr_delay_mult schedule, reverse cosine decay learning rate

## Overview
A learning rate scheduler that applies a delay multiplier to the initial learning rate at the start of optimization, easing it back to the normal rate after a specified number of steps [S1, S2, unofficial]. It combines a 'reverse cosine decay' delay factor with a standard log-linear interpolation between initial and final learning rates [S1, S2, unofficial]. It sets itself apart from its parent [Delayed Cosine Schedule](11854-delayed-cosine-schedule.md) by explicitly defining the delay mechanism as a multiplicative factor applied to a base log-linear schedule, rather than just delaying the start of a base schedule [S1, S2, unofficial].

## How it works
**Inputs:**
- `step`: Current optimization step (integer).
- `max_steps`: Total number of optimization steps (integer).
- `lr_init`: Initial learning rate (float).
- `lr_final`: Final learning rate (float).
- `lr_delay_steps`: Number of steps over which the delay multiplier transitions (integer).
- `lr_delay_mult`: The minimum multiplier value applied during the delay phase (float).

**Outputs:**
- `learning_rate`: The computed learning rate for the current step (float).

**Data Structures:**
None required. The method is stateless and computes the value directly from inputs.

**Parameters:**
- `lr_delay_steps`: The number of steps during which the delay multiplier is applied. If 0, no delay is applied [S1, S2, unofficial].
- `lr_delay_mult`: The multiplier value applied to the learning rate during the delay phase. Default is 1.0 [S1, S2, unofficial].

**Steps:**
1. If `step < 0` or both `lr_init` and `lr_final` are 0.0, return 0.0 to disable the parameter [S1, S2, unofficial].
2. If `lr_delay_steps <= 0`, set `delay_rate = 1.0` [S2, unofficial].
3. If `lr_delay_steps > 0`, calculate `delay_rate` using reverse cosine decay:
   $$
   delay\_rate = lr\_delay\_mult + (1 - lr\_delay\_mult) \times \sin\left(0.5 \times \pi \times \text{clip}\left(\frac{step}{lr\_delay\_steps}, 0, 1\right)\right)
   $$
   Otherwise, `delay_rate = 1.0` [S1, S2, unofficial].
4. Calculate normalized time `t`:
   $$
   t = \text{clip}\left(\frac{step}{max\_steps}, 0, 1\right)
   $$
   [S1, S2, unofficial]
5. Calculate log-linear interpolation `log_lerp`:
   $$
   log\_lerp = \exp\left(\log(lr\_init) \times (1 - t) + \log(lr\_final) \times t\right)
   $$
   [S1, S2, unofficial]
6. Return `delay_rate * log_lerp` [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def learning_rate_schedule_with_delay(step, max_steps, lr_init, lr_final, lr_delay_steps, lr_delay_mult):
    # Step 1: Disable if invalid or zero rates
    if step < 0 or (lr_init == 0.0 and lr_final == 0.0):
        return 0.0
    
    # Step 2 & 3: Calculate delay_rate
    if lr_delay_steps <= 0:
        delay_rate = 1.0
    else:
        # Clip step / lr_delay_steps to [0, 1]
        ratio = step / lr_delay_steps
        if ratio < 0:
            ratio = 0.0
        elif ratio > 1:
            ratio = 1.0
        
        # Reverse cosine decay formula
        # sin(0.5 * pi * ratio) goes from 0 to 1 as ratio goes from 0 to 1
        # delay_rate goes from lr_delay_mult (at ratio=0) to 1.0 (at ratio=1)
        delay_rate = lr_delay_mult + (1.0 - lr_delay_mult) * math.sin(0.5 * math.pi * ratio)
    
    # Step 4: Calculate normalized time t
    t = step / max_steps
    if t < 0:
        t = 0.0
    elif t > 1:
        t = 1.0
    
    # Step 5: Calculate log-linear interpolation
    # Handle edge case where lr_init or lr_final is 0 to avoid log(0)
    if lr_init <= 0 or lr_final <= 0:
        # If either is 0, the log-linear interpolation is problematic.
        # However, Step 1 checks if BOTH are 0.
        # If one is 0, we might need a fallback or assume positive rates for log.
        # Standard implementation assumes positive rates for log_lerp.
        # If strict adherence to log is required, clamp to small epsilon or handle separately.
        # Based on source notes, we proceed with log assuming valid inputs for interpolation.
        # [G] Guard against log(0) by using a small epsilon if necessary, 
        # but sources imply standard log-linear usage.
        log_lr_init = math.log(lr_init) if lr_init > 0 else -1e9 # [G] Fallback for log(0)
        log_lr_final = math.log(lr_final) if lr_final > 0 else -1e9 # [G] Fallback for log(0)
    else:
        log_lr_init = math.log(lr_init)
        log_lr_final = math.log(lr_final)
        
    log_lerp = math.exp(log_lr_init * (1.0 - t) + log_lr_final * t)
    
    # Step 6: Return product
    return delay_rate * log_lerp
```

**Edge Cases and Pitfalls:**
- **Zero Learning Rates:** If both `lr_init` and `lr_final` are 0.0, the function returns 0.0 [S1, S2, unofficial]. If only one is 0.0, `log(0)` is undefined. The sources do not explicitly specify behavior for mixed zero/non-zero rates, but standard log-linear interpolation requires positive values. Implementers should ensure `lr_init` and `lr_final` are positive for the log-linear step, or handle the zero case explicitly [G].
- **Division by Zero:** If `max_steps` is 0, division by zero occurs in step 4. If `lr_delay_steps` is 0, the condition in step 3 handles it by setting `delay_rate = 1.0` [S2, unofficial].
- **Negative Steps:** Handled in step 1 [S1, S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The method is used in the context of Gaussian Splatting optimization, specifically referenced in the MonoGS and FastGS repositories [S1, S2, unofficial]. It is part of the training configuration for NeRF-like radiance field optimization [S1, S2, unofficial].

## Sources
[S1] muskie82/MonoGS, https://github.com/muskie82/MonoGS, fetched 2026-10-27 (unofficial)
[S2] fastgs/FastGS, https://github.com/fastgs/FastGS, fetched 2026-10-27 (unofficial)
