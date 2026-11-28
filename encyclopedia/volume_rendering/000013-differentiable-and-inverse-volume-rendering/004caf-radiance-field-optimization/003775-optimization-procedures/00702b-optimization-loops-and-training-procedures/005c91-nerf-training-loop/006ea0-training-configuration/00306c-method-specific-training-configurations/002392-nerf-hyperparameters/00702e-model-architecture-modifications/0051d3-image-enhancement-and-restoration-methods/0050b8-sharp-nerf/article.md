# Sharp-NeRF

Also called: Grid-based Fast Deblurring Neural Radiance Fields using Sharpness Prior

## Overview
Sharp-NeRF is a grid-based NeRF framework designed to render clean, sharp images from input blurry images by utilizing discrete learnable kernels and a sharpness prior [S1]. It addresses the high computational cost and long training times associated with MLP-based deblurring methods (such as Deblur-NeRF and DP-NeRF) by replacing neural kernel generators with a decomposed-grid representation and a fixed set of discrete blur kernels [S1]. Unlike sibling methods that rely on MLPs to predict blur parameters or require all-in-focus training data, Sharp-NeRF quantizes spatial sharpness levels to index a small set of learned kernels, enabling training completion in approximately half an hour [S1].

## How it works
**Inputs and Outputs:**
*   **Training Inputs:** Blurry images of the scene; precomputed per-pixel sharpness level maps $L$ for each training view.
*   **Training Outputs:** Blurred image patches $I_b$ that match the input blurry images.
*   **Inference Output:** Sharp novel view images $I_c$.

**Data Structures:**
1.  **Grid-based Neural Field (Backbone):** A decomposed-grid representation storing density $\sigma$ and color $\mathbf{c}$ [S1]. This replaces the monolithic MLP used in prior works.
2.  **Discrete Blur Kernel Set ($B$):** A set of $N_k$ learnable 2D convolution kernels. The size of each kernel is determined by the patch size $K \times K$ [S1].
3.  **Sharpness Level Map ($L$):** A quantized map where each pixel value corresponds to an index in $B$. Values are integers in the range $[0, N_k - 1]$ [S1].

**Parameters:**
*   $N_k$: The number of discrete blur kernels used to quantize sharpness levels. [S1]
*   $K$: The spatial dimension of the ray patches (patch size is $K \times K$) [S1].
*   Typical values: $N_k$ is small (e.g., 5–10, exact value not specified in sources) [S1]. $K$ is small (e.g., 5 or 6, matching the "5 or 6 neighboring pixels" mentioned in pitfalls) [S1].

**Steps:**

1.  **Preprocessing (Sharpness Quantization):**
    For each training view, compute a defocus map (sharpness level) for every pixel using a sharpness measure operator. Quantize this continuous map into $N_k$ discrete values to create the sharpness level map $L$. This step is performed once before training and is not part of the forward pass [S1].

2.  **Patch Sampling:**
    During training, sample ray patches with stride 1. Each patch has a spatial size of $K \times K$ [S1].

3.  **Backbone Rendering (Sharp Image):**
    Feed the ray patch inputs into the grid-based backbone neural fields. Render a sharp and clean image patch $I_c'$ from these inputs [S1].

4.  **Kernel Indexing:**
    Use the preprocessed sharpness level map $L$ corresponding to the input ray patch to index into the set of learned discrete blur kernels $B$. For each pixel in the patch, retrieve the specific kernel $b_{idx}$ where $idx = L(x,y)$ [S1]. This yields per-pixel weights $w_x$ (the kernel values) [S1].

