# Deferred Back-propagation

Also called: Deferred Gradient Computation, Patch-wise Gradient Accumulation

## Overview
Deferred back-propagation is a memory-efficient technique for optimizing radiance fields using image losses that require full-resolution inputs, such as VGG-based style losses or Neural Network Feature Matching (NNFM) [S1]. Unlike sparse pixel sampling, which is insufficient for CNN-based losses, this method enables full-resolution optimization by decoupling the forward render from the gradient computation, accumulating gradients in a patch-wise fashion [S1]. It allows the use of sophisticated image losses with a GPU memory footprint equivalent to rendering a single small patch [S1].

## How it works
The method solves the problem where differentiable volume rendering is too memory-intensive for full-resolution images because pixel color is composited from many samples [S1]. Standard sparse sampling fails for losses like Gram matrices that require global image context [S1]. The technique defers the back-propagation step to allow full-resolution loss computation without storing the full computational graph [S1].

**Inputs:**
- A radiance field representation (e.g., 3D Gaussians or volumetric grid).
- Camera parameters for the viewpoint to be rendered.
- A loss function $L$ defined on the full-resolution image $\mathbf{C}$ (e.g., VGG style loss).
- Patch dimensions (width $W_p$, height $H_p$).

**Outputs:**
- Accumulated gradients $\frac{\partial L}{\partial \theta}$ for all scene parameters $\theta$.

**Data Structures:**
- `cached_grads`: A 2D buffer of size $W \times H \times C$ (image width, height, channels) storing $\frac{\partial L}{\partial \mathbf{C}}$. Initialized to zeros.
- `scene_grads`: A buffer storing gradients for scene parameters $\theta$.

**Steps:**

1.  **Full-Resolution Forward Pass (No Gradients):**
    Render the full-resolution image $\mathbf{C}_{full}$ of size $W \times H$ from the current viewpoint. Auto-differentiation is disabled during this pass to save memory [S1].

2.  **Loss and Gradient Computation:**
    Compute the loss $L(\mathbf{C}_{full}, \mathbf{C}_{target})$ using the full-resolution image. Compute the gradient of the loss with respect to the pixel colors:
    $$
    \mathbf{G}_{cached} = \frac{\partial L}{\partial \mathbf{C}_{full}}
    $$
    Store $\mathbf{G}_{cached}$ in `cached_grads` [S1]. This step requires the full image but not the rendering graph.

3.  **Patch-wise Back-propagation:**
    Iterate over all non-overlapping (or overlapping, depending on implementation, but typically non-overlapping for simplicity) patches $P$ covering the image. For each patch $P$ with top-left coordinate $(u, v)$ and dimensions $W_p \times H_p$:
    
    a. **Extract Patch Gradient:**
       Extract the corresponding sub-region from `cached_grads`:
       $$
       \mathbf{g}_P = \mathbf{G}_{cached}[v:v+H_p, u:u+W_p]
       $$
       
    b. **Differentiable Patch Render:**
       Render only the pixels within patch $P$ using auto-differentiation enabled. This produces the rendered patch colors $\mathbf{C}_P$ and the computational graph for these pixels [S1].
       
    c. **Back-propagate to Scene Parameters:**
       Compute the gradients of the loss with respect to the scene parameters $\theta$ using the chain rule. Since the loss is defined on the full image, we use the cached gradients $\mathbf{g}_P$ as the upstream gradient:
       $$
       \frac{\partial L}{\partial \theta} \leftarrow \frac{\partial L}{\partial \mathbf{C}_P} \cdot \frac{\partial \mathbf{C}_P}{\partial \theta}
       $$
       where $\frac{\partial L}{\partial \mathbf{C}_P}$ is effectively $\mathbf{g}_P$ reshaped/aligned to the batch/pixel structure. Accumulate these gradients into `scene_grads` [S1].

