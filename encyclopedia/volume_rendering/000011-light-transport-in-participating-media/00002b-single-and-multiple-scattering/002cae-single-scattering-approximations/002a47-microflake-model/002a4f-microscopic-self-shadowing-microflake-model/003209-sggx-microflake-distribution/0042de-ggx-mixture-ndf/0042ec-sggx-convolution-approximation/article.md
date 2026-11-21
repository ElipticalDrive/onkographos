# SGGX Convolution Approximation

Also called: Convolution of SGGX and Isotropic Distribution, Roughened SGGX

## Overview
The SGGX Convolution Approximation is a method for approximating the convolution of a Single Scattering GGX (SGGX) normal distribution function (NDF) with an isotropic spherical distribution $g(\omega; n)$ by another SGGX distribution with increased roughness [S1, unofficial]. It solves the problem of efficiently modeling surface roughness or scattering anisotropy in participating media and microflake models without resorting to complex analytical integrals. Unlike the [Convolution of Aggregated Microfacet and Surface NDF](10871-convolution-of-aggregated-microfacet-and-surface-ndf.md), which handles aggregated microfacets, this method specifically targets the convolution of a single SGGX lobe with an isotropic blur, allowing for a compact, scene-independent parameterization [S1, unofficial].

## How it works
**Inputs:**
1.  An SGGX distribution parameterized by an eigenbasis $R$ and anisotropic roughness $\alpha = (\alpha_x, \alpha_y)$ [S1, unofficial].
2.  An isotropic spherical distribution $g(\omega; n)$ [S1, unofficial].

**Outputs:**
1.  A roughened SGGX distribution $D_{sggx}(\omega; R, \alpha^+)$ that approximates the convolution [S1, unofficial].
2.  The mapping function $M$ that computes the new roughness $\alpha^+$ from $\alpha$ and $g$ [S1, unofficial].

**Data Structures:**
*   **SGGX Parameters:** The SGGX distribution is defined by a $3 \times 3$ symmetric positive definite matrix $S$, which is decomposed into an eigenbasis $R = (\omega_1, \omega_2, \omega_3)$ and roughness values $\alpha_x, \alpha_y$ [S1, unofficial]. The exact parameterization of $R$ and $\alpha$ from $S$ is detailed in the source's supplemental document [S1, unofficial].
*   **Precomputed Table:** The mapping $\alpha^+ = M(\alpha, g)$ is stored in a precomputed table. Since the mapping is scene-independent and typically smooth, it requires negligible storage and is computed once [S1, unofficial].

**Steps:**
1.  **Parameterize:** Represent the input SGGX lobe $D_{sggx}(n; R, \alpha)$ using its eigenbasis $R$ and anisotropic roughness $\alpha$ [S1, unofficial].
2.  **Fit:** Perform a nonlinear least-square fit to find the mapping $\alpha^+ = M(\alpha, g)$ such that the convolution is approximated by the roughened SGGX [S1, unofficial]:
    $$ \int_{S^2} g(\omega; n) D_{sggx}(n; R, \alpha) \, dn \approx D_{sggx}(\omega; R, \alpha^+) $$
    where $R$ remains fixed and $\alpha$ gains additional values to become $\alpha^+$ [S1, unofficial].
3.  **Store:** Save the mapping $M$ in a lookup table for runtime use [S1, unofficial].

**Formulas:**
The convolution of the mixture NDF $p_N(n) = \sum_{i=1}^k w_i D_{i,sggx}(n)$ with $g(\omega; n)$ is approximated by summing the roughened lobes [S1, unofficial]:
$$ D_{conv}(\omega) = \int_{S^2} g(\omega; n) p_N(n) \, dn \approx \sum_{i=1}^k w_i D_{i,sggx}(\omega; R_i, \alpha^+_i) $$
where $\alpha^+_i = M(\alpha_i, g)$ [S1, unofficial].

**Parameters:**
*   $R$: Eigenbasis of the SGGX distribution [S1, unofficial].
*   $\alpha$: Anisotropic roughness $(\alpha_x, \alpha_y)$ [S1, unofficial].
*   $g$: Isotropic spherical distribution [S1, unofficial].

**Edge Cases and Pitfalls:**
*   The approximation relies on the smoothness of the mapping $M$. If $g$ is highly non-smooth, the approximation may degrade [S1, unofficial].
*   The method assumes the convolution can be represented by a single SGGX lobe with modified roughness. This is an approximation and may not capture all features of the true convolution, particularly for highly anisotropic cases where multi-lobe representations are preferred [S2].

## Subtopics

### Multi-lobe SGGX Phase Function
A representation that combines single-scattering albedo and phase functions into a single entity to facilitate downsampling, addressing the issue that naive downsampling weakens intrinsic shadowing structures [S2]. It represents the scaled phase function $\hat{f}$ as a linear combination of $m$ SGGX lobes, allowing for better approximation of complex scattering than a single lobe [S2]. The scaled phase function is defined as:
$$ \hat{f}(\omega_0 \rightarrow \omega) = \sum_{j=1}^{m} W_j f_j(\omega_0 \rightarrow \omega) $$
where $f_j$ is the $j$-th SGGX lobe and $W_j \in (0, 1]$ is the lobe weight satisfying $\sum_{j=1}^{m} W_j \le 1$ [S2]. The number of lobes $m$ is typically limited to 3 for a balance between model size and accuracy [S2]. Using a single SGGX function ($m=1$) via averaging works adequately for many cases but leads to limited accuracy for highly anisotropic materials [S2].

## Operating characteristics
*   **Time Complexity:** The fitting process is performed offline. At runtime, the method requires a table lookup for the mapping $M$, which is $O(1)$ [S1, unofficial].
*   **Memory Complexity:** The precomputed table for the mapping $M$ requires negligible storage [S1, unofficial].
*   **Strengths:** Scene-independent, compact representation, efficient runtime performance [S1, unofficial].
*   **Limitations:** Approximation error depends on the isotropic distribution $g$ and the anisotropy of the material. Single-lobe approximation may be insufficient for highly anisotropic materials [S2].
*   **When to use:** When modeling surface roughness or scattering anisotropy in a compact way, especially in the context of [GGX Mixture NDF](10859-ggx-mixture-ndf.md) or downsampling anisotropic media [S1, unofficial][S2].
*   **When not to use:** When high accuracy is required for highly anisotropic materials, a multi-lobe representation should be considered [S2].

## Background
The SGGX distribution was introduced by Heitz et al. in 2015 as a compact representation for the surface NDF [S1, unofficial]. The convolution approximation method was proposed in the context of efficient scene appearance aggregation for level-of-detail rendering, building on the idea of approximating convolutions of anisotropic distributions with roughened versions [S1, unofficial]. The multi-lobe extension was introduced in the context of downsampling scattering parameters for rendering anisotropic media to preserve shadowing structures [S2].

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
[S2] Downsampling scattering parameters for rendering anisotropic media, https://doi.org/10.1145/2980179.2980228, fetched 2026-10-28
