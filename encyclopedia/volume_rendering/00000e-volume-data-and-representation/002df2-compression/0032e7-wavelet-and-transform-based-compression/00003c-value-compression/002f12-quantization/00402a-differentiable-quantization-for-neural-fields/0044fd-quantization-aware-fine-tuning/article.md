# Quantization-Aware Fine-Tuning

Also called: Fine-tuning with Quantization Simulation

## Overview
Quantization-Aware Fine-Tuning (QAT) is a post-clustering optimization step designed to recover reconstruction quality lost during the vector quantization of scene parameters. While clustering compresses parameters into compact codebooks, it introduces discretization errors; QAT mitigates this by optimizing the scene parameters at reduced bit-rates using simulated quantization. It differs from post-training quantization by simulating quantization errors during the training process, allowing gradients to remain numerically stable and the model to adapt to the low-precision representation, whereas post-training quantization reduces weights only after full-precision training is complete [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians with parameters (position, opacity, scaling, rotation, color) that have been clustered into codebooks. Training images and camera poses.
*   **Output:** Optimized scene parameters stored at reduced bit-precision (e.g., 8-bit for most parameters, 16-bit for position) and updated codebook entries.

**Data Structures:**
*   **Scene Parameters:** Each Gaussian $i$ has parameters $p_i$.
*   **Codebooks:** Two codebooks store clustered appearance and shape parameters. Entries are updated based on accumulated gradients.
*   **Min-Max Statistics:** Moving averages of minimum and maximum values for each parameter type are maintained to define the quantization range $[min, max]$.

**Parameters:**
*   **Quantization Precision:** 8-bit for all parameters except position. 16-bit float for position [S1].
*   **Optimization Steps:** 5000 steps [S1].
*   **Batch Size:** $2^{18}$ for color features, $2^{20}$ for Gaussian parameters [S1].
*   **Codebook Size:** 4096 (default) [S1].

**Method Steps:**
1.  **Initialization:** Initialize scene parameters from the clustered codebooks. Initialize moving average min/max statistics for each parameter type.
2.  **Forward Pass (Simulation):** For each parameter $p$:
    *   Calculate the quantized value $\hat{p}$ using Min-Max quantization. This involves rounding $p$ based on the number of bits $b$ and the current moving average range $[min, max]$.
    *   Apply specific quantization timing for certain parameters:
        *   **Opacity:** Quantize after the sigmoid activation function.
        *   **Scaling/Rotation:** Quantize before the normalization step.
        *   **Scale Factor:** Quantize before the exponential activation function to preserve fine-grained representation for small Gaussians while allowing large ones [S1].
    *   Render the scene using the quantized parameters $\hat{p}$ to compute the loss against training images.
3.  **Backward Pass (Straight-Through Estimator):**
    *   Ignore the simulated quantization in the backward pass.
    *   Calculate gradients with respect to the full-precision parameters $p$ as if no quantization occurred. This ensures numerically stable gradients [S1].
4.  **Update:**
    *   Update the scene parameters $p$ using the computed gradients.
    *   Accumulate incoming gradients for codebook entries.
    *   Update the codebook parameters based on the accumulated gradients.
    *   Update the moving average min/max statistics for the parameters.
5.  **Iteration:** Repeat steps 2-4 for the specified number of optimization steps (e.g., 5000).
6.  **Final Storage:** After training, store the parameters at the target low precision (e.g., 8-bit). Store the final min/max values at full precision (e.g., 32-bit float) for re-scaling during inference [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quantization_aware_fine_tuning(gaussians, training_data, num_steps=5000, bit_precision=8, pos_bit_precision=16):
    """
    Fine-tune Gaussian parameters using quantization-aware training.
    
    Args:
        gaussians: List of Gaussian objects with parameters (pos, opacity, scale, rot, color).
        training_data: Iterable of (image, camera_pose) pairs.
        num_steps: Number of optimization steps.
        bit_precision: Bit precision for most parameters (default 8).
        pos_bit_precision: Bit precision for position (default 16).
        
    Returns:
        Optimized gaussians and codebooks.
    """
    
    # Initialize moving averages for min/max
    min_vals = {p: float('inf') for p in gaussian_params}
    max_vals = {p: float('-inf') for p in gaussian_params}
    
    # Initialize codebooks (assumed pre-populated by clustering)
    codebooks = initialize_codebooks(gaussians)
    
    for step in range(num_steps):
        # Sample batch
        images, cameras = sample_batch(training_data, batch_size=2**18) # Color batch size
        
        # Forward pass with quantization simulation
        quantized_gaussians = []
        for g in gaussians:
            q_g = Gaussian()
            
            # Position: 16-bit float
            q_g.pos = min_max_quantize(g.pos, min_vals['pos'], max_vals['pos'], pos_bit_precision)
            
            # Opacity: Quantize after sigmoid
            raw_opacity = g.opacity
            sigmoid_opacity = sigmoid(raw_opacity)
            q_g.opacity = min_max_quantize(sigmoid_opacity, min_vals['opacity'], max_vals['opacity'], bit_precision)
            
            # Scale Factor: Quantize before exponential activation
            raw_scale = g.scale
            q_g.scale = min_max_quantize(raw_scale, min_vals['scale'], max_vals['scale'], bit_precision)
            # Note: Exponential activation is applied during rendering, not here
            
            # Scaling/Rotation: Quantize before normalization
            raw_rot = g.rotation
            q_g.rot = min_max_quantize(raw_rot, min_vals['rot'], max_vals['rot'], bit_precision)
            # Normalization is applied during rendering
            
            # Color: Quantize
            q_g.color = min_max_quantize(g.color, min_vals['color'], max_vals['color'], bit_precision)
            
            quantized_gaussians.append(q_g)
            
        # Render scene
        rendered_images = render(quantized_gaussians, cameras)
        
        # Compute loss
        loss = compute_loss(rendered_images, images)
        
        # Backward pass (Straight-Through Estimator)
        # Gradients are computed w.r.t. full-precision parameters, ignoring quantization
        gradients = compute_gradients(loss, gaussians)
        
        # Update parameters
        for i, g in enumerate(gaussians):
            g.pos = g.pos - learning_rate * gradients[i]['pos']
            g.opacity = g.opacity - learning_rate * gradients[i]['opacity']
            g.scale = g.scale - learning_rate * gradients[i]['scale']
            g.rotation = g.rotation - learning_rate * gradients[i]['rotation']
            g.color = g.color - learning_rate * gradients[i]['color']
            
            # Update moving averages
            min_vals['pos'] = min(min_vals['pos'], g.pos)
            max_vals['pos'] = max(max_vals['pos'], g.pos)
            # ... update for other params ...
            
        # Update codebooks
        update_codebooks(codebooks, gaussians, gradients)
        
    return gaussians, codebooks

def min_max_quantize(value, min_val, max_val, bits):
    """
    Simulate Min-Max quantization.
    Returns quantized value.
    """
    if max_val == min_val:
        return value
        
    # Normalize to [0, 1]
    normalized = (value - min_val) / (max_val - min_val)
    
    # Quantize to integer range
    num_levels = 2 ** bits
    quantized_int = round(normalized * (num_levels - 1))
    
    # Dequantize back to [0, 1]
    dequantized_normalized = quantized_int / (num_levels - 1)
    
    # Scale back to original range
    dequantized_value = dequantized_normalized * (max_val - min_val) + min_val
    
    return dequantized_value
```

**Edge Cases and Pitfalls:**
*   **Precision Loss:** Reducing position precision below 16-bit significantly decreases reconstruction quality [S1].
*   **Activation Order:** Quantization must be applied at specific points relative to activations (e.g., before exponential for scale, after sigmoid for opacity) to maintain model integrity [S1].
*   **Gradient Stability:** Using straight-through estimator is crucial; otherwise, gradients through the quantization step would be zero or unstable [S1].

## Operating characteristics
*   **Time Complexity:** Adds an additional optimization phase of 5000 steps to the pipeline [S1].
*   **Memory Complexity:** Reduces storage size by storing parameters at 8-bit (or 16-bit for position) precision, with min/max values stored at 32-bit [S1].
*   **Performance:** Enables real-time novel view synthesis on low-end devices due to reduced memory bandwidth requirements and compatibility with hardware rasterization [S1].
*   **Strengths:** Recovers quality lost during clustering; allows for significant compression ratios while maintaining high PSNR/SSIM [S1].
*   **Limitations:** Requires additional training time; sensitive to the choice of bit precision for different parameters [S1].

## Background
Quantization-Aware Fine-Tuning is proposed in the context of Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis [S1]. It builds upon the idea that post-clustering parameters can be fine-tuned to regain lost information [S1]. The method is inspired by volumetric NeRF compression techniques [15, 25] and adapts quantization-aware training principles from deep learning [18] to the 3D Gaussian Splatting framework [13].

## Sources
[S1] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