4.  **Update:**
    Use the accumulated `scene_grads` to update the scene parameters $\theta$ via an optimizer (e.g., Adam) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def deferred_backprop(radiance_field, camera, loss_fn, patch_size, image_size):
    """
    radiance_field: The scene representation (e.g., 3D Gaussians)
    camera: Camera parameters for the current view
    loss_fn: Function computing loss and gradient w.r.t. image pixels
    patch_size: Tuple (h, w) for patch dimensions
    image_size: Tuple (H, W) for full image dimensions
    """
    H, W = image_size
    ph, pw = patch_size
    
    # Step 1: Full-res render without gradients
    with torch.no_grad():
        full_image = render(radiance_field, camera) # Shape: (H, W, C)
        
    # Step 2: Compute loss and cached gradients
    # loss_fn returns (loss_value, grad_wrt_image)
    loss, cached_grads = loss_fn(full_image)
    
    # Initialize scene gradients
    scene_grads = radiance_field.zero_grad()
    
    # Step 3: Patch-wise back-propagation
    for y in range(0, H, ph):
        for x in range(0, W, pw):
            # Determine patch boundaries
            y_end = min(y + ph, H)
            x_end = min(x + pw, W)
            
            # Extract cached gradient for this patch
            patch_grad = cached_grads[y:y_end, x:x_end]
            
            # Render patch with gradients enabled
            # Note: render_patch extracts only the relevant rays/pixels
            patch_image = render_patch(radiance_field, camera, y, x, y_end, x_end)
            
            # Back-propagate using the cached gradient
            # The loss is conceptually sum(patch_loss), so gradient flows through patch_image
            patch_grad.backward(patch_image, grad_tensors=patch_grad)
            
            # Accumulate gradients into scene parameters
            # Implementation detail: radiance_field.grad accumulates automatically if requires_grad=True
            pass
            
    return scene_grads
```

**Edge Cases and Pitfalls:**
- **Memory:** While significantly reduced, rendering many patches sequentially still requires memory for the radiance field and intermediate gradients per patch. Ensure patch size fits in GPU memory [S1].
- **Gradient Consistency:** The method assumes the loss is differentiable with respect to pixel colors. CNN-based losses like VGG are differentiable, but discrete operations must be avoided or approximated [S1].
- **Boundary Effects:** If patches overlap, gradients must be divided by the number of overlaps to avoid double-counting. The standard description implies non-overlapping patches [S1].

## Operating characteristics
- **Memory Complexity:** The memory footprint is dominated by the full-resolution image buffer for cached gradients ($O(W \cdot H \cdot C)$) and the memory required to render a single patch ($O(W_p \cdot H_p)$). This is significantly lower than storing the full differentiable graph for the entire image ($O(W \cdot H \cdot C \cdot N_{samples})$) [S1].
- **Time Complexity:** The method requires two passes over the scene: one forward pass for the full image (no gradients) and one pass per patch (with gradients). If there are $N_p$ patches, the rendering cost is $O(N_p \cdot W_p \cdot H_p) \approx O(W \cdot H)$, similar to a single full-resolution render, but with higher constant overhead due to kernel launches and patch extraction [S1].
- **Strengths:** Enables the use of full-resolution CNN-based losses (VGG, Gram matrix) which are impossible with sparse sampling [S1]. Produces higher quality stylization and content preservation [S1].
- **Limitations:** Slower than sparse sampling methods. Not suitable for losses that do not require full-image context or where memory is extremely constrained [S1].

## Background
Deferred back-propagation was introduced in the context of "Artistic Radiance Fields" (ARF) for 3D scene stylization [S1]. The authors noted that volumetric radiance field rendering consumes significant memory, often forcing training to regress only sparsely sampled pixels, which is insufficient for computing VGG features used in style losses [S1]. The method was developed to allow optimization on high-resolution images, enabling the transfer of detailed artistic features from 2D style exemplars to 3D scenes [S1].

## Sources
[S1] ARF: Artistic Radiance Fields, https://core.ac.uk/works/391332776, fetched 2026-10-30
