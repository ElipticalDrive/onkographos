# Exponential Decay Learning Rate Schedule with Warm-up
Also called: Adam Optimization with Exponential LR Decay and Warm-up, Exponential Decay with Warm-up, Exponential LR Decay with Warm-up, Exposure Optimization, Warm-up and Exponential Decay LR Schedule

## Overview
This method defines a learning rate schedule for optimizing neural radiance fields, specifically targeting per-camera exposure parameters or the main network weights, combining a linear warm-up phase with an exponential decay trajectory. It distinguishes itself from [Cosine Decay Learning Rate Schedule](14118-cosine-decay-learning-rate-schedule.md) by using a constant rate during warm-up followed by a geometric decay, and from [Exponential Decay Learning Rate Schedule](9145-exponential-decay-learning-rate-schedule.md) by explicitly incorporating a delayed warm-up period to stabilize initial training. It solves the problem of unstable convergence in sparse or complex captures by preventing large gradient updates before the model has established a coherent geometric prior.

## How it works
The method configures the Adam optimizer with specific hyperparameters for learning rate generation, gradient clipping, and optimization delays.

**Inputs:**
- `initial_lr`: Starting learning rate.
- `final_lr`: Target learning rate at the end of training.
- `total_steps`: Total number of optimization steps.
- `warmup_steps`: Number of steps for the warm-up phase.
- `delay_steps`: Number of steps to delay exposure optimization (if applicable).
- `clip_value`: Maximum absolute value for gradient clipping.
- `clip_norm`: Maximum L2 norm for gradient normalization.

**Data Structures:**
- No specific complex data structures; standard optimizer state (Adam moments) and scalar parameters.

**Parameters (Typical Values):**
Two distinct configurations are cited in the literature:
1. **Exposure Optimization (Per-chunk):**
   - `initial_lr`: $1 \times 10^{-3}$ [S1]
   - `final_lr`: $1 \times 10^{-4}$ [S1]
   - `delay_steps`: 5000 [S1]
   - `delay_multiplier`: $1 \times 10^{-3}$ [S1] (Used to scale LR during delay)
2. **Network Optimization (FreeNeRF/SGCNeRF):**
   - `initial_lr`: $2 \times 10^{-3}$ [S2]
   - `final_lr`: $2 \times 10^{-5}$ [S2]
   - `warmup_steps`: 512 [S2]
   - `clip_value`: 0.1 [S2]
   - `clip_norm`: 0.1 [S2]

**Steps:**
1. **Initialize Optimizer:** Set up Adam with `initial_lr`.
2. **Compute Current Step:** Determine the current training step $t$.
3. **Calculate Learning Rate:**
   - If $t < \text{delay\_steps}$ (Exposure only):
     - Apply `delay_multiplier` to the base schedule or set LR to a minimal value. Note: [S1] specifies a "delay multiplier" of $1e-3$ for exposure, implying the LR is scaled down or zeroed during this phase to ensure model coherence before optimizing camera parameters.
   - If $t < \text{warmup\_steps}$ (Warm-up):
     - Linearly interpolate LR from $0$ (or a small base) to `initial_lr`. [S2] specifies a multiplier of 0.01 during the first 512 steps, which effectively scales the initial rate or acts as a warm-up factor. Specifically, [S2] states: "During the initial 512-step training, a warm-up strategy with a multiplier of 0.01 is implemented." This implies $LR(t) = \text{initial\_lr} \times \frac{t}{\text{warmup\_steps}} \times 0.01$ or similar scaling. However, standard warm-up usually ramps up. Let's look closer. "Multiplier of 0.01" might mean the LR is $1/100$th of the target during warmup, then ramps? Or it is a multiplicative factor on the base LR? Given "warm-up strategy with a multiplier", it often implies $LR = \text{base\_lr} \times \text{warmup\_factor}(t)$. If the factor is 0.01, it might be a constant low LR. But "warm-up" implies change. A common interpretation in this context (FreeNeRF) is that the LR ramps up to the initial LR, but scaled by 0.01? Or the initial LR is effectively $2e-3 \times 0.01$? [S2] says "exponentially decays from $2 \cdot 10^{-3}$ to $2 \cdot 10^{-5}$... During the initial 512-step training, a warm-up strategy with a multiplier of 0.01 is implemented." This likely means the LR is $\text{initial\_lr} \times 0.01 \times \frac{t}{512}$ during warmup.
   - If $t \ge \text{warmup\_steps}$ (Decay):
     - Apply exponential decay. The decay factor $\alpha$ is derived from $\text{initial\_lr}$, $\text{final\_lr}$, and the remaining steps.
     - Formula: $LR(t) = \text{final\_lr} + (\text{initial\_lr} - \text{final\_lr}) \cdot \gamma^{(t - \text{warmup\_steps})}$, where $\gamma$ is the per-step decay rate.
     - Alternatively, if the decay is defined over total steps $T$: $LR(t) = \text{initial\_lr} \cdot \left(\frac{\text{final\_lr}}{\text{initial\_lr}}\right)^{\frac{t - \text{warmup\_steps}}{T - \text{warmup\_steps}}}$.
