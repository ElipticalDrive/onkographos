# Inverse Mean Residual Color (IMRC) Metric

Also called: IMRC (Inverse Mean Residual Color)

## Overview
The Inverse Mean Residual Color (IMRC) metric evaluates the geometric quality of a radiance field without requiring ground-truth geometry, by measuring the transmittance-density-weighted residual color in decibels [S1, S2]. Unlike PSNR, which assesses overall image quality, or Chamfer Distance (CD), which requires ground-truth point clouds and suffers from information loss by considering only a single iso-surface, IMRC evaluates the entire density field and is more consistent with human user rankings for geometry [S2].

## How it works
The method computes the residual color at each voxel by comparing the observed color against a low-frequency color prior estimated via spherical harmonics (SH), weighted by the volume rendering transmittance and density [S1, S2].

**Inputs and Outputs:**
*   **Inputs:**
    *   $\mathcal{V}$: Set of voxels in the density volume grid [S1, S2].
    *   $F_v$: Density value at voxel $v$ [S1].
    *   $\delta_v$: Step size, defined as the half-width of voxel $v$ [S1].
    *   $\mathbf{c}_{obs}$: Observed color from camera views [S1, S2].
    *   $T_{v,k}$: Transmittance at voxel $v$ for direction $k$ [S1].
    *   $N_{obs}$: Number of observations (views) for a point [S2].
    *   $D_{SH}$: Degree of spherical harmonics used for color estimation [S2].
*   **Outputs:**
    *   $IMRC$: A scalar value in dB. Higher values indicate better geometry [S1, S2].

**Data Structures:**
*   **Density Volume:** A 3D grid of size $R^3$ (e.g., $512^3$) storing scalar density values $F_v$ [S2].
*   **Spherical Harmonics Coefficients:** For each voxel, coefficients $\hat{h}_{ji}$ are computed to approximate the color field [S1, S2]. The number of coefficients is $(D_{SH}+1)^2$ [S2].

**Parameters:**
*   **SH Degree ($D_{SH}$):** Controls the frequency of the color prior [S2]. Typical value is 2 [S2]. Must be smaller than the number of observations $N_{obs}$ for sparse-view reconstruction to avoid overfitting [S2].
*   **Volume Resolution:** Typical value is $512^3$ [S2]. Higher resolutions slightly improve IMRC but do not change relative method ranks [S2].

**Step-by-Step Method:**

1.  **Estimate Low-Frequency Color Field:**
    For each voxel $v$, estimate the color $\hat{\mathbf{c}}_v$ using spherical harmonics of degree $D_{SH}$ based on observed colors from all viewing directions [S1, S2]. This creates a smooth, low-frequency representation of the scene color [S2].

2.  **Compute Residual Color ($L_{\tilde{c}_{v,k}}$):**
    For each voxel $v$ and direction $k$ (representing viewing directions or integration steps), calculate the residual color [S1]. According to [S1], this is the difference between the ground truth observation color and the recovered radiance based on the geometry. The implementation must calculate the residual between the observed color $\mathbf{c}_{obs}$ and the color predicted by the low-frequency prior or the rendered color from the current density field [S1, S2].

3.  **Calculate Weighted Residual Sum:**
    Compute the numerator and denominator of the IMRC fraction [S1]. The weights are the product of transmittance $T_{v,k}$ and the emission term $(1 - \exp(-F_v \cdot \delta_v))$ [S1].

    $$
    \text{Numerator} = \sum_{v \in \mathcal{V}} \sum_{k} T_{v,k} (1 - \exp(-F_v \cdot \delta_v)) L_{\tilde{c}_{v,k}}^2
    $$

    $$
    \text{Denominator} = \sum_{v \in \mathcal{V}} \sum_{k} T_{v,k} (1 - \exp(-F_v \cdot \delta_v))
    $$

    Where:
    *   $T_{v,k}$ is the transmittance up to voxel $v$ in direction $k$ [S1].
    *   $F_v$ is the density at voxel $v$ [S1].
    *   $\delta_v$ is the half-width of the voxel [S1].
    *   $L_{\tilde{c}_{v,k}}$ is the residual color magnitude [S1].

