# NAN (Noise-Aware NeRFs)

Also called: IBRNet-based architecture with noise-aware extensions, NAN method

## Overview
NAN is a burst-denoising method that extends the IBRNet architecture to handle degraded noisy images, such as those captured in low-light conditions, by modifying feature extraction, aggregation, and color blending to be noise-aware [S1]. Unlike standard NeRFs that synthesize novel views, NAN fixes an existing view image by leveraging multiple noisy inputs of the same scene [S1]. It distinguishes itself from its sibling [MIMO-DONeRF](16217-mimo-donerf.md) and [Point-NeRF Adaptation for Object Manipulation](16395-point-nerf-adaptation-for-object-manipulation.md) by explicitly modeling spatial noise statistics through 3D blending kernels and cross-view variance features, rather than relying solely on single-pixel cross-view consistency [S1].

## How it works

### Inputs and Outputs
**Inputs:** A burst of $N$ noisy RGB images of the same scene, captured from different viewpoints. Each image is associated with its camera pose. The noise model is assumed to be additive white Gaussian noise (AWGN) in linear space, potentially with per-pixel variance estimates [S1].
**Outputs:** A single denoised RGB image corresponding to a target viewpoint (which may be one of the input views or a novel view, though the primary application is burst denoising of an existing view) [S1].

### Data Structures and Layout

**Pre-Net Layer:**
A single convolutional layer at the entry point of the network [S1].
*   **Kernel Size:** $3 \times 3$ [S1].
*   **Output Channels:** 3 (RGB) [S1].
*   **Activation:** None [S1].
*   **Initialization:** Gaussian per-channel filters [S1].
*   **Function:** Preserves spatial and channel dimensions; reduces input noise efficiently [S1].

**Feature Aggregation Input:**
Pixel feature embedding vectors are expanded by concatenation to include cross-view mean and variance feature vectors for the specific pixel [S1]. Additionally, estimated noise variances per-pixel per-color channel are concatenated to the global features [S1].

**Transformer Module:**
A standard single 5-head self-attention transformer is added to the feature aggregation stage to capture intricate relations and statistics of cross-view features [S1].

**Color Blending Weights:**
Instead of outputting $N$ blending scalars, the network outputs $N$ vectors, each of dimension $k^2 \times 3$ [S1]. This represents a per-pixel per-view $k \times k$ spatial kernel of blending weights, with separate weights per color channel [S1].

### Numbered Steps

1.  **Preprocessing:** Linearize input images by applying inverse gamma correction and inverse random white balancing. This ensures the noise model, which refers to linear images, is correctly applied [S1].
2.  **Pre-Net Processing:** Pass the linearized input images through the Pre-Net layer (single $3 \times 3$ convolution, no activation) to perform initial noise reduction while preserving spatial resolution [S1].
3.  **Feature Extraction:** Extract pixel feature embeddings from the Pre-Net outputs using the truncated ResNet34 encoder-decoder network from IBRNet [S1].
4.  **Ray Projection:** For each pixel in the target image, project a ray into the scene. Sample points along the ray [S1].
5.  **Cross-View Projection:** Project each 3D sample point onto each of the $N$ input views to retrieve corresponding features [S1].
6.  **Feature Aggregation:**
    *   Compute cross-view mean and variance feature vectors for the specific pixel [S1].
    *   Concatenate these statistics to the pixel feature embedding vectors [S1].
    *   Concatenate estimated noise variances (per-pixel per-color channel) to the global features [S1].
    *   Process the aggregated features through the 5-head self-attention transformer to capture intricate cross-view relations [S1].
7.  **Density and Color Estimation:**
    *   Predict the point color $c_m$ and density $\rho_m$ for each sample $m$ along the ray [S1].
    *   Calculate the probability $w_m$ that an object exists at sample $m$:
        $$ w_m = 1 - e^{-\rho_m / \sum_{j=1}^{M} \rho_j} $$
        where $\rho_m$ is the density and $M$ is the total number of samples [S1].
8.  **Noise-Aware Color Blending (Spatial Kernel):**
    *   Instead of a single scalar weight per view, predict a $k \times k$ spatial kernel of blending weights for each view and each color channel [S1].
    *   The output is $N$ vectors, each of dimension $k^2 \times 3$ [S1].
    *   Normalize the weights per channel to sum to 1 using a softmax operation prior to blending [S1].
    *   Apply these kernels to the $k \times k$ spatial neighborhood of each projected pixel to gather spatial context [S1].
