# RDLS-SS-DWT+Pred

Also called: Hybrid Transform, Reversible Denoising Lifting Steps with Step Skipping and Prediction

## Overview
RDLS-SS-DWT+Pred is a lossless volumetric compression transform that combines the 3D Discrete Wavelet Transform (DWT) with Reversible Denoising Lifting Steps (RDLS) and step skipping, followed by predictive coding on the final subbands [S1]. It solves the problem of high entropy in wavelet coefficients by applying spatial prediction to the transformed data, similar to techniques used in 2D image compression [S1]. It differs from its sibling [SS-DWT+Pred](5256-residual-updates-in-transform-domain.md) by incorporating denoising filters in the lifting steps, though sources indicate that using actual denoising filters may be avoided without sacrificing bitrate improvements, making it functionally similar to [SS-DWT+Pred](5256-residual-updates-in-transform-domain.md) in practice [S1].

## How it works
The method constructs a hybrid transform by applying prediction to the final subbands produced by RDLS-SS-DWT [S1].

**Inputs and Outputs:**
*   Input: Volumetric data $V$ of size $W \times H \times D$.
*   Output: Entropy-coded residuals from the final subbands.

**Data Structures:**
*   The volume is decomposed into subbands using a 3-level DWT decomposition [S1].
*   Candidate predictors are defined by their spatial dependencies (see Table 1 in [S1] for specific neighbor configurations).

**Algorithm Steps:**

1.  **Transform Application:** Apply RDLS-SS-DWT to the input volume $V$. This involves selecting RDLS filters and steps to skip based on a heuristic (see [Adaptive Transform Construction Heuristic](11258-adaptive-transform-construction-heuristic.md)) [S1].
2.  **Predictor Selection:** For each final subband of the RDLS-SS-DWT output, select a candidate predictor from a predefined set that results in the smallest memoryless entropy of prediction errors [S1].
3.  **Entropy Estimation:** Compute the memoryless entropy $H$ for each candidate predictor to determine the best one. The entropy is calculated as:
    $$ H = -\sum_{i=\text{MinPE}}^{\text{MaxPE}} p_i \log_2 p_i $$
    where $M$ is the number of samples in the subband, $\text{MinPE}$ and $\text{MaxPE}$ are the smallest and greatest prediction errors respectively, $p_i$ is the probability of occurrence of prediction error $i$, and $0 \log_2 0 = 0$ [S1].
4.  **Prediction and Residual Calculation:** Apply the selected predictor to the subband samples in raster scan order: volume slices from front to back, slice rows from top to bottom, and pixels in a row from left to right [S1].
    *   If a predictor cannot be computed for a specific pixel (e.g., due to boundary conditions), fall back to the NOP predictor [S1].
    *   The NOP predictor predicts that each sample is 0 [S1].
    *   Calculate the residual $d = v - \hat{v}$, where $v$ is the actual value and $\hat{v}$ is the predicted value [S1].
5.  **Entropy Encoding:** Encode the residuals using entropy coding (e.g., JP3D) [S1].

**Candidate Predictors:**
The set of candidate predictors includes:
*   **NOP:** Predicts 0.
*   **P_X, P_Y, P_Z:** Predictors based on axial, horizontal, and vertical neighbors (specific configurations defined in [S1]).
*   **AVG_XY, AVG_XZ, AVG_YZ:** Averages of specific neighbor pairs.
*   **MED_XY, MED_XZ, MED_YZ:** Medians of specific neighbor triplets.
*   **AVG3D:** Average of three neighbors.
*   **Linear/Predictive:** Combinations such as $(A+B)/2$, $(A+D)/2$, $(B+D)/2$, and median-based predictors like $\text{median}(A, B, A+B-C)$ [S1].

**Parameters:**
*   Decomposition level: 3 levels [S1].
*   NOP predictor: Used as a fallback or if actual predictors increase entropy [S1].

**Pitfalls:**
*   Boundary conditions: Predictors requiring neighbors outside the volume bounds must be handled by falling back to simpler predictors like NOP [S1].
*   Heuristic cost: Selecting filters based on actual JP3D bitrates is computationally expensive; entropy estimation (H0) is a faster alternative [S1].

## Subtopics

### [Adaptive Transform Construction Heuristic](11258-adaptive-transform-construction-heuristic.md)
A greedy parameter selection method for the RDLS-SS-DWT transform that selects denoising filters and steps to skip to minimize estimated bitrate. It uses fast entropy estimation to approximate JP3D bitrate, allowing image-adaptive configuration without prohibitive computational cost.

## Operating characteristics
*   **Time Complexity:** The heuristic selection of filters and steps can be costly if based on actual JP3D bitrates. Using entropy estimation reduces this cost [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The method yields significant improvements in lossless JPEG 2000 (JP3D) compression ratios, particularly for MRI images [S1]. For example, on the MRI1 image, RDLS-SS-DWT+Pred achieved an 8.973% bitrate reduction compared to standard DWT [S1].
*   **Comparison:** The difference in bitrate improvement between RDLS-SS-DWT+Pred and SS-DWT+Pred is very small, suggesting that the computational cost of actual denoising filters in RDLS may be avoided without sacrificing performance [S1].
*   **Strengths:** Effective for volumetric medical images; combines the benefits of wavelet transform and prediction [S1].
*   **Limitations:** Heuristic selection can be slow if not using entropy estimation; boundary handling requires fallback mechanisms [S1].

## Background
The method was proposed in 2020 by researchers from Vrije Universiteit Brussel (VUB) and Interdisciplinary Institute for BroadBand Technology (IBBT) [S1]. It builds on previous work generalizing 2D-DWT variants to 3D using step skipping and reversible denoising lifting steps [S1]. The implementation is available as a patch to IRIS-JP3D [S1].

## Sources
[S1] Employing New Hybrid Adaptive Wavelet-Based Transform and Histogram Packing to Improve JP3D Compression of Volumetric Medical Images, https://doi.org/10.3390/e22121385, fetched 2026-10-26
