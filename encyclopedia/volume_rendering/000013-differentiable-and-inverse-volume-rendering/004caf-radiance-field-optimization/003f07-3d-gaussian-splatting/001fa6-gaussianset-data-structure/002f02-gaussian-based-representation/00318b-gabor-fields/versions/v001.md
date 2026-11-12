# Gabor Fields


## Overview
Gabor Fields are a continuous volumetric representation that decomposes a signal into a low-pass Gaussian base and a residual of anisotropic Gabor kernels, enabling orientation-selective level-of-detail and accelerated rendering via control variates [S1]. Unlike standard Gaussian-based representations, Gabor Fields achieve approximately 2 dB higher PSNR at equal primitive counts and render up to 1.4× faster at equal variance by concentrating spectral energy in specific frequency bands and orientations [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A target signal or volume to be represented.
*   **Output:** A set of Gabor primitives $\{g_i\}$, each defined by a Gaussian envelope and a cosine modulation, plus a low-pass Gaussian base.

**Data Structures:**
Each Gabor primitive $i$ is defined by the following parameters:
*   **Position:** $\mathbf{c}_i \in \mathbb{R}^3$ (center).
*   **Covariance:** $\mathbf{\Sigma}_i \in \mathbb{R}^{3 \times 3}$ (symmetric positive-definite matrix defining the Gaussian envelope's shape and orientation).
*   **Frequency Vector:** $\boldsymbol{\omega}_i \in \mathbb{R}^3$ (direction and magnitude of the cosine modulation).
*   **Amplitude/Opacity:** $\alpha_i \in \mathbb{R}$ (scalar weight).
*   **Color/Value:** $\mathbf{c}_i^{val} \in \mathbb{R}^3$ (RGB or scalar value).

The decomposition consists of two sets:
1.  **Gaussian Base:** A high-density set of primitives where $\boldsymbol{\omega}_i = \mathbf{0}$. These carry the low-frequency content.
2.  **Gabor Residuals:** A set of primitives where $\boldsymbol{\omega}_i \neq \mathbf{0}$. These carry high-frequency details.

**Mathematical Definition:**
The value $V(\mathbf{x})$ of the Gabor Field at point $\mathbf{x}$ is the sum of the Gaussian base $G(\mathbf{x})$ and the Gabor residuals $R(\mathbf{x})$:
$$ V(\mathbf{x}) = G(\mathbf{x}) + R(\mathbf{x}) = \sum_{i \in \text{Base}} \alpha_i e^{-\frac{1}{2}(\mathbf{x}-\mathbf{c}_i)^T \mathbf{\Sigma}_i^{-1} (\mathbf{x}-\mathbf{c}_i)} + \sum_{j \in \text{Residuals}} \alpha_j e^{-\frac{1}{2}(\mathbf{x}-\mathbf{c}_j)^T \mathbf{\Sigma}_j^{-1} (\mathbf{x}-\mathbf{c}_j)} \cos(\boldsymbol{\omega}_j \cdot (\mathbf{x}-\mathbf{c}_j)) $$

In the Fourier domain, each Gabor kernel consists of two symmetric lobes at $\pm \boldsymbol{\omega}_j$, concentrating the spectrum in a specific frequency band and orientation, unlike a plain Gaussian which spills across all frequencies [S1].

**Rendering Algorithm (Control Variate Path Tracing):**
The rendering leverages the Gaussian base as a control variate to reduce variance and accelerate intersection tests.

1.  **Preprocessing:**
    *   Compute the power spectrum for each Gabor kernel analytically.
    *   Group residuals by frequency band and orientation.
    *   Split frequency bands into oriented, steerable sub-bands to enable ray-space masking.

2.  **Ray Tracing Loop:**
    For each pixel, cast a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$.
    
    a. **Integrate Gaussian Base (Control Variate):**
    Always trace and analytically integrate the contribution of the Gaussian base primitives along the ray. This accounts for the majority of the density and low-frequency structure.
    
    b. **Stochastic Selection of Residuals:**
    For each sample point along the ray, stochastically select which Gabor residuals to include in the estimate.
    *   **Masking:** Before intersection, mask kernels that cannot contribute to the ray. This is done by checking if the ray direction aligns with the kernel's wave direction $\boldsymbol{\omega}_j$ (where contributions collapse due to cancellation) or if the kernel's frequency is above the current LOD cutoff.
    *   **Probability:** Select residuals based on a power-law probability over frequency. A parameter $\beta$ trades high-frequency error for speed [S1].
    
    c. **Closed-Form Integration:**
    For each selected Gabor residual, integrate its contribution in closed form along the ray segment.
    
    d. **Estimator:**
    The final pixel value is the sum of the deterministic Gaussian base integration and the stochastic sum of selected Gabor residuals.

**Parameters:**
*   **$\beta$:** A parameter for the power-law probability selection of Gabor residuals. Higher $\beta$ prioritizes lower frequencies, trading high-frequency accuracy for speed [S1].

**Edge Cases and Pitfalls:**
*   **Ray Alignment:** When a ray aligns with a Gabor kernel's wave direction $\boldsymbol{\omega}_j$, the positive and negative lobes cancel, causing the contribution to collapse. The algorithm must handle this via orientation-selective masking to avoid unnecessary intersection tests [S1].
*   **LOD Cutoff:** As the frequency cutoff shrinks for LOD, kernels are masked before tracing. This must be done continuously to avoid popping artifacts [S1].

## Operating characteristics
*   **Time Complexity:** Rendering is accelerated by reducing the number of primitive intersections per ray. The Gaussian base (~2% of total primitives) is always traced, while Gabor residuals are stochastically sampled.
*   **Memory Complexity:** Maintains two orders of magnitude compression over voxel grids [S1]. No extra storage for mipmaps or pyramids is required due to continuous LOD [S1].
*   **Performance:**
    *   In the Disney Cloud scene, unbiased reference rendering took 977 s. Control Variate PLA (Power-Law Accumulating) reduced this to 623 s. Zero NEE (Next-Event Estimation) further reduced it to 522 s, achieving nearly 2× the speed of the Control Variate PLA baseline with close to the same appearance [S1].
    *   Gabor Fields achieve ~2 dB higher PSNR at equal primitive count compared to Gaussian-based representations [S1].
    *   Gabor Fields render up to ~1.4× faster at equal variance compared to Gaussian-based representations [S1].
*   **Strengths:** Continuous LOD without re-fitting or extra storage; orientation-selective masking; efficient control variate acceleration; high PSNR.
*   **Limitations:** Stochastic nature introduces variance (mitigated by control variates); requires careful tuning of $\beta$ for speed/accuracy trade-off.

## Background
Gabor Fields were introduced by Jorge Condor, Nicolai Hermann, Mehmet Ata Yurtsever, and Piotr Didyk in "Gabor Fields: Orientation-Selective Level-of-Detail for Volume Rendering," published in ACM Transactions on Graphics (SIGGRAPH 2026) [S1]. The method builds on Gaussian-based representations by introducing anisotropic Gabor kernels to capture directional high-frequency details, enabling new forms of acceleration and LOD control.

## Sources
[S1] Gabor Fields: Orientation-Selective Level-of-Detail for Volume Rendering, https://arcanous98.github.io/projectPages/gaborVolumes.html, fetched 2026-10-26