9.  **Rendering:**
    *   Integrate the weight-color pairs along the ray to produce the final color $\hat{C}(r)$:
        $$ \hat{C}(r) = \sum_{m=1}^{M} w_m c_m $$
        where $w_m$ is the weight, $c_m$ is the color, and $M$ is the total samples [S1].
10. **Coarse/Fine Sampling:** Repeat the sampling process in two stages. The 'fine' stage refines sampling around depth values that most influence rendering (visible surfaces) [S1].
11. **Post-processing:**
    *   Apply the respective gamma correction and white balancing to the output linear denoised image to return it to display space [S1].
    *   Apply a simple bilateral filtering post-process to the main network output [S1].

### Parameters
*   **Spatial Kernel Size ($k$):** Set to 3. This provides a balance between quality gain and increase in network complexity [S1].
*   **Transformer Heads:** 5 [S1].

### Edge Cases and Pitfalls
*   **Original IBRNet Feature Extraction:** The original IBRNet uses a $7 \times 7$ kernel size and immediate spatial resolution reduction by a factor of two at the entry point. This makes the layer incapable of performing simple noise filtering, necessitating the addition of the Pre-Net layer [S1].
*   **Standard Blending Insufficiency:** Standard IBRNet blending gathers only cross-view (single pixel) information without spatial extent. This is insufficient for noise filtering, as noise is a spatial phenomenon. The introduction of $k \times k$ spatial kernels addresses this [S1].
*   **Clean vs. Noisy Regimes:** In clean images, IBRNet ignores feature statistics (mean/variance) because variance is low for surface points. In noisy regimes, individual features become more significant than joint statistics, necessitating the transformer module [S1].
*   **Signal-Dependent Noise:** The separation of blending weights per color channel is important, especially in the case of signal-dependent noise, which can vary across color channels [S1].

## Operating characteristics

### Time and Memory Complexity
Not given in the sources. The addition of the Pre-Net layer, transformer module, and spatial kernels increases network complexity compared to standard IBRNet, but the exact computational cost is not quantified [S1].

### Measured Performance
Not given in the sources. The paper demonstrates improved denoising performance on the LLFF-N dataset compared to baseline IBRNet-N, but specific metrics (PSNR, SSIM) are not detailed in the provided notes [S1].

### Scaling
Not given in the sources.

### Strengths and Limitations
*   **Strengths:** Effectively handles degraded noisy images (e.g., low-light) by leveraging spatial context via 3D blending kernels and cross-view statistics via transformers [S1]. The Pre-Net layer efficiently reduces input noise without losing spatial resolution [S1].
*   **Limitations:** The method is designed for burst denoising of existing views rather than novel view synthesis. The reliance on accurate noise variance estimation may be a limitation if noise statistics are unknown or misestimated [S1].

### When to Use
Use NAN when the goal is to denoise a burst of noisy images of the same scene (burst denoising) rather than synthesizing novel views. It is particularly suitable for low-light conditions where noise is significant [S1].

### When Not to Use
Do not use NAN if the primary task is novel view synthesis from clean images, as standard IBRNet or other NeRF variants may be more appropriate. It is also not designed for single-image denoising without a burst of inputs [S1].

## Background

### Origin
NAN was introduced in the paper "NAN: Noise-Aware NeRFs for Burst-Denoising" [S1]. It extends the IBRNet architecture [29] to address the challenges of burst denoising, which differs from novel view synthesis in its focus on degraded noisy images and fixing existing views [S1].

### History and Development
The method builds upon IBRNet, which was designed for novel view synthesis with significant camera motion and complex 3D geometry [S1]. The authors observed that standard IBRNet performs poorly on noisy inputs due to its feature extraction and blending mechanisms. They introduced the Pre-Net layer, cross-view statistics aggregation, a transformer module, and 3D blending kernels to adapt IBRNet for noise-aware rendering [S1].

### Where It Is Used
NAN is used for burst denoising of images captured in low-light conditions, where multiple noisy views of the same scene are available [S1]. It is evaluated on the LLFF-N dataset, which consists of noisy versions of the LLFF dataset [S1].

## Sources

[S1] NAN: Noise-Aware NeRFs for Burst-Denoising, https://doi.org/10.1109/cvpr52688.2022.01234, fetched 2026-10-30