5.  **Blurring Synthesis:**
    Perform a weighted sum (convolution) of the rendered sharp patch $I_c'$ and the indexed discrete kernels to produce the final blurred image patch $I_b$. Mathematically, for a pixel $p$ in the patch:
    $$ I_b(p) = (I_c' * b_{L(p)})(p) $$
    where $*$ denotes convolution with the discrete kernel $b_{L(p)}$ selected by the sharpness level at $p$ [S1].

6.  **Loss Calculation:**
    Compare the synthesized blurred patch $I_b$ with the ground-truth blurry input patch using a standard reconstruction loss (e.g., L1 or L2) [S1].

7.  **Inference:**
    At inference time, the blur kernel module is bypassed. The grid-based backbone renders the sharp image $I_c$ directly, which is used as the final output [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Structures
# grid_model: Decomposed-grid neural field (density and color)
# B: List of N_k learnable 2D convolution kernels, each of size K x K
# L: Precomputed sharpness level map for the current view, shape (H, W), values in [0, N_k-1]

def sharp_nerf_forward(view_params, L_map):
    """
    view_params: Camera parameters and ray origins/directions for the current view.
    L_map: Sharpness level map for the current view.
    """
    # 1. Sample ray patches
    # Sample patches with stride 1. Each patch covers K x K pixels.
    # Let patch_coords be the set of top-left coordinates of these patches.
    patch_coords = sample_patches_with_stride_1(view_params, K=K)
    
    rendered_sharp_patches = []
    blurred_patches = []
    
    for coords in patch_coords:
        # 2. Render sharp patch
        # Extract rays for the K x K patch defined by coords
        rays = get_rays_for_patch(coords, view_params)
        
        # Render using grid-based backbone
        # Returns a tensor of shape (K, K, 3) for color
        I_c_prime = grid_model.render(rays)
        
        # 3. Kernel Indexing
        # Get sharpness levels for this patch from L_map
        # L_patch shape: (K, K)
        L_patch = L_map[coords.y : coords.y + K, coords.x : coords.x + K]
        
        # 4. Blurring Synthesis
        # For each pixel in the patch, select the kernel based on L_patch
        # and convolve I_c_prime with that kernel.
        # Since kernels are discrete and small, this can be implemented as:
        I_b = torch.zeros_like(I_c_prime)
        
        for k in range(N_k):
            # Mask of pixels in this patch with sharpness level k
            mask = (L_patch == k)
            if mask.any():
                # Get the k-th kernel
                kernel_k = B[k]
                # Convolve the sharp patch with the kernel
                # Note: Standard convolution might add padding; 
                # Sharp-NeRF likely uses 'same' padding or crops to match input size.
                # Given the "weighted sum" description, it implies a local operation.
                blurred_k = conv2d(I_c_prime, kernel_k, padding='same')
                
                # Add to result where mask is true
                # Using scatter or masked assignment
                I_b[mask] = blurred_k[mask]
        
        rendered_sharp_patches.append(I_c_prime)
        blurred_patches.append(I_b)
        
    return blurred_patches

def sharp_nerf_inference(view_params):
    """
    Inference only renders the sharp image.
    """
    rays = get_all_rays(view_params)
    I_c = grid_model.render(rays)
    return I_c
```

**Edge Cases and Pitfalls:**
*   **Memory Usage:** Retaining a unique blur kernel for every individual pixel is impractical for high-resolution and multi-view images due to memory constraints [S1]. Sharp-NeRF mitigates this by quantizing sharpness levels into a small number $N_k$ of discrete kernels [S1].
*   **Training Time:** Rendering multiple neighboring pixels (e.g., 5 or 6) to synthesize one pixel's blur increases training time proportionally to the number of rays per pixel [S1]. Sharp-NeRF handles this by rendering a $K \times K$ patch and applying the convolution operation over the entire patch, rather than ray-by-ray [S1].

## Operating characteristics
*   **Training Time:** Approximately half an hour [S1]. This is significantly faster than MLP-based methods like Deblur-NeRF and DP-NeRF [S1].
*   **Memory Complexity:** Reduced compared to MLP-based kernel generators. The grid-based representation is more memory-efficient than large MLPs, and the discrete kernel set $B$ is small ($N_k$ kernels) [S1].
*   **Strengths:** Fast training; handles spatially varying blur (defocus) effectively; does not require all-in-focus training data (unlike DoF-NeRF) [S1].
*   **Limitations:** Relies on the accuracy of the precomputed sharpness measure operator; quantization of sharpness levels may introduce discretization artifacts if $N_k$ is too small [S1].
*   **When to use:** When training time is a critical constraint and the input data consists of blurry images with spatially varying defocus blur.
*   **When not to use:** When the blur is motion blur with a fixed kernel (Hybrid Neural Rendering Model handles this with fixed kernels, but Sharp-NeRF is designed for spatially varying defocus) or when all-in-focus reference images are available (DoF-NeRF might be an alternative) [S1].

## Background
Sharp-NeRF was proposed to overcome the computational bottlenecks of MLP-based NeRF deblurring methods. Previous approaches like Deblur-NeRF and DP-NeRF used MLPs to generate blur kernels, which resulted in long training times and potential bias from the neural network's capacity limits [S1]. PDRF also used grid-based representations but relied on two MLPs (one for density, one for blur kernels) [S1]. Sharp-NeRF introduces a grid-based model with discrete learnable kernels and a sharpness prior, achieving high-quality results with significantly reduced training time [S1]. The method was presented in a paper published in 2024 [S1].

## Sources
[S1] Sharp-NeRF: Grid-based Fast Deblurring Neural Radiance Fields using Sharpness Prior, https://doi.org/10.1109/wacv57701.2024.00367, fetched 2026-10-27
