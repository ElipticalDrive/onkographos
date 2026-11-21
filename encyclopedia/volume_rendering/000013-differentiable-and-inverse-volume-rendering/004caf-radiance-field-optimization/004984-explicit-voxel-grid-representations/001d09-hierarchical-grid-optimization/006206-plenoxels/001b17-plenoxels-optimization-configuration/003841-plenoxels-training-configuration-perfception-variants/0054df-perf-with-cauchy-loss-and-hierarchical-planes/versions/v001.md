# PERF with Cauchy Loss and Hierarchical Planes

Also called: Implementation details for PERF additions, PERF additions, PERF with difference planes and Cauchy loss

## Overview
This method extends the PERF (Plenoxels Optimization Framework) algorithm by introducing difference planes and a Cauchy loss function to mitigate shape-radiance ambiguity in view-dependent radiance fields [S1, unofficial]. It sets itself apart from standard PERF by explicitly separating view-independent density from view-dependent appearance using hierarchical difference planes, while using a robust loss function to handle outliers and sparse real-world data [S1, unofficial].

## How it works
The method operates within a non-linear least-squares optimization framework. It augments the standard Plenoxels representation with additional "difference planes" and modifies the loss function.

**Inputs and Outputs:**
*   **Inputs:** Camera poses and intrinsics (calibrated via COLMAP [S1, unofficial]), input images, initial Plenoxels grid.
*   **Outputs:** Optimized Plenoxels grid (density and base color), difference planes (view-dependent color residuals), and per-voxel weights for the loss function.

**Data Structures:**
1.  **Plenoxels Grid:** Standard 3D voxel grid storing density and base color coefficients.
2.  **Difference Planes:** A set of 2D planes (feature maps) corresponding to the hierarchical levels. These store view-dependent color residuals.
3.  **Weight Buffer:** A buffer storing one float per voxel, representing the per-voxel weight for the Cauchy loss.

**Hierarchical Structure:**
The method uses four hierarchical levels. The highest level has a resolution of $256^3$ [S1, unofficial]. The resolution of the added difference planes follows camera resolutions through hierarchical steps, approximately matching the voxel resolution at any given level [S1, unofficial]. Values for each new hierarchical level are upsampled using linear interpolation from the previous level [S1, unofficial].

**Formulas:**
1.  **Difference Planes (Equation 4 in [S1, unofficial]):** These planes capture the residual view-dependent color information not captured by the base Plenoxels. The exact formulation of Equation 4 is not provided in the source text, but it is added to the rendering pipeline [S1, unofficial].
2.  **Cauchy Loss with Per-Voxel Weight (Equation 7 in [S1, unofficial]):**
    $$ L_c = \sum_{i} w_i \cdot \log\left(1 + \frac{(I_{pred} - I_{gt})^2}{\epsilon}\right) $$
    Where $w_i$ is the per-voxel weight, $I_{pred}$ is the predicted pixel color, $I_{gt}$ is the ground truth pixel color, and $\epsilon$ is a small constant. The source mentions a constant $w = 0.0001$ is added to the baseline case for fair comparison [S1, unofficial]. The exact value of $\epsilon$ is not specified in the provided text.
3.  **Visibility Loss (Equation 8 in [S1, unofficial]):**
    $$ L_s = \lambda_s (-4(T - 0.5)^2 + 1) $$
    Where $T$ is the total visibility for each ray, and $\lambda_s = 0.1$ [S1, unofficial]. This term helps mitigate smoke-like artifacts and keeps real scenes sparse [S1, unofficial].

**Parameters:**
*   **Hierarchical Levels:** 4 [S1, unofficial].
*   **Highest Level Resolution:** $256^3$ [S1, unofficial].
*   **Visibility Loss Weight ($\lambda_s$):** 0.1 [S1, unofficial].
*   **Baseline Cauchy Loss Weight:** 0.0001 (used for fair comparison with baseline PERF) [S1, unofficial].

**Algorithm Steps:**
1.  **Calibration:** Calibrate scenes using COLMAP [S1, unofficial].
2.  **Initialization:** Initialize the Plenoxels grid and difference planes.
3.  **Hierarchical Optimization:**
    a.  For each hierarchical level (starting from the second level, as the first is initialized differently):
        i.  Run Algorithm 1 and Equation 6 from PERF once at the beginning of the level [S1, unofficial].
        ii. Compute per-voxel weights and store them in a buffer [S1, unofficial].
        iii. Upsample difference planes from the previous level using linear interpolation [S1, unofficial].
    b.  In each iteration of the optimization:
        i.  Evaluate the Cauchy loss (Equation 7) using the pre-computed per-voxel weights [S1, unofficial].
        ii. Evaluate the visibility loss (Equation 8) [S1, unofficial].
        iii. Update the Plenoxels grid and difference planes using non-linear least-squares optimization.

**Edge Cases and Pitfalls:**
*   **Real Scenes:** Real scenes have more camera poses than artificial scenes, leading to an extra overhead of about 20-30% in reconstruction time [S1, unofficial].
*   **Baseline Comparison:** To ensure a fair comparison with baseline PERF, the Cauchy loss must be added with a constant weight $w = 0.0001$ to the baseline case [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method generally converges faster than baseline PERF for artificial scenes, resulting in shorter reconstruction times. For real scenes, there is an additional overhead of 20-30% due to the extra computations [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Reconstruction times on an Nvidia GTX 1080:
    *   Artificial scenes (w/ planes vs w/o planes):
        *   Lego: 154s vs 194s
        *   Ficus: 107s vs 208s
        *   Hotdog: 267s vs 355s
        *   Drums: 121s vs 273s
        *   Chair: 90s vs 229s
    *   Real scenes (w/ planes vs w/o planes):
        *   Material: 243s vs 478s
        *   Ignatius: 228s vs 172s
        *   Family: 154s vs 131s
    *   Note: The text states that for artificial scenes, performance is higher due to faster convergence. For real scenes, the extra overhead is 20-30%. The table values for real scenes show mixed results (Material is faster, Ignatius and Family are slower), which aligns with the stated overhead range [S1, unofficial].
*   **Strengths:** Mitigates shape-radiance ambiguity, separates view-dependent information effectively, reduces smoke-like artifacts.
*   **Limitations:** Additional overhead for real scenes, requires careful calibration (COLMAP).

## Background
This method was introduced in the paper "Addressing the Shape-Radiance Ambiguity in View-Dependent Radiance Fields" [S1, unofficial]. It builds upon the PERF (Plenoxels Optimization Framework) algorithm [S1, unofficial]. The work aims to improve the quality of view-dependent radiance field reconstruction by explicitly handling the ambiguity between shape and appearance.

## Sources
[S1] Addressing the Shape-Radiance Ambiguity in View-Dependent Radiance Fields, https://doi.org/10.48550/arxiv.2203.01553, fetched 2026-10-29 (unofficial)
