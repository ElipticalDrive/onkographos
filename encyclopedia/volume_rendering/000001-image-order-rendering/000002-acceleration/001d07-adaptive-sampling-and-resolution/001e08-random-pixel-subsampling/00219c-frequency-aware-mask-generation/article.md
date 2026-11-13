# Frequency-aware Mask Generation

Also called: Frequency-specific mask generation, Multi-scale frequency mask

## Overview
Frequency-aware Mask Generation produces binary spatial masks to guide neural rendering training by highlighting frequency-specific structures in rendered images [S2, unofficial]. It distinguishes between coarse structures (prioritized in early training) and fine structures (prioritized in late training) to improve high-frequency detail reconstruction [S2, unofficial]. Unlike random subsampling methods that select pixels uniformly or via simple heuristics, this method weights the reconstruction loss based on the spectral content of the ground truth image, allowing the model to adaptively focus optimization on relevant spatial frequencies [S1, S2, unofficial]. It builds on [Random Pixel Subsampling](1668-random-pixel-subsampling.md) by providing a structured, frequency-dependent selection signal rather than a purely stochastic one.

## How it works
The method comprises two distinct approaches found in the literature: a spectral Difference-of-Gaussians (DoG) method for importance scoring [S2, unofficial], and a parametric random sampling method for acceleration [S1, unofficial].

### Method 1: Spectral Difference-of-Gaussians Mask (DoG)
This method generates a binary mask $M_{freq}$ based on the image's frequency content using a schedule parameter $f$ [S2, unofficial].

**Inputs:**
- Rendered image tensor $I \in \mathbb{R}^{B \times C \times H_{img} \times W_{img}}$ (batch size $B$, channels $C$, height $H_{img}$, width $W_{img}$).
- Schedule parameter $f \in [0, 100]$, typically increasing linearly with training iteration $t$ as $f(t) = 100t/T_{total}$ [S2, unofficial].
- Downsampling scale factor $s$ (typically $0.5$) [S2, unofficial].
- Numerical stabilization constant $\varepsilon_{norm} > 0$ [S2, unofficial].

**Outputs:**
- Binary mask $M_{freq} \in \{0, 1\}^{B \times 1 \times H_{img} \times W_{img}}$ [S2, unofficial].

**Data Structures:**
- Grayscale image: $\mathbb{R}^{B \times 1 \times H \times W}$.
- Gaussian Kernels: 2D arrays of size $k_1 \times k_1$ and $k_2 \times k_2$.

**Steps:**
1. **Grayscale Conversion:** Average color channels to obtain $I_{gray}$:
   $$I_{gray} \leftarrow \text{mean}(I, \dim = 1, \text{keepdim} = \text{true})$$
2. **Downsampling:** Reduce resolution by factor $s$ using bilinear interpolation:
   $$I_{down} \leftarrow \text{BilinearInterp}(I_{gray}, s)$$
3. **Bandwidth Calculation:** Compute $\sigma_1$ based on $f$:
   $$
   \sigma_1 = \begin{cases} 
   0.1 + (100 - f) \times 0.1, & \text{if } f \geq 50 \\
   0.1 + f \times 0.1, & \text{otherwise}
   \end{cases}
   $$
   Set $\sigma_2 \leftarrow 2\sigma_1$ [S2, unofficial].
4. **Kernel Generation:** Compute kernel sizes and generate normalized 2D Gaussian kernels $\mathcal{K}_1, \mathcal{K}_2$:
   $$k_1 \leftarrow 2 \lceil 3\sigma_1 \rceil + 1, \quad k_2 \leftarrow 2 \lceil 3\sigma_2 \rceil + 1$$
   Kernels are normalized to unit sum [S2, unofficial].
5. **Filtering:** Apply dimension-preserving zero-padded 2D convolution:
   $$I_1^{blur} \leftarrow \mathcal{K}_1 * I_{down}, \quad I_2^{blur} \leftarrow \mathcal{K}_2 * I_{down}$$
6. **Difference-of-Gaussians:** Compute absolute difference response:
   $$H_{freq} \leftarrow |I_1^{blur} - I_2^{blur}|$$
7. **Upsampling:** Restore original resolution:
   $$H_{up} \leftarrow \text{BilinearInterp}(H_{freq}, (H_{img}, W_{img}))$$
8. **Normalization:** Normalize each image independently over spatial pixels:
   $$H_{norm} \leftarrow \frac{H_{up} - \min(H_{up})}{\max(H_{up}) - \min(H_{up}) + \varepsilon_{norm}}$$
   The minima and maxima are taken over spatial pixels only [S2, unofficial].
9. **Thresholding:** Generate binary mask $M_{freq}$:
   $$
   M_{freq} = \begin{cases} 
   \mathbb{I}[H_{norm} \geq 0.5], & \text{if } f \geq 50 \\
   \mathbb{I}[1 - H_{norm} \geq 0.5], & \text{if } f < 50
   \end{cases}
   $$
   where $\mathbb{I}[\cdot]$ is the indicator function [S2, unofficial].

**Edge Cases and Pitfalls:**
- If the spatial response is constant ($H_{norm}$ has zero variance), the denominator becomes $\varepsilon_{norm}$. Under this rule, $H_{norm} = 0$, yielding an all-one mask for $f < 50$ and an all-zero mask for $f \geq 50$ [S2, unofficial].
- Scores are normalized independently per image; heatmap colors indicate relative selection strength within each image, not absolute values [S2, unofficial].

