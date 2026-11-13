# Grid-based Blur Kernel

Also called: Sharpness-indexed Blur Kernels, Trainable Grid-based Blur Kernels

## Overview
Grid-based Blur Kernels replace MLP-generated blur kernels with a learnable grid of kernels indexed by sharpness level to model spatially-varying blurriness in radiance fields [S1]. This method improves expressivity by allowing kernel capacity to scale with grid size without additional computational cost for kernel generation, enabling effective deblurring even in harshly defocused regions [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training views, ground truth images $I_{gt}$, sharpness level map $L$ (precomputed), number of blur kernels $N_k$, kernel size $K$, number of channels $C$.
*   **Outputs:** Blurred image $I_b$, optimized blur kernel grid $\mathcal{B}$.

**Data Structures:**
*   **Blur Kernel Grid ($\mathcal{B}$):** A tensor of shape $\mathbb{R}^{N_{img} \times N_k \times K \times K \times C}$.
    *   $N_{img}$: Number of training views.
    *   $N_k$: Number of unique sharpness groups (blur kernels).
    *   $K$: Spatial kernel size (e.g., 11).
    *   $C$: Number of color channels.
    *   *Note:* The channel dimension $C$ is often omitted in notation for brevity [S1].

**Parameters:**
*   $K$: Kernel size. Default value: 11 [S1].
*   $N_k$: Number of unique sharpness groups. Default value: 400 [S1].
*   Learning Rate: 0.05 for the kernel parameters [S1].
*   Initialization: Blur kernels are initialized with a 2D Gaussian distribution [S1].
*   Patch Sampling: Random patch sampling is used. Patch size $P=22$, sampling 28 patches per iteration [S1].

**Steps:**
1.  **Preprocessing (Sharpness Prior):** Estimate a spatially varying defocus map for every training view using a Defocus Map Estimation Network (DMENet) [S1]. Quantize the defocus map uniformly into $N_k$ values to create the sharpness level map $L$ [S1]. Optionally, segment the scene based on depth information (e.g., from MIDAS) to assess focus levels within segments [S1]. This step is performed once and does not affect training time [S1].
2.  **Kernel Indexing:** For each pixel $x$ in a training view, determine its sharpness level $L(x)$ from the map $L$. Map this level to a specific blur kernel $w_x$ from the grid $\mathcal{B}$ using a mapping function $F$:
    $$ w_x = F(\mathcal{B}, L, x) $$
    Here, $F$ retrieves the kernel corresponding to the sharpness group of pixel $x$ [S1].
3.  **Image Cropping:** Crop the clean image $I_c$ (the output of the radiance field) along height and width with stride 1 to obtain a cropped tensor $I'_c$ of shape $P' \times P' \times K \times K$. This tensor contains local neighborhoods for each pixel in the $P' \times P'$ region of interest [S1].
4.  **Blurring:** Compute the blurred image $I_b$ by convolving the cropped clean image with the indexed blur kernels:
    $$ I_b[h,w] = \sum_{i=0}^{K} \sum_{j=0}^{K} I'_c[h,w,i,j] w_x[h,w,i,j] $$
    where $h, w$ are indices along the height and width of the image [S1].
5.  **Optimization:** Optimize both the blur kernels $\mathcal{B}$ and the neural fields jointly using the reconstruction loss $\mathcal{L}_{recon}$:
    $$ \mathcal{L}_{recon} = ||I_b - I_{gt}||_2 $$
    where $I_{gt}$ is the ground truth image [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def grid_based_blur_kernel_step(I_c, L_map, B, K, N_k, N_img, C):
    """
    Performs one step of grid-based blur kernel application and loss calculation.
    
    Args:
        I_c: Clean image from radiance field, shape (H, W, C).
        L_map: Sharpness level map, shape (H, W), values in [0, N_k-1].
        B: Blur kernel grid, shape (N_img, N_k, K, K, C).
        K: Kernel size.
        N_k: Number of blur kernels.
        N_img: Number of training views (index for B).
        C: Number of channels.
        
    Returns:
        I_b: Blurred image, shape (H, W, C).
        loss: Reconstruction loss.
    """
    H, W, _ = I_c.shape
    
    # 1. Crop I_c to get local neighborhoods for all pixels
    # I_c' shape: (H, W, K, K, C)
    # For each pixel (h, w), I_c'[h, w, :, :, :] is the KxK patch centered at (h, w)
    # Note: Boundary handling (padding) is required for the crop operation.
    # Assuming valid padding or boundary handling is implemented in crop function.
    I_c_prime = crop_with_stride_1(I_c, K) 
    
    # 2. Initialize blurred image
    I_b = torch.zeros_like(I_c)
    
    # 3. Iterate over pixels to apply indexed blur kernels
    # Optimization: Skip convolution for high sharpness pixels if configured
    # For implementation completeness, we show the general case.
    for h in range(H):
        for w in range(W):
            # Get sharpness level for pixel (h, w)
            sharpness_idx = int(L_map[h, w])
            
            # Retrieve blur kernel for this sharpness level and view
            # B shape: (N_img, N_k, K, K, C)
            kernel = B[N_img, sharpness_idx, :, :, :]
            
            # Get local patch from cropped image
            # I_c_prime shape: (H, W, K, K, C)
            patch = I_c_prime[h, w, :, :, :]
            
            # Convolve patch with kernel
            # Element-wise multiplication and sum over K, K, C
            blurred_pixel = torch.sum(patch * kernel)
            
            I_b[h, w] = blurred_pixel
            
    # 4. Calculate Loss (assuming I_gt is available)
    # loss = MSE(I_b, I_gt)
    return I_b
```

## Subtopics

### Sharpness Prior with Defocus Map Estimation
A preprocessing variant that estimates a spatially varying defocus map to determine pixel sharpness levels for kernel assignment [S1]. It uses DMENet (an end-to-end CNN) to estimate the defocus map and quantizes it into $N_k$ values to create the sharpness level map $L$ [S1]. It may also use MIDAS for depth segmentation to assess focus levels within scene segments [S1]. This step is performed once before training [S1].

## Operating characteristics
*   **Time Complexity:** The preprocessing step (defocus map estimation) takes less than a couple of minutes and does not affect training or testing times [S1]. During training, the method uses random patch sampling (e.g., 28 patches of size $22 \times 22$) to reduce the number of neighboring rays required per pixel from $K^2$ to $\frac{K^2}{2} (\frac{P'+K+1}{P'})^2$ compared to full ray sampling [S1].
*   **Memory Complexity:** The blur kernel grid $\mathcal{B}$ has size $N_{img} \times N_k \times K \times K \times C$. With $N_{img}=1$, $N_k=400$, $K=11$, $C=3$, this is manageable. The method avoids the memory overhead of storing per-pixel kernels by sharing kernels within sharpness groups [S1].
*   **Performance:** Successfully renders clean images even from harshly defocused regions [S1]. Improves expressivity by increasing grid size without extra cost for kernel generation [S1].
*   **Strengths:** Joint optimization with neural fields starting from the first iteration; rapid convergence of neural fields allows effective kernel learning [S1]. Generic applicability of the sharpness prior (any operator measuring sharpness can be used) [S1].
*   **Limitations:** Requires accurate defocus map estimation; performance depends on the quality of the sharpness prior [S1].

## Background
Originated in Sharp-NeRF, which proposed grid-based fast deblurring neural radiance fields using a sharpness prior [S1]. It addresses the limitation of previous works that used MLPs to generate blur kernels, which were less expressive and computationally more expensive to generate [S1]. The method leverages the transversal relation among neighboring rays and depth information to improve deblurring [S1].

## Sources
[S1] Sharp-NeRF: Grid-based Fast Deblurring Neural Radiance Fields using Sharpness Prior, https://doi.org/10.1109/wacv57701.2024.00367, fetched 2026-10-27
