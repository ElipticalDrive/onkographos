# Quantization-Aware Fine-Tuning

Also called: Fine-tuning with Quantization Simulation

## Overview
Quantization-Aware Fine-Tuning (QAT) is a post-clustering optimization step designed to recover reconstruction quality lost during the vector quantization of 3D scene parameters. While clustering compresses parameters into compact codebooks, it introduces discretization errors; QAT mitigates this by fine-tuning the scene representation on training images while simulating the target low-precision quantization during the forward pass. This method differs from post-training quantization, where weights are reduced after training, by simulating quantization during training to obtain numerically stable gradients and reduce final inference errors [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of 3D Gaussians with parameters (position, opacity, scaling, rotation, color) that have been clustered into codebooks. The minimum and maximum values for each parameter type are tracked.
*   **Output:** Optimized scene parameters stored at reduced bit-precision (e.g., 8-bit), along with the full-precision min/max values required for dequantization.

**Data Structures**
The scene is represented as a set of $N$ 3D Gaussians. Each Gaussian $i$ has parameters:
*   Position $\mathbf{p}_i \in \mathbb{R}^3$
*   Opacity $\alpha_i \in [0, 1]$
*   Scaling factor $\mathbf{s}_i \in \mathbb{R}^3$
*   Rotation quaternion/vector $\mathbf{q}_i$
*   Color coefficients $\mathbf{c}_i$ (encoded in codebooks)

**Parameters**
*   **Quantization Precision:** 8-bit for all parameters except position. 16-bit float for position [S1].
*   **Optimization Steps:** 5000 steps [S1].
*   **Quantization Scheme:** Min-Max quantization [S1].

**Method Steps**
1.  **Initialize:** Start with the clustered scene parameters. Maintain moving averages of the minimum and maximum values for each parameter type to define the quantization range.
2.  **Forward Pass (Simulation):** For each parameter $p$, simulate quantization using a rounding operation. The range is determined by the moving average of the parameter's minimum and maximum values.
    *   For opacity: Apply quantization after the sigmoid activation function.
    *   For scaling and rotation: Apply quantization before the respective normalization step.
    *   For scale factor: Apply quantization before the exponential activation function.
    *   For position: Use 16-bit float quantization.
    *   For other parameters: Use 8-bit quantization.
3.  **Loss Computation:** Render the scene using the quantized parameters and compute the loss against training images (e.g., MSE, SSIM).
4.  **Backward Pass (Straight-Through):** Ignore the simulated quantization in the backward pass. Calculate the gradient with respect to the full-precision parameter $p$ as if no quantization occurred. This provides numerically stable gradients [S1].
5.  **Update:** Update the scene parameters and codebook entries using the accumulated gradients.
    *   Optimize for position, opacity, scaling factor, color, and Gaussian codebook entries.
    *   Accumulate incoming gradients for codebook entries and update codebook parameters [S1].
6.  **Iterate:** Repeat steps 2–5 for the specified number of optimization steps (5000) [S1].
7.  **Finalize:** After training, store the parameters at the target low precision (e.g., 8-bit). Store the minimum and maximum values required for re-scaling at full precision (e.g., 32-bit float) [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quantization_aware_forward(p, bit_count, min_val, max_val):
    """
    Simulate quantization for parameter p.
    p: full precision parameter value
    bit_count: number of bits for quantization
    min_val, max_val: moving average min/max for the parameter type
    """
    # Calculate quantization step
    if max_val == min_val:
        return p # Avoid division by zero
    
    # Map to [0, 1] range
    normalized = (p - min_val) / (max_val - min_val)
    normalized = clamp(normalized, 0.0, 1.0)
    
    # Quantize to integer
    num_levels = 2 ** bit_count
    quantized_int = round(normalized * (num_levels - 1))
    
    # Dequantize back to float
    quantized_float = quantized_int / (num_levels - 1)
    
    # Map back to original range
    return quantized_float * (max_val - min_val) + min_val

def forward_pass(gaussians, codebooks):
    """
    Render scene with simulated quantization.
    """
    rendered_image = zeros_like(target_image)
    
    for gaussian in gaussians:
        # Simulate quantization for each parameter type
        # Position uses 16-bit, others use 8-bit
        p_quant = quantization_aware_forward(gaussian.position, 16, min_pos, max_pos)
        
        # Opacity: quantize after sigmoid
        alpha_raw = sigmoid(gaussian.opacity)
        alpha_quant = quantization_aware_forward(alpha_raw, 8, 0.0, 1.0)
        
        # Scale: quantize before exponential
        scale_raw = gaussian.scale
        scale_quant = quantization_aware_forward(scale_raw, 8, min_scale, max_scale)
        scale_final = exp(scale_quant)
        
        # Rotation: quantize before normalization
        rot_raw = gaussian.rotation
        rot_quant = quantization_aware_forward(rot_raw, 8, min_rot, max_rot)
        rot_final = normalize(rot_quant)
        
        # Color: use codebook entry (already quantized/clustered)
        color = codebooks[gaussian.color_index]
        
        # Render gaussian with quantized parameters
        rendered_image += render_gaussian(p_quant, alpha_quant, scale_final, rot_final, color)
        
    return rendered_image

def backward_pass(rendered_image, target_image):
    """
    Compute gradients ignoring quantization.
    """
    loss = mse(rendered_image, target_image)
    # Gradients are computed w.r.t. full-precision parameters
    # The quantization operation is treated as identity in backward pass
    gradients = compute_gradients(loss)
    return gradients

def update_parameters(gaussians, codebooks, gradients):
    """
    Update scene parameters and codebooks.
    """
    for gaussian in gaussians:
        # Update full-precision parameters using gradients
        gaussian.position -= learning_rate * gradients.position
        gaussian.opacity -= learning_rate * gradients.opacity
        gaussian.scale -= learning_rate * gradients.scale
        gaussian.rotation -= learning_rate * gradients.rotation
        
    # Update codebook entries based on accumulated gradients
    for codebook in codebooks:
        codebook.update(gradients.codebook)

def quantization_aware_fine_tuning(gaussians, codebooks, training_images, num_steps=5000):
    """
    Main fine-tuning loop.
    """
    # Initialize min/max tracking
    min_pos, max_pos = get_min_max(gaussians.position, 16)
    min_opacity, max_opacity = get_min_max(gaussians.opacity, 8)
    min_scale, max_scale = get_min_max(gaussians.scale, 8)
    min_rot, max_rot = get_min_max(gaussians.rotation, 8)
    
    for step in range(num_steps):
        # Forward pass with quantization simulation
        rendered = forward_pass(gaussians, codebooks)
        
        # Backward pass without quantization simulation
        grads = backward_pass(rendered, training_images[step])
        
        # Update parameters
        update_parameters(gaussians, codebooks, grads)
        
        # Update min/max moving averages
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

## Operating characteristics
*   **Time Complexity:** The fine-tuning process requires 5000 optimization steps [S1]. Each step involves rendering and gradient computation, similar to standard 3D Gaussian Splatting training.
*   **Memory Complexity:** The final storage uses reduced bit-precision (8-bit for most parameters, 16-bit for position), significantly reducing memory bandwidth requirements compared to full 32-bit precision [S1].
*   **Performance:** Enables real-time novel view synthesis on low-end devices due to reduced memory bandwidth and compatibility with hardware rasterization [S1].
*   **Strengths:** Recovers quality lost during clustering; allows for low-bit storage without significant quality loss; compatible with standard optimizers.
*   **Limitations:** Requires additional training time (5000 steps); requires tracking min/max values for dequantization.

## Background
Quantization-Aware Fine-Tuning is proposed in the context of Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis [S1]. It builds upon the observation that post-training quantization often leads to significant quality degradation, whereas simulating quantization during training allows the model to adapt to the reduced precision [S1]. The method is part of a larger pipeline that includes sensitivity-aware vector clustering and entropy encoding [S1].

## Sources
[S1] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
