# Sharp-NeRF

Also called: Grid-based Fast Deblurring Neural Radiance Fields using Sharpness Prior

## Overview
Sharp-NeRF is a grid-based Neural Radiance Field (NeRF) framework designed to render clean, sharp images from input blurry images by modeling spatially varying defocus blur [S1]. It distinguishes itself from sibling methods like [DeblurNeRF](11964-deblurnerf.md) and PDRF by replacing neural network-based kernel generators with discrete learnable kernels indexed by a sharpness prior, and by using a decomposed-grid representation instead of MLPs, reducing training time to approximately half an hour [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of blurry training images (and their corresponding camera poses).
*   **Outputs:** A trained grid-based neural field model capable of rendering sharp novel views.

**Data Structures:**
1.  **Sharpness Level Map ($L$):** A 2D map of the same resolution as the input image, where each pixel value is an integer index in $[0, N_k - 1]$. This is computed during preprocessing and stored as a static texture or array.
2.  **Discrete Blur Kernels ($B$):** A set of $N_k$ learnable 2D convolution kernels. Each kernel $b_i$ has a fixed size $K \times K$ (where $K$ is the patch size, e.g., 5 or 6). The total parameters are $N_k \times K^2$.
3.  **Grid-based Neural Field:** A decomposed-grid representation (e.g., Triplane or Hash Grid) that maps 3D coordinates $(x, y, z)$ and viewing direction $(\theta, \phi)$ to density $\sigma$ and color $c$. This backbone renders the sharp image.

**Preprocessing:**
1.  For each training view, compute a defocus map (sharpness measure) using a sharpness measure operator (e.g., variance of Laplacian or gradient-based metric) [S1].
2.  Quantize the continuous defocus values into $N_k$ discrete bins to create the sharpness level map $L$. This step is performed once before training [S1].

**Training Steps:**
1.  **Ray Patch Sampling:** Instead of sampling individual rays, sample a patch of rays with stride 1. The spatial size of the patch corresponds to $K \times K$ pixels in the image plane [S1].
2.  **Sharp Rendering:** Pass the 3D coordinates of the rays in the patch through the grid-based neural field to render a sharp image patch $I_c'$ [S1].
3.  **Kernel Indexing:** For each pixel $(u, v)$ in the patch, retrieve its sharpness level index $l_{u,v}$ from the preprocessed map $L$. Use this index to select the corresponding blur kernel $b_{l_{u,v}}$ from the set $B$ [S1].
4.  **Blurring Synthesis:** Apply the selected kernel to the sharp patch $I_c'$ to synthesize the blurred patch $I_b$. This is implemented as a weighted sum of the sharp patch pixels convolved with the per-pixel kernel weights. Specifically, for a pixel $p$ in the output blurred patch:
    $$ I_b(p) = \sum_{k \in \text{kernel\_support}} I_c'(p - k) \cdot b_{l_p}(k) $$
    where $b_{l_p}$ is the kernel selected for pixel $p$ based on its sharpness level [S1].
5.  **Loss Computation:** Compare the synthesized blurred patch $I_b$ with the ground-truth blurry training image patch using a photometric loss (e.g., L1 or L2) [S1].
6.  **Optimization:** Update the grid-based neural field parameters and the discrete blur kernels $B$ using gradient descent [S1].

**Inference:**
1.  Sample rays for the target novel view.
2.  Render the sharp image $I_c$ directly using the trained grid-based neural field.
3.  The blur kernel module is bypassed; no blurring is applied [S1].

**Parameters:**
*   $N_k$: The number of discrete blur kernels. Not given in the sources.
*   $K$: The size of the blur kernel and the rendered patch (e.g., 5 or 6). Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Memory:** Storing a unique blur kernel for every pixel is impractical for high-resolution, multi-view images. Sharp-NeRF mitigates this by quantizing sharpness levels into $N_k$ discrete kernels [S1].
*   **Training Speed:** Rendering multiple neighboring pixels to simulate one blurred pixel increases training time proportionally. Sharp-NeRF handles this by rendering a $K \times K$ patch at once and applying the convolution operation over the patch, significantly reducing overhead [S1].

## Operating characteristics
*   **Time Complexity:** Training time is approximately half an hour, significantly faster than MLP-based methods like Deblur-NeRF and DP-NeRF [S1].
*   **Memory Complexity:** Lower than MLP-based methods due to the use of a decomposed-grid representation. The memory for blur kernels is $O(N_k \cdot K^2)$, which is small compared to the grid parameters [S1].
*   **Strengths:** Fast training, handles spatially varying defocus blur, preserves fine details and vivid colors [S1].
*   **Limitations:** Requires precomputed sharpness maps. Performance depends on the accuracy of the sharpness measure operator [S1].
*   **When to use:** When training time is a constraint and the blur is primarily defocus-based. Not suitable for motion blur or camera shake without modification [S1].

## Background
Sharp-NeRF was proposed to address the high computational cost of MLP-based NeRF methods for image deblurring, such as Deblur-NeRF and DP-NeRF, which suffer from long training times [S1]. It builds on grid-based NeRF architectures (e.g., [Triplane](link), [Hash Grid](link)) to accelerate rendering and uses a sharpness prior to model spatially varying blur without the need for a neural network to generate kernels, unlike PDRF which uses two MLPs [S1]. The method was evaluated on real-world datasets with blurry images, demonstrating superior training speed and competitive quantitative results [S1].

## Sources
[S1] Sharp-NeRF: Grid-based Fast Deblurring Neural Radiance Fields using Sharpness Prior, https://doi.org/10.1109/wacv57701.2024.00367, fetched 2026-10-27