4. **Gradient Clipping:**
   - Clip gradients by value: $g_{ij} = \text{clip}(g_{ij}, -\text{clip\_value}, \text{clip\_value})$ [S2].
   - Normalize gradients: $g = g / \max(1, \|g\|_2 / \text{clip\_norm})$ [S2].
5. **Update Parameters:** Perform Adam update step.
6. **Exposure Handling (Specific to [S1]):**
   - If $t < 5000$, do not optimize exposure (or use scaled LR).
   - After 5000 steps, optimize per-camera exposures.
   - For hierarchy post-optimization, reuse the trained exposure and fix them (do not optimize further) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_lr(step, total_steps, initial_lr, final_lr, warmup_steps, delay_steps=None, delay_multiplier=1.0):
    """
    Computes the learning rate for the current step.
    Supports both exposure-specific delay and standard warm-up.
    """
    if delay_steps is not None and step < delay_steps:
        # Exposure optimization is delayed or scaled down
        # [S1] uses a delay multiplier of 1e-3
        return initial_lr * delay_multiplier
    
    if step < warmup_steps:
        # Warm-up phase
        # [S2] applies a multiplier of 0.01 during warm-up
        # Linear ramp from 0 to initial_lr * 0.01? Or initial_lr * (step/warmup) * 0.01?
        # Assuming linear warm-up to a scaled initial rate:
        current_lr = initial_lr * (step / warmup_steps) * 0.01
        return current_lr
    
    # Exponential decay phase
    # Calculate decay factor gamma such that LR reaches final_lr at total_steps
    # LR(t) = final_lr + (initial_lr - final_lr) * gamma^(step - warmup_steps)
    # At step = total_steps: final_lr = final_lr + (initial_lr - final_lr) * gamma^(total_steps - warmup_steps)
    # 0 = (initial_lr - final_lr) * gamma^(remaining_steps) -> This implies gamma < 1
    # Actually, standard exponential decay: LR(t) = initial_lr * decay_rate^step
    # But we have a target final_lr.
    # Let's use the ratio: (final_lr / initial_lr) = decay_rate^(total_steps - warmup_steps)
    # decay_rate = (final_lr / initial_lr) ^ (1 / (total_steps - warmup_steps))
    
    remaining_steps = total_steps - warmup_steps
    if remaining_steps <= 0:
        return final_lr
        
    decay_rate = (final_lr / initial_lr) ** (1.0 / remaining_steps)
    
    # Current decay steps
    decay_steps = step - warmup_steps
    current_lr = initial_lr * (decay_rate ** decay_steps)
    
    return current_lr

def apply_gradients(grads, clip_value=0.1, clip_norm=0.1):
    # Clip by value
    clipped_grads = []
    for g in grads:
        clipped_g = torch.clamp(g, -clip_value, clip_value)
        clipped_grads.append(clipped_g)
    
    # Normalize by L2 norm
    total_norm = torch.norm(torch.stack([torch.norm(g) for g in clipped_grads]))
    if total_norm > clip_norm:
        clipped_grads = [g * (clip_norm / total_norm) for g in clipped_grads]
        
    return clipped_grads
```

## Operating characteristics
- **Time Complexity:** O(1) per step for LR calculation. The warm-up and decay calculations are constant time.
- **Memory Complexity:** O(1) additional memory for LR state.
- **Performance:** The delay steps (5000) prevent exposure optimization from destabilizing the initial geometric reconstruction, which is critical for sparse consumer-level captures [S1]. The warm-up prevents early divergence in sensitive parameters [S2].
- **Strengths:** Stabilizes training for sparse views and exposure parameters. Prevents "mound" artifacts in underdetermined regions by ensuring geometric coherence first [S1].
- **Limitations:** Requires careful tuning of delay steps relative to dataset size. The fixed warm-up multiplier (0.01) may be too aggressive for some scenes [S2].
- **When to use:** Use for optimizing per-camera exposures in large, sparse datasets (e.g., vehicular captures) [S1], or for general NeRF training when stability is a concern (e.g., FreeNeRF settings) [S2].

## Background
The method originates from two distinct lines of work in neural rendering optimization. The exposure-specific variant (with delay) was introduced for hierarchical 3D Gaussian representations to handle very large, sparse datasets like those from vehicle-based captures [S1]. The network optimization variant (with warm-up and gradient clipping) is adapted from FreeNeRF settings and used in SGCNeRF for few-shot neural rendering with sparse geometric consistency [S2]. Both aim to address the instability inherent in optimizing radiance fields with limited or sparse input data.

## Sources
[S1] A Hierarchical 3D Gaussian Representation for Real-Time Rendering of Very Large Datasets, https://www.youtube.com/watch?v=p-mb2TzVJZk, fetched 2026-10-31
[S2] SGCNeRF: Few-Shot Neural Rendering via Sparse Geometric Consistency Guidance, https://doi.org/10.1109/tcsvt.2025.3610950, fetched 2026-10-30
