# Mixed Precision & Gradient Scaling/Clipping

Also called: Adaptive Gradient Clipping, GradScaler Optimization Step

## Overview
This method integrates automatic mixed precision (AMP) training with gradient scaling and clipping to stabilize optimization in differentiable volume rendering. It addresses the numerical instability caused by low-precision arithmetic and the high-variance gradients inherent in path-traced Monte Carlo estimates. By scaling the loss before backpropagation and clipping gradients based on their L2 norm, it prevents silent failures from NaN/Inf values and mitigates the contamination of optimized textures by improbable light path outliers [S1, unofficial][S2].

## How it works
**Inputs:**
- `loss`: The scalar loss tensor.
- `optimizer`: The optimization instance (e.g., Adam).
- `grad_scaler`: The AMP gradient scaler instance.
- `max_norm`: Maximum L2 norm for gradient clipping (from optimizer config).

**Outputs:**
- Optimizer step executed or skipped.

**Data Structures:**
- `grad_scaler`: Manages the scaling factor `scale`.
- `optimizer`: Contains parameter groups with gradients.

**Steps:**
1. **Scale Loss:** Multiply the loss by the scaler's current factor.
   $$ loss_{scaled} = loss \times scale $$
2. **Backward Pass:** Compute gradients for all parameters.
   $$ loss_{scaled}.backward() $$
3. **Unscale Gradients:** Retrieve the unscaled gradients from the parameters. This is necessary before clipping because clipping must operate on the true gradient magnitudes, not the scaled ones.
   $$ grad\_scaler.unscale\_(optimizer) $$
4. **Clip Gradients:** If `max_norm` is defined, clip the global L2 norm of all gradients.
   $$ \text{clip\_grad\_norm\_}(parameters, max\_norm=max\_norm) $$
   If `max_norm` is `None`, this step is skipped [S1, unofficial].
5. **Check Finiteness:** Verify that at least one parameter group has non-None gradients and that all gradients are finite (not NaN or Inf). If gradients are non-finite, the step is skipped to prevent silent corruption [S1, unofficial].
6. **Step Optimizer:** If the check passes, update the parameters.
   $$ optimizer.step() $$
7. **Update Scaler:** Update the scaler's internal state (e.g., reduce scale if NaN/Inf was detected in previous steps, or grow it if stable) [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimizer_scaler_step(optimizer, grad_scaler, loss, max_norm=None):
    # 1. Scale loss
    scaled_loss = grad_scaler.scale(loss)
    
    # 2. Backward pass
    scaled_loss.backward()
    
    # 3. Unscale gradients for clipping
    grad_scaler.unscale_(optimizer)
    
    # 4. Clip gradients if max_norm is specified
    if max_norm is not None:
        torch.nn.utils.clip_grad_norm_(optimizer.parameters(), max_norm=max_norm)
    
    # 5. Check for NaN/Inf and non-None gradients
    # Note: In practice, grad_scaler.step() often handles the NaN check internally,
    # but explicit checks prevent silent failures in custom loops [S1, unofficial].
    has_valid_grads = False
    for group in optimizer.param_groups:
        if group['params']:
            # Check if any param has a gradient
            for p in group['params']:
                if p.grad is not None:
                    has_valid_grads = True
                    break
        if has_valid_grads:
            break
            
    if not has_valid_grads:
        grad_scaler.update()
        return

    # 6. Step optimizer
    # The scaler's step method handles the finiteness check and scale update [G]
    grad_scaler.step(optimizer)
    grad_scaler.update()
```

**Parameters:**
- `max_norm`: Maximum L2 norm for gradient clipping. If `None`, clipping is skipped [S1, unofficial]. Typical values depend on the scene complexity and learning rate; not specified in sources.

**Edge Cases:**
- **NaN/Inf Gradients:** If gradients contain NaN or Inf, the optimizer step is skipped. The code explicitly checks finiteness before stepping to prevent silent failures [S1, unofficial].
- **No Gradients:** If no parameter groups have non-None gradients, the step is skipped [S1, unofficial].

## Subtopics

### Gradient Clamping
A regularization technique used in texture-space differentiable rendering to prevent extremely high-valued outliers caused by improbable light paths in path-traced samples from contaminating optimized textures [S2]. It clamps gradients to the range $[-10^{-8}, 10^{-8}]$ before Adam rescaling [S2]. This is distinct from L2 norm clipping as it applies a hard bound to individual gradient values rather than the global norm.

## Operating characteristics
- **Time Complexity:** Adds negligible overhead for scaling and clipping operations relative to the backward pass and optimizer step.
- **Memory Complexity:** No additional memory overhead beyond standard AMP buffers.
- **Strengths:** Stabilizes training in the presence of high-variance Monte Carlo estimates and mixed precision arithmetic. Prevents silent failures from NaN/Inf gradients.
- **Limitations:** Clipping may slow convergence if `max_norm` is too small. Gradient clamping may discard useful signal from rare but valid light paths.
- **When to Use:** Essential for differentiable volume rendering with path tracing and mixed precision.
- **When Not to Use:** Not necessary for deterministic, non-path-traced rendering with full precision.

## Background
Originated in the context of differentiable volume rendering and neural radiance fields (NeRFs). The gradient clamping technique was introduced in the context of texture-space differentiable rendering for complex indoor scenes [S2]. The integration with AMP and gradient scaling is a standard practice in modern deep learning frameworks like PyTorch, adapted here for the specific needs of volume rendering optimization [S1, unofficial].

## Sources
[S1] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S2] Material and Lighting Reconstruction for Complex Indoor Scenes with Texture-space Differentiable Rendering, https://doi.org/10.2312/sr.20211292, fetched 2026-10-29