4.  **Compute IMRC:**
    Apply the logarithmic transformation to obtain the metric in decibels [S1, Eq. 12]:

    $$
    IMRC = -10 \log_{10} \left( \frac{\text{Numerator}}{\text{Denominator}} \right)
    $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_imrc(F_v, T_v_k, L_c_tilde, delta_v):
    """
    Computes the Inverse Mean Residual Color (IMRC) metric.
    
    Args:
        F_v (np.ndarray): 3D array of density values.
        T_v_k (np.ndarray): 3D array of transmittance values.
        L_c_tilde (np.ndarray): 3D array of residual color magnitudes.
        delta_v (float): Half-width of a voxel.
        
    Returns:
        float: IMRC value in dB.
    """
    # Calculate emission weight: T_{v,k} * (1 - exp(-F_v * delta_v))
    emission_weight = T_v_k * (1.0 - np.exp(-F_v * delta_v))
    
    # Numerator: Sum of weighted squared residual color
    weighted_residual_sum = np.sum(emission_weight * (L_c_tilde ** 2))
    
    # Denominator: Sum of weights
    weight_sum = np.sum(emission_weight)
    
    # Avoid division by zero
    if weight_sum == 0:
        return 0.0
        
    # Calculate ratio
    ratio = weighted_residual_sum / weight_sum
    
    # Calculate IMRC in dB: -10 * log10(ratio)
    imrc = -10.0 * np.log10(ratio)
    
    return imrc
```

**Edge Cases and Pitfalls:**
*   **Specular Surfaces:** The metric relies on a low-frequency color prior [S2]. It fails on purely specular surfaces or mirrors where this prior does not hold [S2].
*   **Empty Density Fields:** If the density field is empty except for a few points observed by few cameras, IMRC may yield high scores incorrectly indicating good geometry [S2].
*   **Sparse Views:** The SH degree must be smaller than the number of observations to prevent the residual color from becoming indistinctly small [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:**
    *   Evaluates the entire density field, not just an iso-surface [S2].
    *   Does not require ground-truth geometry (unlike CD) [S2].
    *   More consistent with human user rankings for geometry than PSNR [S2].
    *   Stable across different SH degrees and volume resolutions regarding relative method ranks [S2].
*   **Limitations:**
    *   Fails on specular surfaces [S2].
    *   Can be misleading on sparse/empty fields [S2].
    *   Should be combined with PSNR and SSIM to reduce evaluation bias [S2].
*   **Performance Data:**
    *   On DTU dataset, SH degree 2 yields an average IMRC of 15.64, while degree 3 yields 16.28. Ranks remain stable [S2].
    *   On DTU dataset, resolution $512^3$ yields 15.64, same as $256^3$. Resolution $768^3$ yields 15.67 [S2].
    *   DVGO with distortion loss improves IMRC on DTU (18.19 to 18.47), NeRF Synthetic (17.25 to 17.94), and LLFF (18.87 to 20.76) [S2].

## Background
The IMRC metric was introduced to address the lack of proper quantitative metrics for evaluating the geometry of radiance fields without ground-truth data [S2]. It is based on the insight that better geometry should better recover the color field, leveraging a low-frequency color prior [S1, S2]. It was benchmarked against state-of-the-art methods like JaxNeRF, Plenoxels, DVGO, and TensoRF on datasets including DTU, NeRF Synthetic, and LLFF [S2].

## Subtopics

### [Temporal Instability Metric](1921-temporal-instability-metric.md)
*   **What it is:** A metric evaluating the stability of rendered images over time in dynamic scenes.
*   **Difference:** Focuses on temporal consistency rather than geometric accuracy of a static radiance field.
*   **When to choose:** For dynamic novel view synthesis or video generation tasks.

### [Across View-direction Inconsistency (AVI) Metric](11106-across-view-direction-inconsistency-avi-metric.md)
*   **What it is:** A metric measuring the inconsistency of rendered colors across different view directions for the same 3D point.
*   **Difference:** Evaluates view-dependent artifacts rather than the global geometric quality via color residual.
*   **When to choose:** When assessing the quality of view-dependent appearance models.

## Sources
[S1] Reducing Shape-Radiance Ambiguity in Radiance Fields with a Closed-Form Color Estimation Method, https://doi.org/10.52202/075280-2570, fetched 2026-10-26
[S2] Evaluate Geometry of Radiance Fields with Low-Frequency Color Prior, https://doi.org/10.1609/aaai.v38i2.27938, fetched 2026-10-26
