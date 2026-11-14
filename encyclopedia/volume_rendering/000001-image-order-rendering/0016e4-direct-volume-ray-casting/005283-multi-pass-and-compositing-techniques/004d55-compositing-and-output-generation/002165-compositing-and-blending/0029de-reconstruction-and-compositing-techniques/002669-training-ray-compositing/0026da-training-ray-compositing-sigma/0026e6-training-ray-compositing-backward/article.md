# Training Ray Compositing (Backward)

Also called: Volume Rendering Composite Backward

## Overview
This method computes the gradients of the volume rendering loss with respect to the input density (`sigmas`) and color (`rgbs`) parameters during the backward pass of differentiable volume rendering [S1, unofficial]. It implements the chain rule to propagate gradients from the final rendered image and the total accumulated alpha weights back to the individual sample points along each ray [S1, unofficial]. It is the backward counterpart to [Training Ray Compositing (Sigma)](3868-training-ray-compositing-sigma.md) [S1, unofficial].

## How it works
**Inputs:**
- `grad`: Gradient of the loss with respect to the final image RGB, shape `[N, 3]` where `N` is the number of rays [S1, unofficial].
- `grad_weights_sum`: Gradient of the loss with respect to `weights_sum` (total alpha), shape `[N]` [S1, unofficial].
- `weights_sum`: The computed total alpha values from the forward pass, shape `[N]` [S1, unofficial].
- `image`: The computed final RGB values from the forward pass, shape `[N, 3]` [S1, unofficial].
- `sigmas`: Density values for all sample points, shape `[M]` [S1, unofficial].
- `rgbs`: Color values for all sample points, shape `[M, 3]` [S1, unofficial].
- `deltas`: Distances between consecutive sample points along rays, shape `[M]` [S1, unofficial].
- `ray_start_idx`: Starting index in the sample arrays for each ray, shape `[N]` [S1, unofficial].
- `num_steps`: Number of valid sample steps for each ray, shape `[N]` [S1, unofficial].

**Outputs:**
- `grad_sigmas`: Output buffer for density gradients, shape `[M]` [S1, unofficial].
- `grad_rgbs`: Output buffer for color gradients, shape `[M, 3]` [S1, unofficial].

**Data Structures:**
The method operates on flat arrays representing the ray marching data [S1, unofficial]. Each ray `i` occupies a contiguous segment in the sample arrays starting at `ray_start_idx[i]` with length `num_steps[i]` [S1, unofficial].

**Steps:**
1. **Initialization:** For each ray `i` from `0` to `N-1`:
    - Check for empty rays: If `num_steps[i] == 0` or `ray_start_idx[i] + num_steps[i] >= M`, skip this ray [S1, unofficial].
    - Initialize accumulators for the backward pass: `T = 1.0` (transmittance), `r = 0.0`, `g = 0.0`, `b = 0.0` (accumulated color components) [S1, unofficial].
    - Retrieve final forward pass values: `r_final = image[i, 0]`, `g_final = image[i, 1]`, `b_final = image[i, 2]`, `T_final = 1.0 - weights_sum[i]` [S1, unofficial].
    - Set the sample pointer to the start of the ray's data: `idx = ray_start_idx[i]` [S1, unofficial].

