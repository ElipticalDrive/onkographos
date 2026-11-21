# Deferred Back-propagation

Also called: Deferred Back-propagation for Generalizable Gaussians, Deferred Gradient Computation, Patch-wise Gradient Accumulation

## Overview
Deferred Back-propagation is a memory-efficient training technique that enables the optimization of radiance fields and Gaussian splats using full-resolution image losses (such as CNN-based style losses or global attention mechanisms) despite limited GPU memory [S1, S2]. It solves the problem where standard differentiable rendering is too memory-intensive for full-resolution images, forcing a trade-off between resolution and the ability to compute complex, non-pixel-wise losses [S1, S2]. Unlike sparse pixel sampling, which fails for global losses, or naive cropping, which breaks global consistency, DBP separates the forward pass from the backward pass, allowing full-resolution rendering without gradients followed by patch-wise re-rendering with gradients [S1, S2].

## How it works
The method decouples the high-memory forward rendering pass from the backward gradient computation pass. It operates in two distinct stages per optimization step: a forward-only pass to generate the full-resolution image, and a patch-wise re-rendering pass to compute gradients [S1, S2].

### Inputs and Outputs
*   **Inputs:** Scene parameters (e.g., Gaussian attributes: position, covariance, color, opacity; or NeRF weights), camera poses, and a loss function $L$ that depends on the full-resolution rendered image $\mathbf{I}$ [S1, S2].
*   **Outputs:** Updated scene parameters after gradient accumulation [S1, S2].

### Data Structures
*   **Rendered Image Buffer ($\mathbf{I}$):** A full-resolution image tensor (e.g., $H \times W \times 3$) storing the color values from the forward pass [S1, S2].
*   **Cached Gradient Buffer ($\frac{\partial L}{\partial \mathbf{I}}$):** A tensor of the same shape as $\mathbf{I}$, storing the gradient of the loss with respect to the pixel colors [S1, S2].
*   **Patch Grid:** The image is conceptually or explicitly divided into $M \times M$ patches [S2].

### Steps
1.  **Forward Pass (No Gradients):** Render the full-resolution image $\mathbf{I}$ from the current scene parameters and camera poses. Auto-differentiation is disabled during this pass to minimize memory usage [S1, S2].
2.  **Loss Computation:** Compute the loss $L(\mathbf{I})$ using the full-resolution image $\mathbf{I}$. Calculate the gradient of the loss with respect to the image pixels, $\frac{\partial L}{\partial \mathbf{I}}$, and cache this gradient tensor [S1, S2].
3.  **Patch-wise Re-rendering:** Iterate over each patch $p$ in the image grid:
    a. Define the spatial bounds of patch $p$.
    b. Re-render the pixels within patch $p$ using the same scene parameters and camera poses, but with auto-differentiation **enabled** [S1, S2].
    c. Compute the gradient of the loss with respect to the scene parameters $\theta$ for this patch using the chain rule:
       $$
       \frac{\partial L}{\partial \theta} \bigg|_p = \sum_{\mathbf{x} \in \text{patch } p} \frac{\partial L}{\partial \mathbf{I}(\mathbf{x})} \frac{\partial \mathbf{I}(\mathbf{x})}{\partial \theta}
       $$
       where $\frac{\partial L}{\partial \mathbf{I}(\mathbf{x})}$ is retrieved from the cached gradient buffer [S1].
    d. Accumulate $\frac{\partial L}{\partial \theta} \big|_p$ into the global gradient buffer for $\theta$ [S1, S2].
