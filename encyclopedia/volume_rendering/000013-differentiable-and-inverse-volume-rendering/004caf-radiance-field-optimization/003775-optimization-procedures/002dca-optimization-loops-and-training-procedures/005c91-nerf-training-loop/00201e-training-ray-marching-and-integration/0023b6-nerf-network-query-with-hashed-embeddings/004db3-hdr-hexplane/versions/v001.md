# HDR-HexPlane

Also called: Fast High Dynamic Range Radiance Fields for Dynamic Scenes

## Overview
HDR-HexPlane is an end-to-end neural radiance field framework for dynamic scenes that learns 3D scene representations from dynamic 2D images captured with various exposures, allowing for novel view synthesis with adjustable exposure levels [S1]. It explicitly separates the modeling of dynamic scene geometry/radiance from the learning of scene illumination (exposure), addressing color inconsistency caused by objects moving between dark and bright regions [S1]. Unlike static HDR-NeRF methods, HDR-HexPlane handles dynamic scenes by incorporating temporal information (timestamp $t$) into the HexPlane representation [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $J$ LDR images captured from different viewpoints and times, with known or unknown exposure levels. Each image $j$ has an associated index.
*   **Outputs:** Rendered LDR images for novel views and times, and optionally HDR radiance fields.

**Data Structures:**
*   **HexPlane Representation:** Based on [HexPlane](4842-hexplane.md), this uses 6 learnable parameter planes ($M_{rXY}, M_{rXZ}, M_{rYZ}, M_{rXT}, M_{rYT}, M_{rZT}$) for spatial and temporal features. The voxel grid representation $D$ is computed via bilinear interpolation and vector-matrix products [S1].
*   **Exposure Embedding:** A feature embedding vector $a_j$ for each image index $j$.
*   **Exposure MLP:** A Multi-Layer Perceptron $\phi_e$ that maps $a_j$ to the exposure coefficient $e_j$.
*   **Camera Response Function (CRF):** A fixed sigmoid function $g(x) = \frac{1}{1 + e^{-x}}$ [S1].

**Numbered Steps:**

1.  **Ray Sampling:** For each ray $r$ with origin $o$, direction $d$, and time $t$, sample a series of points $x$ along the ray between near and far clipping distances [S1].
2.  **HexPlane Query:** Feed the sampled points $x$, current timestamp $t$, and direction $d$ into the HexPlane module $f_\beta$ to calculate the logarithmic spatial radiance value $E' = \ln(E)$ and volume density $\sigma$ [S1].
    $$ (E', \sigma) = f_\beta(x, d, t) $$
3.  **Exposure Mapping:** For the image index $j$ corresponding to the ray's source image:
    *   Retrieve the feature embedding $a_j = \ell(j)$ [S1].
    *   Compute the logarithmic exposure coefficient $e_j$ using the exposure MLP $\phi_e$:
        $$ e_j = \phi_e(\ell(j)) $$
    *   Note: The final exposure coefficient used in the color calculation is the exponential of this value, or the value is treated as $\ln(e_j)$ in the sum. The paper defines $e_j$ as the coefficient such that $\ln(e_j)$ is added to $\ln(E)$ [S1]. Specifically, Eq 3.10 uses $\ln(e_j)$ where $e_j$ is the output of $\phi_e$? No, Eq 3.11 says $e_j = \phi_e(\ell(j))$. Eq 3.10 says $c_j = g(\ln E + \ln e_j)$. This implies $\phi_e$ outputs the log-exposure or the exposure itself? The text says "logarithmic exposure coefficients $e_j$" in the caption of Fig 2, but Eq 3.11 defines $e_j$ directly. Eq 3.10 uses $\ln e_j$. If $e_j$ is the coefficient, then $\ln e_j$ is the log-coefficient. Let's assume $\phi_e$ outputs the exposure value $e_j$, and we take its log. Or $\phi_e$ outputs the log-exposure. The text says "learn the logarithmic exposure coefficient $e_j$" in the overview, but Eq 3.11 defines $e_j$. Let's look at Eq 3.10: $c_j = g(\ln E + \ln e_j)$. If $e_j$ is the exposure coefficient, then $\ln e_j$ is the log-exposure. The text in Sec 3.3 says "map the spatial radiance and exposure into the logarithm domain... $c_j = g(\ln E + \ln e_j)$". It also says "learn the corresponding exposure coefficients". We will assume $\phi_e$ outputs $e_j$ (the linear exposure coefficient) and we use $\ln(e_j)$ in the formula.
4.  **Spatial Color Computation:** Compute the spatial color $c_j$ for image index $j$ using the camera response function $g$ applied to the sum of logarithmic spatial radiance $E'$ and logarithmic exposure $\ln(e_j)$:
    $$ c_j = g(\ln E + \ln e_j) $$
    where $g(x) = \frac{1}{1 + e^{-x}}$ [S1].
5.  **Volume Rendering:** Render the LDR image using the volume rendering equation with the computed spatial color $c$ and density $\sigma$:
    $$ \hat{C}(r) = \int_{t_n}^{t_f} T(t) \sigma(r(t)) c(r(t), d) dt $$
    where $T(t) = \exp\left(-\int_{t_n}^{t} \sigma(r(s)) ds\right)$ [S1].

**Formulas:**
*   **HexPlane Feature Aggregation:**
    $$ D = \sum_{r=1}^{R_1} M_{rXY} \odot M_{rZT} \odot v_{r1} + \sum_{r=1}^{R_2} M_{rXZ} \odot M_{rTY} \odot v_{r2} + \sum_{r=1}^{R_3} M_{rYZ} \odot M_{rXT} \odot v_{r3} $$
    where $M_{AB} \in \mathbb{R}^{A \times B}$ are feature planes and $v$ are vectors derived from position and time [S1].
*   **Camera Response Function:**
    $$ g(x) = \frac{1}{1 + e^{-x}} $$
    [S1]
*   **Zero-Point Constraint Loss:**
    $$ L_u = \| g(0) - C_0 \| $$
    where $C_0$ is a constant [S1].
*   **Total Loss:**
    $$ L = \| C - \hat{C} \|^2 + w_{tv} L_{tv} + \lambda_u L_u $$
    where $C$ is the ground truth LDR pixel color, $L_{tv}$ is the total variational loss on planes, $w_{tv}$ is the weight for TV loss, and $\lambda_u$ is the weight for the zero-point constraint [S1].

**Parameters:**
*   **Tone Mapping (Evaluation):** $\mu = 5000$. $M(E) = \frac{\log(1 + \mu E)}{\log(1 + \mu)}$ [S1].
*   **Learning Rates:**
    *   Time offset parameter: $2 \times 10^{-2}$ [S1].
    *   Exposure embedding: $2 \times 10^{2}$ [S1].
    *   Exposure MLP: $1 \times 10^{3}$ [S1].
*   **Optimizer:** Adam with $\alpha = 0.9, \beta = 0.99$ [S1].
*   **Time Grid:** Initial value 16, expanded to 24 during training [S1].

**Pitfalls:**
*   **Ambiguity:** Scaling $E$ by $\alpha$ and giving $\ln \alpha$ offset to $g$ results in the same outcome. This is handled by fixing $g(0)$ to a constant $C_0$ via the zero-point constraint loss [S1].
*   **Overfitting:** Without explicit exposure modeling, HexPlane tends to overfit scene exposure, achieving good training results but failing to render high-quality novel view images [S1].
*   **Unknown Exposure:** Estimating both Camera Response Function (CRF) and exposure values when exposure is unknown can be difficult. HDR-HexPlane fixes the CRF to a sigmoid to ease optimization [S1].

## Operating characteristics
*   **Time Complexity:** Accelerates training speed over $10\times$ compared to NeRF-WT [S1]. Training time is approximately 46 minutes on synthetic datasets (80-700 images, 800x800 resolution) using a single RTX A5000 [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Outperforms HexPlane, NeRF-WT, HDR-Plenoxels, and HDR-NeRF in PSNR, SSIM, and LPIPS on synthetic dynamic HDR datasets [S1].
*   **Strengths:** Handles dynamic scenes with varying exposures; explicit separation of geometry/radiance and illumination; stable learning via fixed sigmoid CRF.
*   **Limitations:** Relies on the assumption that the camera response function can be approximated by a sigmoid; performance depends on the quality of the HexPlane representation for dynamic scenes.

## Background
HDR-HexPlane was proposed to extend HDR NeRF methods to dynamic scenes, addressing the limitations of static HDR-NeRF methods which fail to reconstruct dynamic scenes [S1]. It builds on HexPlane [4] for efficient dynamic scene representation and HDR-NeRF [22] for HDR synthesis. The method was introduced in 2024 [S1].

## Sources
[S1] Fast High Dynamic Range Radiance Fields for Dynamic Scenes, https://doi.org/10.1109/3dv62453.2024.00052, fetched 2026-10-27
