# Grid-based Blur Kernel

Also called: Sharpness-indexed Blur Kernels, Trainable Grid-based Blur Kernels

## Overview
Grid-based Blur Kernels replace MLP-generated blur kernels with a learnable tensor indexed by sharpness levels to model spatially-varying blurriness in NeRF optimization [S1]. Unlike [Negative Gaussian Splatting](9152-negative-gaussian-splatting.md) which modifies primitive signs, or [Scaffold-GS](9186-scaffold-gs.md) which uses structural priors, this method explicitly deconvolves defocus blur by assigning one of $N_k$ trainable kernels to pixels based on their local sharpness [S1]. It improves expressivity by increasing grid size without extra computational cost during kernel generation [S1].

## How it works
**Inputs:**
- Clean image $I_c$ (ground truth or reference).
- Sharpness level map $L$ (quantized defocus map).
- Number of training views $N_{img}$.
- Number of blur kernels $N_k$.
- Kernel size $K$.
- Number of channels $C$.

**Outputs:**
- Blurred image $I_b$.
- Optimized blur kernel tensor $\mathcal{B}$.

**Data Structures:**
- $\mathcal{B}$: Tensor of shape $\mathbb{R}^{N_{img} \times N_k \times K \times K \times C}$.
  - $N_{img}$: Number of training views.
  - $N_k$: Number of unique sharpness groups (blur kernels).
  - $K$: Spatial kernel size (e.g., 11).
  - $C$: Number of color channels.
- $L$: Map of shape $\mathbb{R}^{P' \times P'}$ containing integer indices $0 \dots N_k-1$ for each pixel, indicating which kernel from $\mathcal{B}$ to use.
- $I_c'$: Cropped image tensor of shape $\mathbb{R}^{P' \times P' \times K \times K \times C}$, obtained by cropping $I_c$ with stride 1.

**Parameters:**
- $K$: Kernel size. Typical value: 11 [S1].
- $N_k$: Number of blur kernels. Typical value: 400 [S1].
- Learning rate for $\mathcal{B}$: 0.05 [S1].
- Initialization: 2D Gaussian distribution [S1].
- $P'$: Patch size dimension. Typical value: 22 (patch size $P=22$) [S1].

**Steps:**
1. **Sharpness Prior Estimation:** Estimate a defocus map for each training view using a defocus map estimation network (e.g., DMENet) [S1]. Quantize the defocus map uniformly into $N_k$ values to create the sharpness level map $L$ [S1]. Optionally, segment the scene using depth information (e.g., from MIDAS) to assess focus levels within segments [S1]. This step is performed once as preprocessing [S1].
2. **Image Cropping:** Crop the clean image $I_c$ along height and width with stride 1 to obtain $I_c'$ of shape $P' \times P' \times K \times K \times C$ [S1].
3. **Kernel Selection:** For each pixel $(h, w)$ in the patch, determine its sharpness level $L[h, w]$. Select the corresponding blur kernel $w_x[h, w]$ from $\mathcal{B}$ using the mapping function $F$:
   $$ w_x = F(\mathcal{B}, L, x) $$
   where $w_x \in \mathbb{R}^{K \times K \times C}$ [S1].
4. **Blurred Image Synthesis:** Compute the blurred image $I_b$ by convolving the cropped clean image with the selected kernels:
   $$ I_b[h, w] = \sum_{i=0}^{K} \sum_{j=0}^{K} I_c'[h, w, i, j] \cdot w_x[h, w, i, j] $$
   where $h, w$ are indices along the height and width of the image and blur kernels, respectively [S1].