### Method 2: Parametric Random Sampling Mask
This method generates binary masks for 1D or 2D data using Gaussian or uniform distributions, with optional center oversampling [S1, unofficial].

**Inputs:**
- Image size `size` (integer).
- Acceleration factor `acc_factor` (integer, default 8) [S1, unofficial].
- Mask type `type` (string: 'gaussian2d', 'uniformrandom2d', 'gaussian1d', 'uniform1d') [S1, unofficial].
- Center fraction `center_fraction` (float, default 0.04) [S1, unofficial].
- Covariance factor `cov_factor` (float, derived from size) [S1, unofficial].
- `fix` (boolean): If False, generate independent masks per batch element [S1, unofficial].

**Outputs:**
- Binary mask tensor of shape matching input image.

**Steps:**
1. **Calculate Sample Count:**
   - For 2D: $N_{samp} = \text{size}^2 // \text{acc\_factor}$
   - For 1D: $N_{samp} = \text{size} // \text{acc\_factor}$
2. **Initialize Mask:** Create tensor of zeros with shape matching image.
3. **Sample Coordinates:**
   - For Gaussian types: Sample from multivariate normal (2D) or normal (1D) with mean at center and covariance scaled by `size * cov_factor`.
   - For Uniform types: Sample uniformly from $[0, \text{size}-1]$.
4. **Clip Samples:** Clip coordinates to $[0, \text{size}-1]$ to handle boundary conditions [S1, unofficial].
5. **Set Mask Indices:** Set mask indices corresponding to sampled coordinates to 1.
6. **Center Oversampling (1D only):** If type is 'gaussian1d' or 'uniform1d', set a center region of size $N_{samp\_center} = \text{int}(\text{size} \times \text{center\_fraction})$ to 1 [S1, unofficial].
7. **Batch Independence:** If `fix=False`, repeat steps 3-6 for each batch element with independent random seeds [S1, unofficial].

**Pitfalls:**
- Random samples may fall outside valid grid bounds; clipping is required [S1, unofficial].

## Subtopics

### [Frequency-Modulated Edge-Aware Loss](4658-frequency-modulated-edge-aware-loss.md)
Uses a DoG filter to create an edge-aware mask, focusing optimization on specific frequency bands to tune the shape parameter $\beta$ and improve high-frequency detail reconstruction in Generalized Exponential Splatting (GES). It builds on [Frequency-aware Mask Generation](2537-frequency-aware-mask-generation.md) by spatially weighting the error based on the spectral content of the ground truth image, allowing the model to prioritize low-frequency structures early in training and high-frequency details later.

### [Cross-Dimensional Residual Frequency Compensation](14438-cross-dimensional-residual-frequency-compensation.md)
A frequency-domain compensation scheme that targets high-frequency details difficult to fit in the spatial domain, aiming to overcome inherent blurring artifacts of Gaussian kernels.

## Operating characteristics
- **Time Complexity:** The DoG method involves downsampling, two convolutions, upsampling, and normalization. Convolution complexity is $O(H \cdot W \cdot k^2)$ where $k$ is kernel size. The kernel size depends on $\sigma$, which is small (e.g., $\sigma_1 \in [0.1, 5.1]$), so $k$ is small (typically $< 11$). Thus, it is efficient relative to image size. The random sampling method is $O(N_{samp})$, which is significantly faster as $N_{samp}$ is a fraction of the total pixels.
- **Memory Complexity:** The DoG method requires storing intermediate downsampled, filtered, and upsampled images. Peak VRAM usage fluctuates within a bounded range without sustained upward trends over sequences [S2, unofficial]. The random sampling method requires minimal memory, only storing the mask and sampled coordinates.
- **Performance:** The DoG mask improves reconstruction quality, particularly in outdoor environments with clearer edges and finer texture details compared to FastGS and Speedy-Splat [S2, unofficial]. It reduces band-wise spectral magnitude errors [S2, unofficial].
- **Strengths:** Adapts to training stage (coarse to fine details); handles frequency-specific structures effectively; stable memory footprint [S2, unofficial].
- **Limitations:** Requires careful tuning of schedule parameter $f$ and normalization constants; DoG method is more computationally expensive than random sampling [S1, S2, unofficial].
- **When to Use:** Use DoG method when high-fidelity reconstruction of frequency-specific details is required and computational budget allows. Use random sampling method when acceleration is the primary goal and frequency awareness is secondary or handled by other means [S1, S2, unofficial].

## Background
Frequency-aware Mask Generation originates from research in efficient Gaussian Splatting, specifically the EffGS framework [S2, unofficial]. It was developed to address the challenge of reconstructing high-frequency details in neural rendering by providing a structured, frequency-dependent selection signal for training guidance [S2, unofficial]. The method is used in importance scoring and mask-weighted reconstruction loss throughout training [S2, unofficial]. The parametric random sampling variant is associated with the DGR framework [S1, unofficial].

## Sources
[S1] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
[S2] EffGS: Efficient and High-Fidelity Gaussian Splatting, https://arxiv.org/abs/2609.39553, fetched 2026-10-29 (unofficial)
