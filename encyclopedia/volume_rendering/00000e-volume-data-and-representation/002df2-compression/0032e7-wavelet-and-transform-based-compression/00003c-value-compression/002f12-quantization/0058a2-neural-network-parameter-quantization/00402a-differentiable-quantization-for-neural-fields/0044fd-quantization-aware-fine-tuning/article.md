# Quantization-Aware Fine-Tuning

Also called: Fine-tuning with Quantization Simulation

## Overview
Quantization-Aware Fine-Tuning (QAT) is a post-clustering optimization step designed to recover reconstruction quality lost during the vector quantization of 3D scene parameters [S1]. While clustering compresses parameters into compact codebooks, it introduces discretization errors; QAT mitigates this by fine-tuning the scene representation on training images while simulating the target low-precision quantization during the forward pass [S1]. This method differs from post-training quantization, where weights are reduced after training, by simulating quantization during training to obtain numerically stable gradients and reduce final inference errors [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of 3D Gaussians with parameters (position, opacity, scaling, rotation, color) that have been clustered into codebooks [S1]. The minimum and maximum values for each parameter type are tracked [S1].
*   **Output:** Optimized scene parameters stored at reduced bit-precision (e.g., 8-bit), along with the full-precision min/max values required for dequantization [S1].

**Data Structures**
The scene is represented as a set of $N$ 3D Gaussians [S1]. Each Gaussian $i$ has parameters:
*   Position $\mathbf{p}_i \in \mathbb{R}^3$ [S1].
*   Opacity $\alpha_i \in [0, 1]$ [S1].
*   Scaling factor $\mathbf{s}_i \in \mathbb{R}^3$ [S1].
*   Rotation quaternion/vector $\mathbf{q}_i$ [S1].
*   Color coefficients $\mathbf{c}_i$ (encoded in codebooks) [S1].

**Parameters**
*   **Quantization Precision:** 8-bit for all parameters except position. 16-bit float for position [S1].
*   **Optimization Steps:** 5000 steps [S1].
*   **Quantization Scheme:** Min-Max quantization [S1].

**Method Steps**
1.  **Initialize:** Start with the clustered scene parameters [S1]. Maintain moving averages of the minimum and maximum values for each parameter type to define the quantization range [S1].
2.  **Forward Pass (Simulation):** For each parameter $p$, simulate quantization using a rounding operation [S1]. The range is determined by the moving average of the parameter's minimum and maximum values [S1].
    *   For opacity: Apply quantization after the sigmoid activation function [S1].
    *   For scaling and rotation: Apply quantization before the respective normalization step [S1].
    *   For scale factor: Apply quantization before the exponential activation function [S1].
    *   For position: Use 16-bit float quantization [S1].
    *   For other parameters: Use 8-bit quantization [S1].
3.  **Loss Computation:** Render the scene using the quantized parameters and compute the loss against training images (e.g., MSE, SSIM) [G].
4.  **Backward Pass (Straight-Through):** Ignore the simulated quantization in the backward pass [S1]. Calculate the gradient with respect to the full-precision parameter $p$ as if no quantization occurred [S1]. This provides numerically stable gradients [S1].
5.  **Update:** Update the scene parameters and codebook entries using the accumulated gradients [S1].
    *   Optimize for position, opacity, scaling factor, color, and Gaussian codebook entries [S1].
    *   Accumulate incoming gradients for codebook entries and update codebook parameters [S1].
6.  **Iterate:** Repeat steps 2–5 for the specified number of optimization steps (5000) [S1].
7.  **Finalize:** After training, store the parameters at the target low precision (e.g., 8-bit) [S1]. Store the minimum and maximum values required for re-scaling at full precision (e.g., 32-bit float) [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def zeros_like(arr):
    return [0.0] * len(arr)

def sigmoid(x):
    return 1.0 / (1.0 + math.exp(-x))

def update_min_max(old_min, old_max, new_val):
    return min(old_min, new_val), max(old_max, new_val)

def get_min_max(arr):
    return min(arr), max(arr)

def mse(a, b):
    return sum((x - y) ** 2 for x, y in zip(a, b)) / len(a)

def clamp(x, lo, hi):
    return max(lo, min(hi, x))

def quantization_aware_forward(p, bit_count, min_val, max_val):
    if max_val == min_val:
        return p
    normalized = clamp((p - min_val) / (max_val - min_val), 0.0, 1.0)
    num_levels = 2 ** bit_count
    quantized_int = round(normalized * (num_levels - 1))
    quantized_float = quantized_int / (num_levels - 1)
    return quantized_float * (max_val - min_val) + min_val

def render_gaussian(p, alpha, scale, rotation, color):
    # Simplified differentiable splatting projection [G]
    return color * alpha

def compute_gradients(loss, params):
    # Standard automatic differentiation engine [G]
    return params.grad

def quantization_aware_fine_tuning(gaussians, codebooks, training_images, num_steps=5000, lr=0.0025):
    min_pos, max_pos = get_min_max([g.position for g in gaussians])
    min_opacity, max_opacity = get_min_max([g.opacity for g in gaussians])
    min_scale, max_scale = get_min_max([g.scale for g in gaussians])
    min_rot, max_rot = get_min_max([g.rotation for g in gaussians])

    for step in range(num_steps):
        rendered = zeros_like(training_images[step])
        for gaussian in gaussians:
            p_quant = quantization_aware_forward(gaussian.position, 16, min_pos, max_pos)
            alpha_raw = sigmoid(gaussian.opacity)
            alpha_quant = quantization_aware_forward(alpha_raw, 8, 0.0, 1.0)
            scale_quant = quantization_aware_forward(gaussian.scale, 8, min_scale, max_scale)
            scale_final = math.exp(scale_quant)
            rot_quant = quantization_aware_forward(gaussian.rotation, 8, min_rot, max_rot)
            rot_final = rot_quant / math.sqrt(sum(x**2 for x in rot_quant))
            color = codebooks[gaussian.color_index]
            rendered = [r + render_gaussian(p_quant, alpha_quant, scale_final, rot_final, color) for r in rendered]

        loss = mse(rendered, training_images[step])
        gradients = compute_gradients(loss, gaussians)
        for gaussian in gaussians:
            gaussian.position -= lr * gradients.position
            gaussian.opacity -= lr * gradients.opacity
            gaussian.scale -= lr * gradients.scale
            gaussian.rotation -= lr * gradients.rotation
        for codebook in codebooks:
            codebook.update(gradients.codebook)

        min_pos, max_pos = update_min_max(min_pos, max_pos, gaussians.position)
        min_opacity, max_opacity = update_min_max(min_opacity, max_opacity, gaussians.opacity)
        min_scale, max_scale = update_min_max(min_scale, max_scale, gaussians.scale)
        min_rot, max_rot = update_min_max(min_rot, max_rot, gaussians.rotation)
    return gaussians, codebooks
```

**Edge Cases and Pitfalls**
*   **Division by Zero:** If `max_val` equals `min_val` during quantization, the normalization step fails. A guard is required to return the original value or handle this case [G].
*   **Precision Mismatch:** Position requires 16-bit float; using 8-bit for position significantly decreases reconstruction quality [S1].
*   **Activation Order:** Quantization must be applied at specific points relative to activations (e.g., before exponential for scale, after sigmoid for opacity) to maintain model integrity [S1].

## Subtopics

### [JointRF Training Procedure](11657-jointrf-training-procedure.md)
A differentiable optimization method for dynamic neural radiance fields (NeRFs) that jointly optimizes scene representation and compression. It sets itself apart from methods that treat compression and representation as independent steps by integrating an entropy model into the forward pass, allowing end-to-end gradient propagation through the quantization simulation. Choose it when training dynamic scenes and minimizing rate-distortion trade-offs is required. See [JointRF Training Procedure](11657-jointrf-training-procedure.md).

## Operating characteristics
*   **Time Complexity:** The fine-tuning process requires 5000 optimization steps [S1]. Each step involves rendering and gradient computation, similar to standard 3D Gaussian Splatting training [S1].
*   **Memory Complexity:** The final storage uses reduced bit-precision (8-bit for most parameters, 16-bit for position), significantly reducing memory bandwidth requirements compared to full 32-bit precision [S1].
*   **Performance:** Enables real-time novel view synthesis on low-end devices due to reduced memory bandwidth and compatibility with hardware rasterization [S1].
*   **Strengths:** Recovers quality lost during clustering; allows for low-bit storage without significant quality loss; compatible with standard optimizers [S1].
*   **Limitations:** Requires additional training time (5000 steps); requires tracking min/max values for dequantization [S1].

## Background
Quantization-Aware Fine-Tuning is proposed in the context of Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis [S1]. It builds upon the observation that post-training quantization often leads to significant quality degradation, whereas simulating quantization during training allows the model to adapt to the reduced precision [S1]. The method is part of a larger pipeline that includes sensitivity-aware vector clustering and entropy encoding [S1].

## Sources
[S1] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
