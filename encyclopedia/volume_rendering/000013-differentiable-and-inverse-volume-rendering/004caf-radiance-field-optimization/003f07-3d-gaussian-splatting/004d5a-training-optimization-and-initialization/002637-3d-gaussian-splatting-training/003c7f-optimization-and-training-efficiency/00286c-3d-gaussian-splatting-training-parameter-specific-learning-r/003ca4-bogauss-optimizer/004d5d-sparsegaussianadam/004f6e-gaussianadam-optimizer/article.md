# GaussianAdam Optimizer

Also called: GaussianAdam

## Overview
GaussianAdam is a custom variant of the Adam optimizer designed specifically for 3D Gaussian Splatting that tracks per-Gaussian visibility via a last-seen step counter and maintains a learning rate history buffer for adaptive scheduling [S1, unofficial]. It differs from standard Adam by explicitly managing per-Gaussian visibility and a dedicated learning rate history buffer, enabling step-dependent learning rate scheduling per Gaussian rather than a global schedule [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Parameter tensor `param`, gradient tensor `param.grad`, hyperparameters `lr` (base learning rate), `eps` (numerical stability epsilon), `feature_width` (number of contiguous scalar values per Gaussian), `max_steps` (buffer size), `current_step` (optimization iteration).
- **Output:** Updated `param` tensor and updated state dictionary.

**Data Structures:**
The state dictionary contains the following fields with exact layouts:
- `step`: `int64` scalar, tracking the current optimization step.
- `exp_avg`: `float32` tensor, shape matches `param`, storing the first moment estimate.
- `exp_avg_sq`: `float32` tensor, shape matches `param`, storing the second moment estimate.
- `last_seen_step`: `int32` tensor, shape `(N,)`, where `N` is the number of Gaussians, storing the last step each Gaussian was visible.
- `lr_history`: `float32` tensor, shape `(max_steps,)`, storing the learning rate history for adaptive scheduling.

**Parameters:**
- `max_steps`: `int`, maximum optimization steps for `lr_history` buffer size [S1, unofficial].
- `lr`: `float`, base learning rate [S1, unofficial].
- `eps`: `float`, epsilon for numerical stability in denominator [S1, unofficial].
- `feature_width`: `int`, number of contiguous scalar values per Gaussian in the parameter group [S1, unofficial].

**Numbered Steps:**
1. Initialize state with `step` (int64), `exp_avg` (float32, same shape as param), `exp_avg_sq` (float32, same shape as param), `last_seen_step` (int32, size N), and `lr_history` (float32, size max_steps) [S1, unofficial].
2. Compute `current_step` [S1, unofficial].
3. Ensure `lr_history` has sufficient capacity for `current_step` [S1, unofficial].
4. Record `lr` into `lr_history` at index `current_step` [S1, unofficial].
5. Compute `visible` mask based on Gaussian visibility (e.g., radii or rendering contribution) [S1, unofficial].
6. Resolve `feature_width` [S1, unofficial].
7. Call the C++/CUDA kernel `_C.adamUpdateTorch` with the following arguments:
   - `param`: Parameter tensor.
   - `param.grad`: Gradient tensor.
   - `state["exp_avg"]`: First moment buffer.
   - `state["exp_avg_sq"]`: Second moment buffer.
   - `state["last_seen_step"]`: Last seen step buffer.
   - `lr_history`: Learning rate history buffer.
   - `visible`: Visibility mask.
   - `0.9`: Beta1 hyperparameter.
   - `0.999`: Beta2 hyperparameter.
   - `eps`: Epsilon for numerical stability.
   - `current_step`: Current optimization step.
   - `N`: Number of Gaussians.
   - `feature_width`: Feature width per Gaussian.
8. Update `state['step'].fill_(current_step)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_adam_update(param, param_grad, state, lr, eps, feature_width, max_steps, current_step, N, visible):
    # Guard: Skip update if gradient is None
    if param_grad is None:
        return

    # Guard: feature_width must evenly divide param.numel()
    if param.numel() % feature_width != 0:
        raise ValueError("feature_width must evenly divide param.numel()")

    # Guard: N cannot be negative
    if N < 0:
        raise ValueError("N cannot be negative")

    # Initialize state buffers if not present
    if "exp_avg" not in state:
        state["exp_avg"] = torch.zeros_like(param, dtype=torch.float32)
        state["exp_avg_sq"] = torch.zeros_like(param, dtype=torch.float32)
        state["last_seen_step"] = torch.zeros(N, dtype=torch.int32)
        state["lr_history"] = torch.zeros(max_steps, dtype=torch.float32)
        state["step"] = torch.tensor(0, dtype=torch.int64)

    # Ensure lr_history capacity
    if current_step >= state["lr_history"].numel():
        # Extend buffer if necessary, or clamp current_step
        # Note: Implementation detail for buffer management not fully specified in sources [G]
        pass

    # Record lr in history
    state["lr_history"][current_step] = lr

    # Call the optimized C++/CUDA kernel for the actual update
    # This kernel handles the Adam update logic, visibility masking, and per-Gaussian LR scheduling
    _C.adamUpdateTorch(
        param,
        param_grad,
        state["exp_avg"],
        state["exp_avg_sq"],
        state["last_seen_step"],
        state["lr_history"],
        visible,
        0.9,
        0.999,
        eps,
        current_step,
        N,
        feature_width,
    )

    # Update step counter
    state["step"].fill_(current_step)
```

**Edge Cases and Pitfalls:**
- If `param.grad` is `None`, the update is skipped entirely [S1, unofficial].
- `feature_width` must evenly divide `param.numel()`, otherwise a `ValueError` is raised [S1, unofficial].
- `N` (number of Gaussians) cannot be negative [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] XiaoBin2001/Improved-GS, https://github.com/XiaoBin2001/Improved-GS, fetched 2026-10-27 (unofficial)
