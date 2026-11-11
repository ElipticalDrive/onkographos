# Quantization-Aware Fine-Tuning

Also called: Fine-tuning with Quantization Simulation

## Overview
Quantization-Aware Fine-Tuning is a post-clustering optimization step designed to regain reconstruction quality lost during the vector quantization (clustering) of scene parameters. It solves the problem of information loss inherent in compressing continuous neural field or Gaussian splatting parameters into discrete codebooks by re-optimizing the scene representation under the constraint of reduced bit-rates. Unlike post-training quantization, which applies quantization only after convergence, this method simulates quantization during the optimization process to allow the model to adapt its parameters to the specific errors introduced by the target precision, thereby minimizing the final reconstruction error [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of scene parameters (e.g., 3D Gaussian positions, opacities, scales, rotations, colors) that have been clustered into compact codebooks. The minimum and maximum values for each parameter type are tracked.
*   **Output:** Optimized scene parameters stored at reduced bit-precision (e.g., 8-bit integers or 16-bit floats), along with the full-precision min/max values required for dequantization during inference.

**Data Structures:**
*   **Codebooks:** Arrays storing the clustered representative values for appearance and shape parameters.
*   **Parameter Buffers:** Full-precision buffers (typically 32-bit float) holding the current state of parameters during optimization.
*   **Min/Max Buffers:** Full-precision buffers tracking the moving average of minimum and maximum values for each parameter type to define the quantization range.

**Method Steps:**
1.  **Initialization:** Initialize the scene parameters using the values from the clustered codebooks. Initialize the min/max buffers with the initial parameter values.
2.  **Forward Pass (Simulation):** For each parameter $p$ to be quantized:
    *   Determine the current quantization range $[min, max]$ based on the moving average buffers.
    *   Simulate quantization by rounding $p$ to the target bit-depth $b$. This involves mapping $p$ to $[0, 2^b-1]$, rounding to the nearest integer, and mapping back to $[min, max]$.
    *   Use the quantized value for the rendering computation.
3.  **Loss Computation:** Compute the reconstruction loss (e.g., MSE) between the rendered image and the ground truth training image.
4.  **Backward Pass (Straight-Through Estimator):**
    *   Compute gradients with respect to the full-precision parameters $p$.
    *   Ignore the simulated quantization operation in the backward pass. The gradient flows through the quantization simulation as if it were an identity function (Straight-Through Estimator).
5.  **Parameter Update:** Update the full-precision parameters using an optimizer (e.g., Adam).
6.  **Codebook Update:** Accumulate incoming gradients for the codebook entries corresponding to the quantized parameters. Update the codebook parameters using these accumulated gradients.
7.  **Range Update:** Update the moving average of the min/max values for each parameter type based on the current full-precision parameter values.
8.  **Iteration:** Repeat steps 2-7 for the specified number of optimization steps.
9.  **Finalization:** After optimization, store the parameters at the target low precision (e.g., 8-bit) and the full-precision min/max values.

**Formulas:**
The Min-Max quantization for a parameter $p$ with bit-depth $b$ is simulated as follows:
$$ q(p) = \text{clamp}\left( \text{round}\left( \frac{p - min}{max - min} \cdot (2^b - 1) \right), 0, 2^b - 1 \right) $$
$$ p_{quantized} = min + \frac{q(p)}{2^b - 1} \cdot (max - min) $$
Where $min$ and $max$ are the moving average minimum and maximum values for the parameter.

**Parameters:**
*   **Quantization Precision:** 8-bit for all parameters except position. 16-bit float for position [S1].
*   **Optimization Steps:** 5000 steps [S1].
*   **Batch Size:** 218 for color features, 220 for Gaussian parameters [S1].
*   **Codebook Size:** 4096 (default) [S1].
*   **Learning Rate Coefficients:** $\beta_c = 6 \cdot 10^{-7}$, $\beta_g = 3 \cdot 10^{-6}$ [S1].

**Edge Cases and Pitfalls:**
*   **Position Precision:** Reducing position precision below 16-bit float considerably decreases reconstruction quality; thus, 16-bit is used for position while other parameters use 8-bit [S1].
*   **Activation Order:** Quantization must be applied at specific points relative to activation functions:
    *   Opacity: After sigmoid activation [S1].
    *   Scaling/Rotation: Before normalization [S1].
    *   Scale Factor: Before exponential activation to allow fine-grained representation of small Gaussians without losing the ability to model large ones [S1].
*   **Gradient Stability:** The straight-through estimator is used to ensure numerically stable gradients, as the rounding operation is non-differentiable [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quantization_aware_fine_tuning(scene_params, codebooks, training_images, num_steps=5000):
    # Initialize min/max buffers with full precision
    min_vals = {p: scene_params[p].clone() for p in scene_params}
    max_vals = {p: scene_params[p].clone() for p in scene_params}
    
    optimizer = Adam(scene_params.values(), lr=...)
    codebook_optimizer = Adam(codebooks.values(), lr=...)
    
    for step in range(num_steps):
        optimizer.zero_grad()
        codebook_optimizer.zero_grad()
        
        rendered_image = render(scene_params)
        loss = compute_loss(rendered_image, training_images)
        
        # Forward pass with quantization simulation
        quantized_params = {}
        for p_name, p_val in scene_params.items():
            if p_name == 'position':
                # Position uses 16-bit float, no min/max quantization simulation in this context
                quantized_params[p_name] = p_val
            else:
                # Min-Max quantization simulation
                min_v = min_vals[p_name]
                max_v = max_vals[p_name]
                # Clamp to avoid division by zero
                range_v = max(max_v - min_v, 1e-8)
                # Map to [0, 255] for 8-bit
                normalized = (p_val - min_v) / range_v
                quantized_int = torch.round(normalized * 255).clamp(0, 255)
                # Map back to [min, max]
                quantized_p = min_v + (quantized_int / 255.0) * range_v
                
                # Apply specific quantization timing rules
                if p_name == 'opacity':
                    quantized_p = apply_sigmoid(quantized_p) # Quantize after sigmoid
                elif p_name in ['scale', 'rotation']:
                    quantized_p = normalize(quantized_p) # Quantize before normalization
                elif p_name == 'scale_factor':
                    quantized_p = torch.exp(quantized_p) # Quantize before exp activation
                    
                quantized_params[p_name] = quantized_p
        
        # Re-render with quantized params for loss calculation? 
        # Note: The source says "simulate quantization... to obtain numerically stable gradients"
        # and "backward pass ignores simulated quantization". 
        # This implies the loss is computed on the quantized output, but gradients flow to full precision.
        # However, standard QAT usually computes loss on the quantized output.
        # Let's assume the render function takes the quantized params.
        rendered_image_quantized = render(quantized_params)
        loss = compute_loss(rendered_image_quantized, training_images)
        
        # Backward pass
        loss.backward()
        
        # Update full precision parameters
        optimizer.step()
        
        # Update codebooks based on accumulated gradients
        # The source mentions accumulating gradients for codebook entries
        for p_name, codebook in codebooks.items():
            # Assuming gradients are accumulated in the codebook entries
            codebook_optimizer.step()
            
        # Update min/max moving averages
        for p_name in scene_params:
            if p_name != 'position':
                min_vals[p_name] = update_moving_average(min_vals[p_name], scene_params[p_name])
                max_vals[p_name] = update_moving_average(max_vals[p_name], scene_params[p_name])
                
    return scene_params, codebooks, min_vals, max_vals
```

## Subtopics

## Operating characteristics
*   **Time Complexity:** The fine-tuning step adds significant training time, typically 5000 optimization steps [S1]. The complexity is dominated by the rendering and gradient computation per step.
*   **Memory Complexity:** During training, full-precision buffers are used, so memory usage is similar to standard training. After training, storage is reduced to 8-bit for most parameters and 16-bit for position, significantly reducing memory bandwidth requirements [S1].
*   **Performance:** The method enables real-time novel view synthesis on low-end devices due to reduced memory bandwidth and the use of hardware rasterization [S1]. It achieves high reconstruction quality (PSNR, SSIM) comparable to uncompressed methods while using much less storage [S1].
*   **Strengths:** Recovers quality lost during clustering; allows for low-bit storage without significant quality drop; compatible with standard optimizers.
*   **Limitations:** Requires additional training time; requires careful handling of quantization timing for different parameters.

## Background
Quantization-Aware Fine-Tuning was proposed in the context of Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis [S1]. It builds on the idea that post-training quantization can lead to significant quality degradation, and simulating quantization during training allows the model to adapt to the quantization errors. The method is part of a larger compression pipeline that includes sensitivity-aware vector clustering and entropy encoding [S1]. It is particularly relevant for applications requiring real-time rendering on resource-constrained devices, such as web-based viewers using WebGPU [S1].

## Sources
[S1] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