4.  **Parameter Update:** Apply the accumulated gradients to update the scene parameters [S1, S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def deferred_backprop(scene_params, camera_poses, loss_fn, patch_size):
    """
    scene_params: Current scene parameters (Gaussians/NeRF weights)
    camera_poses: Camera poses for the current view
    loss_fn: Function that takes an image and returns (loss, grad_wrt_image)
    patch_size: Tuple (h, w) defining the size of each patch
    """
    # Stage 1: Forward pass without gradients
    with torch.no_grad():
        # Render full-resolution image
        full_image = render(scene_params, camera_poses, enable_grad=False)
        
        # Compute loss and cache gradients wrt image pixels
        loss, grad_image = loss_fn(full_image)
        
    # Stage 2: Patch-wise re-rendering with gradients
    # Initialize gradient accumulator for scene parameters
    grad_params = torch.zeros_like(scene_params)
    
    # Iterate over all patches
    for y in range(0, full_image.height, patch_size[0]):
        for x in range(0, full_image.width, patch_size[1]):
            # Define patch boundaries
            y_end = min(y + patch_size[0], full_image.height)
            x_end = min(x + patch_size[1], full_image.width)
            
            # Extract cached gradients for this patch
            patch_grad = grad_image[y:y_end, x:x_end, :]
            
            # Re-render this specific patch with gradients enabled
            # Note: The renderer must support rendering a subset of pixels
            # or rendering the full image and masking, but only computing
            # gradients for the patch region.
            with torch.enable_grad():
                # Re-render the patch region
                # To ensure consistency, the same scene_params are used.
                # The renderer computes dI/dtheta for the pixels in the patch.
                patch_image = render_patch(scene_params, camera_poses, 
                                           y_start=y, y_end=y_end, 
                                           x_start=x, x_end=x_end, 
                                           enable_grad=True)
                
                # Compute local gradient contribution
                # dL/dtheta = sum(dL/dI * dI/dtheta) for pixels in patch
                # This is typically handled automatically by the autograd engine
                # if we compute the loss contribution for the patch.
                
                # However, since we already have dL/dI, we can use it directly
                # if the renderer allows passing external gradients.
                # Standard autograd:
                # loss_patch = (patch_image - target).pow(2) # Example
                # But here we use the cached gradient.
                
                # Implementation detail: 
                # Most frameworks compute dL/dtheta = dL/dI * dI/dtheta.
                # We can simulate this by creating a dummy loss that uses 
                # the cached gradient as the "target" gradient signal.
                
                # Alternative: Use torch.autograd.grad
                # grad_params += torch.autograd.grad(
                #     outputs=patch_image, 
                #     inputs=scene_params, 
                #     grad_outputs=patch_grad, 
                #     retain_graph=True
                # )[0]
                
                # Since we need to accumulate, we do:
                local_grad = torch.autograd.grad(
                    outputs=patch_image,
                    inputs=scene_params,
                    grad_outputs=patch_grad,
                    retain_graph=True,
                    create_graph=False
                )[0]
                
                grad_params += local_grad

    # Update parameters
    scene_params -= learning_rate * grad_params
    return scene_params
```

### Parameters
*   **Patch Size:** Determines the trade-off between memory usage and overhead. Smaller patches use less memory but increase Python/CUDA kernel launch overhead. Typical values are not explicitly given, but the method implies patches small enough to fit in GPU memory with gradients [S1, S2].
*   **Number of Patches ($M \times M$):** In [S2], the image is split into $M \times M$ patches for consistency with global self-attention mechanisms.

### Edge Cases and Pitfalls
*   **Memory Overflow:** If the patch size is too large, the re-rendering pass with gradients enabled will still cause Out-Of-Memory (OOM) errors [S2].
*   **Consistency Errors:** If the re-rendering pass does not exactly match the forward pass (e.g., due to non-deterministic operations or different random seeds), the cached gradients will be incorrect. The method requires exact reproducibility of the forward pass [S1, S2].
*   **Global Context Loss:** In methods using self-attention (like [S2]), naive patch rendering breaks global attention. [S2] addresses this by using Local Self-Attention with cropped positional encodings to maintain consistency between full-image and patch-wise training [S2].
*   **Inefficiency for Per-Pixel Losses:** For simple per-pixel losses (L1/L2), sparse sampling is more efficient. DBP is specifically designed for CNN-based or global losses [S1].

## Subtopics

### [Local Self-Attention for DBP](10040-sns-backward-pass-with-intermediate-variable-q.md)
A technique used in to maintain consistency between full-image and patch-wise rendering by adapting self-attention to patches with cropped positional encodings.

### [Epipolar Sampler for Patches](9733-tof-gradient-computation-for-gaussian-splatting.md)
In, epipolar lines are induced by patches to guide feature aggregation during the re-rendering phase.

## Operating characteristics
*   **Memory Complexity:** The memory footprint is dominated by the largest patch re-rendering pass, not the full image. This allows training at resolutions that would otherwise cause OOM errors [S1, S2].
*   **Time Complexity:** The method requires two rendering passes per iteration: one forward pass (no gradients) and one backward pass (patch-wise with gradients). This doubles the rendering time compared to a single forward pass, but is significantly faster than full-resolution gradient computation [S1, S2].
*   **Performance:** On a single RTX 3090 GPU, standard back-propagation limits training to 2 reference views at 384×496 resolution. DBP enables training with 5-7 reference views at the same resolution, improving PSNR by 1.37–3.02 dB [S2].
*   **Strengths:** Enables full-resolution training for memory-intensive losses (VGG, Gram matrix, global attention) [S1, S2]. Avoids the performance degradation associated with resolution reduction [S2].
*   **Limitations:** Increased training time due to double rendering. Requires careful implementation to ensure consistency between forward and backward passes [S1, S2]. Not efficient for simple per-pixel losses where sparse sampling suffices [S1].

## Background
Deferred Back-propagation was introduced in the context of 3D stylization of radiance fields, specifically in the "ARF: Artistic Radiance Fields" paper [S1]. It was developed to overcome the memory limitations of volumetric rendering when applying CNN-based style losses (like VGG features) that require full-resolution images. The technique was later adapted for generalizable 3D Gaussian Splatting (GGRt) to enable high-resolution training and inference under limited GPU memory [S2].

## Sources
[S1] ARF: Artistic Radiance Fields, https://core.ac.uk/works/391332776, fetched 2026-10-30
[S2] GGRt: Towards Pose-Free Generalizable 3D Gaussian Splatting in Real-Time, https://doi.org/10.1007/978-3-031-73209-6_19, fetched 2026-10-29
