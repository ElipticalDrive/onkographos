# Grid-based Blur Kernel

Also called: Sharpness-indexed Blur Kernels, Trainable Grid-based Blur Kernels

## Overview
Grid-based Blur Kernel is a method for modeling spatially-varying blurriness in differentiable volume rendering by replacing MLP-generated blur kernels with a set of trainable, grid-indexed kernels [S1]. It solves the problem of accurately deblurring scenes with harsh defocus by allowing expressivity to scale with grid size without additional computational cost for kernel generation [S1]. Unlike [DARBF-Splatting](9911-darbf-splatting.md) which alters the kernel function itself, this method applies a spatially varying post-rasterization blur [Parent](10145-representation-and-kernel-modifications.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A set of training views, ground truth images $I_{gt}$, and a sharpness level map $L$ derived from a defocus map.
*   **Output:** A rendered blurred image $I_b$ and optimized blur kernels $\mathcal{B}$.

**Data Structures:**
*   $\mathcal{B}$: Tensor of shape $\mathbb{R}^{N_{img} \times N_k \times K \times K \times C}$ [S1].
    *   $N_{img}$: Number of training views.
    *   $N_k$: Number of unique sharpness groups (blur kernels).
    *   $K$: Kernel size (spatial dimension).
    *   $C$: Number of channels.
*   $L$: Sharpness level map, quantized defocus map with values in $\{0, \dots, N_k-1\}$ [S1].
*   $I_c'$: Cropped clean image tensor of shape $P' \times P' \times K \times K$ [S1].

**Parameters:**
*   $K$: Kernel size. Set to 11 in experiments [S1].
*   $N_k$: Number of blur kernels. Set to 400 in experiments [S1].
*   Learning rate for $\mathcal{B}$: 0.05 [S1].
*   Initialization: 2D Gaussian distribution [S1].
*   $P'$: Patch size dimension. In experiments, $P=22$ is sampled, implying $P'=22$ [S1].

**Steps:**
1.  **Preprocessing (Sharpness Prior):** Estimate a spatially varying defocus map for every training view using a Defocus Map Estimation Network (DMENet) [S1]. Quantize this map uniformly into $N_k$ values to create the sharpness level map $L$ [S1]. Optionally, segment the scene using depth information (e.g., from MIDAS) to assess focus levels within segments [S1]. This step is performed once and does not affect training time [S1].
2.  **Kernel Indexing:** For each pixel $x$ (with coordinates $h, w$) in a training view, determine its sharpness level $L[h,w]$ [S1]. Map this level to a specific blur kernel $w_x$ from the grid $\mathcal{B}$ using a mapping function $F$:
    $$ w_x = F(\mathcal{B}, L, x) $$
    where $w_x \in \mathbb{R}^{K \times K \times C}$ [S1]. Pixels sharing the same sharpness level index share the same kernel [S1].
3.  **Image Cropping:** Crop the clean image $I_c$ (the output of the standard rendering pipeline before blur) along height and width with stride 1 to obtain $I_c'$ of shape $P' \times P' \times K \times K$ [S1]. This captures the local neighborhood required for convolution [S1].
4.  **Blurred Image Convolution:** Compute the blurred image $I_b$ at each pixel $(h, w)$ by convolving the local crop $I_c'$ with the indexed kernel $w_x$:
    $$ I_b[h,w] = \sum_{i=0}^{K} \sum_{j=0}^{K} I_c'[h,w,i,j] w_x[h,w,i,j] $$
    where $I_b$ has shape $P' \times P'$ [S1].
5.  **Optimization:** Optimize $\mathcal{B}$ and the neural fields jointly using the reconstruction loss $\mathcal{L}_{recon}$:
    $$ \mathcal{L}_{recon} = ||I_b - I_{gt}||_2 $$
    [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
# B: Tensor of shape (N_img, N_k, K, K, C), initialized with 2D Gaussian
# L: Sharpness level map of shape (H, W), values in [0, N_k-1]
# I_gt: Ground truth image of shape (H, W, C)
# I_c: Clean rendered image from standard pipeline of shape (H, W, C)

def get_blur_kernel(B, L, h, w):
    """
    Retrieves the blur kernel for pixel (h, w) based on its sharpness level.
    """
    sharpness_level = L[h, w]
    # Assuming B is indexed by view, sharpness level, and kernel spatial dims
    # In practice, B might be (N_k, K, K, C) if shared across views, 
    # but source specifies B shape includes N_img. 
    # We assume the view index is handled by selecting the correct view's kernel set.
    # For a single view optimization step:
    return B[sharpness_level] 

def apply_grid_blur(B, L, I_c, I_gt, N_k, K, skip_groups):
    """
    Applies grid-based blur and computes loss.
    """
    H, W, C = I_c.shape
    I_b = torch.zeros_like(I_c)
    
    # Identify groups to skip (high sharpness)
    # Source mentions skipping convolution for pixels in 100 groups of each view with high sharpness value.
    # This implies L has high values for sharp pixels.
    # We assume skip_groups contains the indices of sharpness levels to skip.
    
    for h in range(H):
        for w in range(W):
            level = L[h, w]
            
            # Skip optimization/rendering for high sharpness pixels if configured
            if level in skip_groups:
                I_b[h, w] = I_c[h, w]
                continue
                
            # Get kernel for this sharpness level
            w_x = get_blur_kernel(B, L, h, w) # Shape: (K, K, C)
            
            # Crop local neighborhood from clean image
            # I_c' is conceptual; we access neighbors directly
            # Indices i, j range from 0 to K-1
            local_crop = I_c[max(0, h-K//2):min(H, h+K//2+1), 
                             max(0, w-K//2):min(W, w+K//2+1), :]
            
            # Handle boundary padding if necessary (not specified in source, assume valid indices or padding)
            # Source implies stride 1 crop of shape P'xP'xKxK, suggesting a pre-cropped tensor.
            # Here we simulate the convolution:
            # I_b[h,w] = sum_{i,j} I_c'[h,w,i,j] * w_x[i,j]
            
            # Simplified convolution for the specific pixel
            # Note: Source formula uses I_c'[h,w,i,j] which corresponds to neighbors
            # relative to (h,w).
            
            # To implement strictly from source formula:
            # I_c' is pre-cropped. Let's assume we have I_c_padded or handle bounds.
            # For brevity, assuming valid indexing within bounds or padding applied to I_c beforehand.
            
            # Convolution
            blur_val = 0.0
            for i in range(K):
                for j in range(K):
                    # Access neighbor. Source uses I_c'[h,w,i,j]. 
                    # This implies I_c' is structured such that [h,w] is the center 
                    # and i,j are offsets.
                    # In a flat tensor I_c, neighbor is at (h + i - K//2, w + j - K//2)
                    # Assuming K is odd (11), center is at 5.
                    offset_i = i - K // 2
                    offset_j = j - K // 2
                    nh, nw = h + offset_i, w + offset_j
                    
                    if 0 <= nh < H and 0 <= nw < W:
                        blur_val += I_c[nh, nw] * w_x[i, j]
                    else:
                        # Boundary handling not specified in source [G]
                        blur_val += 0.0 # Zero padding [G]
            
            I_b[h, w] = blur_val

    # Compute Loss
    loss = torch.mean((I_b - I_gt) ** 2)
    return loss, I_b
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The convolution requires neighboring pixels. The source does not specify padding strategy; zero-padding is a standard assumption [G].
*   **High Sharpness:** Pixels with high sharpness values (low defocus) are skipped to save computation. The threshold for "high" is not explicitly defined but corresponds to the top 100 groups out of $N_k$ [S1].
*   **Precision:** The method relies on differentiable indexing and convolution. Standard floating-point precision is assumed [G].

## Subtopics

### Sharpness Prior with Defocus Map Estimation
A preprocessing variant that estimates a spatially varying defocus map to determine pixel sharpness levels for kernel assignment [S1]. It uses DMENet, an end-to-end CNN, to estimate the defocus map for each training view [S1]. The map is quantized uniformly into $N_k$ values to create the sharpness level map $L$ [S1]. Depth information from MIDAS is used to segment the scene and assess focus levels within segments, improving the extraction of sharp regions [S1]. This step is performed once and does not impact training or testing time [S1].

## Operating characteristics
*   **Time Complexity:** The preprocessing (defocus estimation) takes less than a couple of minutes and is done once [S1]. Training time is not significantly increased compared to baseline NeRF because the grid-based kernels are indexed directly, avoiding the cost of MLP evaluation per pixel [S1].
*   **Memory Complexity:** Requires storing the kernel tensor $\mathcal{B}$ of shape $\mathbb{R}^{N_{img} \times N_k \times K \times K \times C}$ [S1]. With $N_{img}$ views, $N_k=400$, $K=11$, and $C=3$, this is a fixed overhead [S1].
*   **Performance:** Successfully renders clean images even from harshly defocused regions [S1]. Improves expressivity by increasing grid size without extra cost [S1].
*   **Strengths:** More expressive than MLP-based kernels; efficient indexing; preprocessing is fast [S1].
*   **Limitations:** Requires a reliable sharpness/defocus estimator (DMENet) [S1]. Performance depends on the accuracy of the sharpness prior [S1].

## Background
The method was introduced in "Sharp-NeRF: Grid-based Fast Deblurring Neural Radiance Fields using Sharpness Prior" [S1]. It addresses limitations in previous works that used MLPs to generate blur kernels, which were less expressive and computationally more expensive per pixel [S1]. The approach builds on the concept of spatially varying blur in NeRF-based deblurring [S1].

## Sources
[S1] Sharp-NeRF: Grid-based Fast Deblurring Neural Radiance Fields using Sharpness Prior, https://doi.org/10.1109/wacv57701.2024.00367, fetched 2026-10-27