5. **Optimization:** Optimize $\mathcal{B}$ jointly with the neural fields using the reconstruction loss:
   $$ \mathcal{L}_{recon} = ||I_b - I_{gt}||_2 $$
   where $I_{gt}$ is the ground truth image [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_blurred_image(I_c, L, B, K, N_k, N_img, C):
    """
    I_c: Clean image tensor, shape (H, W, C)
    L: Sharpness level map, shape (H, W), values in [0, N_k-1]
    B: Blur kernel tensor, shape (N_img, N_k, K, K, C)
    K: Kernel size
    N_k: Number of kernels
    N_img: Number of views (index into B)
    C: Channels
    """
    H, W, C = I_c.shape
    # Step 2: Crop I_c with stride 1 to get I_c'
    # I_c' shape: (H, W, K, K, C)
    # For each pixel (h, w), I_c'[h, w, :, :] is a KxKxC patch centered at (h, w)
    # Note: Boundary handling for stride 1 crop is implementation-specific [G]
    I_c_prime = crop_with_stride_1(I_c, K) 
    
    I_b = torch.zeros((H, W, C))
    
    # Step 3 & 4: Kernel selection and convolution
    for h in range(H):
        for w in range(W):
            # Get sharpness level for this pixel
            sharpness_idx = L[h, w]
            
            # Select kernel for the specific view (assuming single view for simplicity, 
            # or batched processing)
            # B shape: (N_img, N_k, K, K, C)
            # We assume we are processing one view at index v
            kernel = B[0, sharpness_idx, :, :, :] # Shape: (K, K, C)
            
            # Get patch from cropped image
            patch = I_c_prime[h, w, :, :, :] # Shape: (K, K, C)
            
            # Convolve
            # I_b[h, w] = sum_{i,j} patch[i,j,:] * kernel[i,j,:]
            blurred_pixel = torch.sum(patch * kernel)
            I_b[h, w] = blurred_pixel
            
    return I_b

def crop_with_stride_1(I_c, K):
    """
    Crop I_c with stride 1 to get patches of size KxK.
    Returns tensor of shape (H, W, K, K, C).
    Assumes padding or boundary handling is needed for edges.
    """
    H, W, C = I_c.shape
    # Create a tensor to hold all patches
    # Using unfold or manual indexing
    # For simplicity, assuming valid padding or ignoring boundaries for this example
    # In practice, one might pad I_c first
    patches = []
    for h in range(H):
        for w in range(W):
            # Extract KxK patch centered at (h, w)
            # Handle boundaries by clamping or padding [G]
            h_start = max(0, h - K // 2)
            h_end = min(H, h + K // 2 + 1)
            w_start = max(0, w - K // 2)
            w_end = min(W, w + K // 2 + 1)
            
            patch = I_c[h_start:h_end, w_start:w_end, :]
            # Resize or pad patch to KxK if necessary [G]
            # This is a placeholder for the actual cropping logic
            pass 
    # Return full tensor structure
    return torch.zeros((H, W, K, K, C)) # Placeholder return
```

**Edge Cases and Pitfalls:**
- **Boundary Handling:** The cropping operation with stride 1 requires careful handling of image boundaries to ensure $I_c'$ has the correct shape [G].
- **High Sharpness Skip:** Pixels in 100 groups of each view with high sharpness values can skip convolution to save computation [S1].
- **Preprocessing Cost:** Sharpness prior calculation takes less than a couple of minutes and does not affect training/testing times [S1].

## Subtopics

### Sharpness Prior with Defocus Map Estimation
A preprocessing step that estimates a spatially varying defocus map to determine the sharpness level of image pixels, which is used to group pixels for blur kernel assignment [S1]. It uses DMENet, an end-to-end CNN framework, to estimate the defocus map for every training view [S1]. The defocus map is quantized uniformly into $N_k$ values to create the sharpness level map $L$ [S1]. Additionally, it segments the scene based on depth information from MIDAS to assess focus levels within each segment [S1]. This method is generic; any operator from the literature can be used as a sharpness prior as long as it reliably measures sharpness [S1].

## Operating characteristics
- **Time Complexity:** The sharpness prior is computed once as preprocessing, taking less than a couple of minutes [S1]. During training, the grid-based kernels are optimized jointly with neural fields, starting from the first iteration [S1].
- **Memory Complexity:** The blur kernel tensor $\mathcal{B}$ has size $N_{img} \times N_k \times K \times K \times C$. With $N_{img}$ views, $N_k=400$, $K=11$, and $C=3$, this is a significant but manageable memory footprint [S1].
- **Performance:** Successfully renders clean images even from harshly defocused regions [S1]. Improves expressivity by increasing grid size without extra cost to produce kernels [S1].
- **Strengths:** Direct optimization of blur kernels allows modeling spatially-varying blurriness more effectively than MLP-generated kernels [S1].
- **Limitations:** Requires a sharpness prior (defocus map) as input, which must be estimated beforehand [S1].

## Background
The method was introduced in "Sharp-NeRF: Grid-based Fast Deblurring Neural Radiance Fields using Sharpness Prior" [S1]. It addresses the limitation of previous works that used MLPs to generate blur kernels, which were less expressive and computationally more expensive to generate [S1]. The approach leverages the transversal relation among neighboring rays and uses depth information to improve focus region extraction [S1].

## Sources
[S1] Sharp-NeRF: Grid-based Fast Deblurring Neural Radiance Fields using Sharpness Prior, https://doi.org/10.1109/wacv57701.2024.00367, fetched 2026-10-27
