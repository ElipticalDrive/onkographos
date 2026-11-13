# Deferred Back-propagation

Also called: Deferred Gradient Computation, Patch-wise Gradient Accumulation

## Overview
Deferred back-propagation is a memory-efficient technique for optimizing radiance fields using image losses that require full-resolution inputs, such as VGG-based style losses or Neural Network Feature Matching (NNFM) [S1]. Unlike sparse pixel sampling, which is insufficient for CNN-based losses, this method enables full-resolution optimization by decoupling the loss computation from the differentiable rendering pass, maintaining a GPU memory footprint equivalent to a single small patch [S1].

## How it works
The method solves the problem where standard differentiable volume rendering is too memory-intensive for full-resolution images because pixel color is composited from many samples [S1]. It allows the use of complex image losses that fail with sparse sampling by caching gradients and re-rendering only necessary regions [S1].

**Inputs:**
- A radiance field representation (e.g., 3D Gaussian Splatting or NeRF).
- A full-resolution target image (e.g., style image or ground truth).
- A loss function $L$ that operates on full-resolution images (e.g., NNFM, Gram matrix).

**Outputs:**
- Accumulated gradients for scene parameters (positions, scales, rotations, colors).

**Data Structures:**
- `cached_grads`: A 2D array (image) of the same resolution as the rendered image, storing $\frac{\partial L}{\partial I(x,y)}$.
- `scene_params`: The set of differentiable parameters for the radiance field.
- `patch_bounds`: Coordinates defining the spatial extent of the current patch being processed.

**Steps:**
1. **Forward Render (Non-differentiable):** Render a full-resolution image $I_{full}$ from the current viewpoint using the radiance field, with auto-differentiation disabled.
2. **Loss Computation:** Compute the image loss $L(I_{full})$ and its gradient with respect to the pixel values, $\frac{\partial L}{\partial I_{full}}$. Store this result in `cached_grads`.
3. **Patch Iteration:** Iterate over the image in patches (e.g., $P \times P$ pixels). For each patch:
    a. **Local Render (Differentiable):** Re-render only the pixels within the current patch bounds using the radiance field, with auto-differentiation enabled.
    b. **Back-propagation:** Back-propagate the corresponding values from `cached_grads` for this patch through the differentiable renderer to compute gradients for the scene parameters.
    c. **Accumulation:** Accumulate these gradients into the global gradient buffers for the scene parameters.
4. **Optimization Step:** Update the scene parameters using the accumulated gradients.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def deferred_backprop(radiance_field, target_image, loss_fn, patch_size):
    # Step 1: Render full resolution without gradients
    I_full = render(radiance_field, differentiable=False)
    
    # Step 2: Compute loss and cache gradients
    loss = loss_fn(I_full, target_image)
    # Compute gradient of loss w.r.t. image pixels
    cached_grads = compute_gradient(loss, I_full)
    
    # Step 3: Iterate over patches
    num_patches_x = I_full.width // patch_size
    num_patches_y = I_full.height // patch_size
    
    for py in range(num_patches_y):
        for px in range(num_patches_x):
            # Define patch boundaries
            x_start = px * patch_size
            y_start = py * patch_size
            x_end = x_start + patch_size
            y_end = y_start + patch_size
            
            # Extract cached gradients for this patch
            patch_grads = cached_grads[y_start:y_end, x_start:x_end]
            
            # Step 3a: Re-render patch with gradients enabled
            # The renderer only computes gradients for pixels in this patch
            I_patch = render(radiance_field, differentiable=True, 
                             pixel_bounds=(x_start, y_start, x_end, y_end))
            
            # Step 3b: Back-propagate cached gradients to scene parameters
            # The chain rule is applied: dL/dParams = dL/dI_patch * dI_patch/dParams
            grads = compute_gradient(loss_fn(I_patch, target_image), I_patch, 
                                     inputs=radiance_field.parameters)
            
            # Scale gradients by the cached image-space gradients
            # Note: In practice, the gradient of the loss w.r.t. I_patch 
            # is already computed in Step 2, so we multiply or pass it directly 
            # depending on the autodiff engine's API.
            # Here we assume compute_gradient can take an upstream gradient.
            grads = grads * patch_grads
            
            # Step 3c: Accumulate gradients
            radiance_field.accumulate_gradients(grads)

    return radiance_field
```

**Edge Cases and Pitfalls:**
- **Memory Footprint:** The memory usage is dominated by the storage of `cached_grads` (full resolution) and the temporary buffer for the differentiable patch render. Ensure `cached_grads` fits in memory; if not, the patch size must be increased or the image split further [S1].
- **Gradient Consistency:** The gradients computed in the patch-wise pass must exactly match the gradients that would have been computed in a single full-resolution differentiable pass. This relies on the linearity of the gradient accumulation with respect to independent pixels [S1].
- **Boundary Artifacts:** If patches overlap or have boundaries, ensure that the gradient accumulation handles shared pixels correctly. The standard implementation uses non-overlapping patches [S1].

## Operating characteristics
- **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of pixels and $P$ is the number of patches. Since each pixel is rendered twice (once non-differentiable, once differentiable), the computational cost is approximately double that of a single non-differentiable render plus one differentiable render per patch.
- **Memory Complexity:** $O(N + P_{render})$, where $N$ is the memory for the full-resolution cached gradients and $P_{render}$ is the memory for the differentiable rendering of a single patch. This is significantly lower than $O(N)$ for a full differentiable render of the entire image.
- **Strengths:** Enables the use of sophisticated CNN-based losses (VGG, NNFM) on full-resolution images for radiance field optimization, which was previously impossible due to memory constraints [S1].
- **Limitations:** Requires two rendering passes per optimization step (one non-differentiable, one differentiable per patch). Not suitable for losses that are strictly per-pixel and independent, where sparse sampling is more efficient [S1].

## Background
Deferred back-propagation was introduced in the context of "Artistic Radiance Fields" (ARF) for 3D scene stylization [S1]. The authors noted that volumetric radiance field rendering consumes significant memory, often forcing training to regress only sparsely sampled pixels, which is insufficient for computing VGG features used in style losses [S1]. The method was developed to allow optimization on high-resolution images by accumulating cached gradients in a patch-wise fashion [S1].

## Sources
[S1] ARF: Artistic Radiance Fields, https://core.ac.uk/works/391332776, fetched 2026-10-30
