# LC Score Weighting

Also called: LC score sensitivity analysis

## Overview
LC Score Weighting defines the scalar formula used to compute the Likelihood Constraint (LC) score, which guides Gaussian reallocation in error-guided optimization. It solves the problem of determining which primitives are most likely to cause reconstruction errors by combining mean color error, maximum color error, and mean gradient magnitude. It is distinguished from other weighting schemes by its exponential decay form, which penalizes high-error regions more aggressively than linear sums.

## How it works
**Inputs:**
*   $C_{mean}$: Mean color error of a primitive or region.
*   $C_{max}$: Maximum color error of a primitive or region.
*   $\bar{G}_{mean}$: Mean gradient magnitude of a primitive or region.
*   $z(\cdot)$: A normalization or scaling function applied to the input measures (specific implementation details of $z$ are not provided in the sources, but it is applied to each measure individually) [S1].

**Outputs:**
*   $s_{LC}$: The LC score, a scalar value in $(0, 1]$.

**Parameters:**
*   $\lambda_1, \lambda_2, \lambda_3$: Weights for $C_{mean}$, $C_{max}$, and $\bar{G}_{mean}$ respectively.
*   Default configuration: $(\lambda_1, \lambda_2, \lambda_3) = (1, 1, 1)$ [S1].

**Formula:**
The LC score is calculated as:
$$ s_{LC} = \exp(-\lambda_1 z(C_{mean}) - \lambda_2 z(C_{max}) - \lambda_3 z(\bar{G}_{mean})) $$

**Steps:**
1.  Compute the normalized/scaling-transformed values $z(C_{mean})$, $z(C_{max})$, and $z(\bar{G}_{mean})$ for the target primitive or region.
2.  Multiply each transformed value by its corresponding weight $\lambda_i$.
3.  Sum the weighted values.
4.  Apply the exponential function to the negative sum to produce $s_{LC}$.

**Edge Cases and Pitfalls:**
*   **Insensitivity:** The score is largely insensitive to moderate changes in weight configurations. On Tanks and Temples and Deep Blending datasets, multi-scale average PSNR varies by no more than 0.10 dB across tested configurations (e.g., $(1,1,1)$, $(2,1,1)$, $(1,2,1)$, etc.) [S1].
*   **Default Selection:** Because no single configuration consistently outperforms others across datasets and scales, the equal-weight configuration $(1,1,1)$ is retained as the fixed default for stability [S1].

## Subtopics

### Scene-Level Constraint Coefficient (SCC)
A coefficient estimating scene degradation from training view statistics to uniformly scale loss weights of VTC, geometric anchoring, and candidate modules. It is used in scenes with substantial texture degradation where uniform constraint strength is insufficient [S2, unofficial].

**Definition:**
SCC computes a scalar $s$ based on low-texture ratio, boundary ratio, and color contrast [S2, unofficial].

**Formula:**
$$ s = \text{clip}(s_{min}, s_{max}, s_{min} + (s_{max}-s_{min}) \cdot (a_1 \hat{r}_{lt} + a_2 \hat{r}_{bd} + a_3 \hat{c}_{fb})) $$
Where $\hat{r}_{lt}, \hat{r}_{bd}, \hat{c}_{fb}$ are normalized statistics [S2, unofficial].

**Parameters:**
*   $a_1, a_2, a_3$: Weights for low-texture ratio, boundary ratio, and color contrast. Values: $0.65, 0.20, 0.15$ [S2, unofficial].
*   $s_{min}$: Minimum constraint coefficient. Value: $0.35$ [S2, unofficial].
*   $r_{lt}$: Proportion of low-texture pixels in the foreground whose gradient magnitude is below threshold $t_{lt}$ [S2, unofficial].
*   $r_{bd}$: Ratio of foreground-mask boundary pixels to all foreground pixels [S2, unofficial].
*   $c_{fb}$: Mean color contrast between foreground and background regions [S2, unofficial].

**Usage:**
All statistics are computed only from the training split [S2, unofficial]. The degradation score of the "Drums" scene reaches the clipping upper bound, making it insensitive to $s_{min}$ variations [S2, unofficial]. Within tested ranges, PSNR span for "Lego" and "Chair" is no greater than 0.025 dB, and weight-based scan remains below 0.033 dB [S2, unofficial].

## Operating characteristics
*   **Performance:** On Mip-NeRF 360, ACO-EGR (which uses LC Score Weighting) increases multi-scale average PSNR by 2.02 dB for Mip-Splatting, 1.44 dB for AA-2DGS, and 1.96 dB for Analytic-Splatting compared to their base versions [S1].
*   **Scaling:** PSNR gains over MS-GS reached 1.96 dB at 4× scale and 2.09 dB at 128× scale on Mip-NeRF 360 [S1].
*   **Sensitivity:** The method is robust to weight tuning; moderate changes in $\lambda$ values result in negligible performance differences (< 0.10 dB) on standard benchmarks [S1].
*   **Limitations:** At scales close to training resolution, EGR may lead to small PSNR decreases at some fine scales while improving coarse-scale quality, due to reallocating Gaussians toward persistent-error regions [S1].

## Background
LC Score Weighting was introduced in the context of Adaptive-Scale Color Offset and Error-Guided Gaussian Reallocation (ACO-EGR) for Multi-Scale 3D Gaussian Splatting [S1]. The method retains three measures (mean color, max color, mean gradient) to maintain a fixed and conservative relocation criterion across datasets and rendering scales [S1].

## Sources
[S1] Adaptive-Scale Color Offset and Error-Guided Gaussian Reallocation for Multi-Scale 3D Gaussian Splatting, https://doi.org/10.3390/math14152713, fetched 2026-11-01
[S2] Visibility-Aware Texture Consensus and Local Structural Constraints for 3D Gaussian Reconstruction in Texture- Degraded Scenes, https://doi.org/10.21203/rs.3.rs-10727268/v1, fetched 2026-10-26 (unofficial)
