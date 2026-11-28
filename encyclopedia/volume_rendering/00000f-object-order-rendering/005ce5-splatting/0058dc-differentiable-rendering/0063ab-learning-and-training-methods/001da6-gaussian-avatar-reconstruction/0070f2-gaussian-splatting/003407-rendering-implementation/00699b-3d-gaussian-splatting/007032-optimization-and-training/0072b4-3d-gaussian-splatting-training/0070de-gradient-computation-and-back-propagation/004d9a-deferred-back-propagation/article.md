# Deferred Back-propagation

Also called: Deferred Back-propagation for Generalizable Gaussians, Deferred Gradient Computation, Patch-wise Gradient Accumulation

## Overview
Deferred Back-propagation is a memory-efficient training technique that enables the optimization of radiance fields and Gaussian splats using full-resolution image losses (such as CNN-based style losses or global attention mechanisms) despite limited GPU memory [S1, S2]. It solves the problem where standard differentiable rendering is too memory-intensive for full-resolution images, forcing a trade-off between resolution and the ability to compute complex, non-pixel-wise losses [S1, S2]. Unlike sparse pixel sampling, which fails for global losses, or naive cropping, which breaks global consistency, DBP separates the forward pass from the backward pass, allowing full-resolution rendering without gradients followed by patch-wise re-rendering with gradients [S1, S2].

## How it works
The method decouples the high-memory forward rendering pass from the backward gradient computation pass [S1, S2]. It operates in two distinct stages per optimization step: a forward-only pass to generate the full-resolution image, and a patch-wise re-rendering pass to compute gradients [S1, S2].

### Inputs and Outputs
*   **Inputs:** Scene parameters (e.g., Gaussian attributes: position, covariance, color, opacity; or NeRF weights), camera poses, and a loss function $L$ that depends on the full-resolution rendered image $\mathbf{I}$ [S1, S2].
*   **Outputs:** Updated scene parameters after gradient accumulation [S1, S2].

### Data Structures
*   **Rendered Image Buffer ($\mathbf{I}$):** A full-resolution image tensor (e.g., $H \times W \times 3$) storing the color values from the forward pass [S1, S2].
*   **Cached Gradient Buffer ($\frac{\partial L}{\partial \mathbf{I}}$):** A tensor of the same shape as $\mathbf{I}$, storing the gradient of the loss with respect to the pixel colors [S1, S2].
*   **Patch Grid:** The image is conceptually or explicitly divided into $M \times M$ patches [S2].

### Steps
1.  **Forward Pass (No Gradients):** Render the full-resolution image $\mathbf{I}$ from the current scene parameters and camera poses. Auto-differentiation is disabled during this pass to minimize memory usage [S2].
2.  **Loss Computation:** Compute the loss $L(\mathbf{I})$ using the full-resolution image $\mathbf{I}$. Calculate the gradient of the loss with respect to the image pixels, $\frac{\partial L}{\partial \mathbf{I}}$, and cache this gradient tensor [S1].
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
def deferred_backprop(scene_params, camera_poses, loss_fn, patch_h, patch_w, img_h, img_w, lr):
    """
    scene_params: Current scene parameters (Gaussians/NeRF weights)
    camera_poses: Camera poses for the current view
    loss_fn: Function that takes an image and returns (loss, grad_wrt_image)
    patch_h, patch_w: Height and width of each patch
    img_h, img_w: Full image height and width
    lr: Learning rate scalar
    """
    # Stage 1: Forward pass without gradients
    full_image = render(scene_params, camera_poses, enable_grad=False)
    loss, grad_image = loss_fn(full_image)
    
    # Stage 2: Patch-wise re-rendering with gradients
    grad_params = zeros_like(scene_params)
    
    for y in range(0, img_h, patch_h):
        for x in range(0, img_w, patch_w):
            y_end = min(y + patch_h, img_h)
            x_end = min(x + patch_w, img_w)
            
            patch_grad = grad_image[y:y_end, x:x_end, :]
            
            with torch.enable_grad():
                # render is defined in the parent article [18365-3d-gaussian-splatting-backward-pass.md]
                patch_image = render(scene_params, camera_poses, enable_grad=True, crop=(y, y_end, x, x_end))
                local_grad = compute_gradients(patch_image, scene_params, grad_outputs=patch_grad)
                grad_params += local_grad
                
    scene_params -= lr * grad_params
    return scene_params
```

### Parameters
*   **Patch Size:** Determines the trade-off between memory usage and overhead, with patches sized to fit within GPU memory limits during the differentiable re-rendering pass [S1].
*   **Number of Patches ($M \times M$):** In [S2], the image is split into $M \times M$ patches to maintain consistency with global self-attention mechanisms.

### Edge Cases and Pitfalls
*   **Global Context Loss:** In methods using self-attention (like [S2]), naive patch rendering breaks global attention. [S2] addresses this by using Local Self-Attention with cropped positional encodings to maintain consistency between full-image and patch-wise training [S2].
*   **Inefficiency for Per-Pixel Losses:** For simple per-pixel losses (L1/L2), sparse sampling is more efficient. DBP is specifically designed for CNN-based or global losses [S1].

## Operating characteristics
*   **Memory Complexity:** The memory footprint is dominated by the largest patch re-rendering pass, not the full image [S1, S2].
*   **Time Complexity:** The method requires two rendering passes per iteration: one forward pass (no gradients) and one backward pass (patch-wise with gradients) [S1, S2].
*   **Performance:** On a single RTX 3090 GPU, standard back-propagation limits training to 2 reference views at 384×496 resolution [S2]. DBP enables training with 5-7 reference views at the same resolution, improving PSNR by 1.37–3.02 dB [S2].
*   **Strengths:** Enables full-resolution training for memory-intensive losses (VGG, Gram matrix, global attention) [S1, S2]. Avoids the performance degradation associated with resolution reduction [S2].
*   **Limitations:** Increased training time due to double rendering. Requires careful implementation to ensure consistency between forward and backward passes [S1, S2]. Not efficient for simple per-pixel losses where sparse sampling suffices [S1].

## Background
Deferred Back-propagation was introduced in the context of 3D stylization of radiance fields, specifically in the "ARF: Artistic Radiance Fields" paper [S1]. It was developed to overcome the memory limitations of volumetric rendering when applying CNN-based style losses (like VGG features) that require full-resolution images [S1]. The technique was later adapted for generalizable 3D Gaussian Splatting (GGRt) to enable high-resolution training and inference under limited GPU memory [S2].

## Sources
[S1] ARF: Artistic Radiance Fields, https://core.ac.uk/works/391332776, fetched 2026-10-30
[S2] GGRt: Towards Pose-Free Generalizable 3D Gaussian Splatting in Real-Time, https://doi.org/10.1007/978-3-031-73209-6_19, fetched 2026-10-29