2. **Backward Loop:** Iterate `step` from `0` to `num_steps[i] - 1`:
    - Retrieve current sample properties: `sigma = sigmas[idx]`, `rgb = rgbs[idx]`, `delta = deltas[idx]` [S1, unofficial].
    - Compute alpha: $\alpha = 1 - \exp(-\sigma \cdot \delta)$ [G].
    - Compute weight: $w = \alpha \cdot T$ [S1, unofficial].
    - **Compute RGB Gradients:** The gradient of the loss with respect to the current color $c_i$ is $\frac{\partial L}{\partial c_i} = w \cdot \frac{\partial L}{\partial C}$, where $\frac{\partial L}{\partial C}$ is the gradient from the final image `grad[i]` [S1, unofficial]. Update `grad_rgbs[idx] = grad[i] * w` [S1, unofficial].
    - **Update Accumulators:** Update accumulated color: $r \leftarrow r + w \cdot c_r$, $g \leftarrow g + w \cdot c_g$, $b \leftarrow b + w \cdot c_b$ [S1, unofficial]. Update transmittance: $T \leftarrow T \cdot (1 - \alpha)$ [S1, unofficial].
    - **Compute Sigma Gradient:** The gradient with respect to density $\sigma_i$ is computed as $\frac{\partial L}{\partial \sigma_i} = \delta \left( \frac{\partial L}{\partial R}(T \cdot c_R - (C_R - r)) + \frac{\partial L}{\partial G}(T \cdot c_G - (C_G - g)) + \frac{\partial L}{\partial B}(T \cdot c_B - (C_B - b)) + \frac{\partial L}{\partial \alpha_{\text{total}}} T_{\text{final}} \right)$ [S1, unofficial]. Here, $r, g, b$ are the accumulated color components *before* the current step's contribution, and $C_R, C_G, C_B$ are the final image components [S1, unofficial].
    - Advance the sample pointer: `idx += 1` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def training_ray_compositing_backward(
    grad,             # [N, 3]
    grad_weights_sum, # [N]
    weights_sum,      # [N]
    image,            # [N, 3]
    sigmas,           # [M]
    rgbs,             # [M, 3]
    deltas,           # [M]
    ray_start_idx,    # [N]
    num_steps,        # [N]
    M,                # Total number of samples
    N                 # Total number of rays
):
    grad_sigmas = [0.0] * M
    grad_rgbs = [[0.0, 0.0, 0.0] for _ in range(M)]

    for i in range(N):
        if num_steps[i] == 0 or ray_start_idx[i] + num_steps[i] >= M:
            continue

        T = 1.0
        r_acc = 0.0
        g_acc = 0.0
        b_acc = 0.0

        r_final = image[i][0]
        g_final = image[i][1]
        b_final = image[i][2]
        T_final = 1.0 - weights_sum[i]

        grad_r = grad[i][0]
        grad_g = grad[i][1]
        grad_b = grad[i][2]
        grad_w_sum = grad_weights_sum[i]

        for step in range(num_steps[i]):
            idx = ray_start_idx[i] + step

            sigma = sigmas[idx]
            rgb0 = rgbs[idx][0]
            rgb1 = rgbs[idx][1]
            rgb2 = rgbs[idx][2]
            delta = deltas[idx]

            alpha = 1.0 - math.exp(-sigma * delta)
            weight = alpha * T

            grad_rgbs[idx][0] = grad_r * weight
            grad_rgbs[idx][1] = grad_g * weight
            grad_rgbs[idx][2] = grad_b * weight

            term_r = T * rgb0 - (r_final - r_acc)
            term_g = T * rgb1 - (g_final - g_acc)
            term_b = T * rgb2 - (b_final - b_acc)

            grad_sigmas[idx] = delta * (
                grad_r * term_r +
                grad_g * term_g +
                grad_b * term_b +
                grad_w_sum * T_final
            )

            r_acc += weight * rgb0
            g_acc += weight * rgb1
            b_acc += weight * rgb2

            T = T * (1.0 - alpha)

    return grad_sigmas, grad_rgbs
```

**Parameters:**
- `grad_weights_sum`: Gradient of the loss with respect to `weights_sum` (total alpha), shape `[N]` [S1, unofficial].
- `grad`: Gradient of the loss with respect to the final image RGB, shape `[N, 3]` [S1, unofficial].
- `weights_sum`: The computed total alpha values from the forward pass, shape `[N]` [S1, unofficial].
- `image`: The computed final RGB values from the forward pass, shape `[N, 3]` [S1, unofficial].
- `grad_sigmas`: Output buffer for density gradients, shape `[M]` [S1, unofficial].
- `grad_rgbs`: Output buffer for color gradients, shape `[M, 3]` [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Empty Rays:** Rays with `num_steps == 0` or exceeding the sample buffer bounds must be skipped to avoid out-of-bounds access [S1, unofficial].
- **Precision:** The computation involves exponentials and subtractions of similar magnitudes (e.g., `T_final` and `weights_sum`) [G]. Care should be taken with floating-point precision, especially for very small or large densities [G].
- **Boundary Conditions:** The transmittance `T` is updated sequentially, and the backward pass relies on forward pass values stored in `image` and `weights_sum` [G].

## Operating characteristics
- **Time Complexity:** $O(M)$, where $M$ is the total number of sample points across all rays, as each sample is processed exactly once [S1, unofficial].
- **Memory Complexity:** $O(M)$ for storing the output gradient buffers `grad_sigmas` and `grad_rgbs` [S1, unofficial].
- **Performance:** Implemented as a CUDA kernel, it is designed for high parallelism on GPUs, scaling linearly with the number of rays and samples per ray [S1, unofficial].
- **Strengths:** Efficiently computes gradients for differentiable volume rendering, enabling end-to-end training of neural radiance fields (NeRF) and similar models [S1, unofficial].
- **Limitations:** Requires the forward pass values (`image`, `weights_sum`) to be stored or recomputed, increasing peak memory usage for scenes with many rays and samples [S1, unofficial].
- **When to use:** When training neural volume rendering models that require gradients with respect to density and color parameters [S1, unofficial].
- **When not to use:** For inference-only scenarios where gradients are not needed [S1, unofficial].

## Background
This method is part of the differentiable volume rendering pipeline used in implementations of Instant Neural Graphics Primitives (instant-ngp) [S1, unofficial]. It enables the training of neural radiance fields by providing the necessary gradients for the volume rendering composite operation [S1, unofficial].

## Sources
[S1] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
