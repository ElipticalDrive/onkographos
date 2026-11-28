# Spherical Harmonics Degree Selection

Also called: SH basis selection

## Overview
Spherical Harmonics Degree Selection determines the maximum bandwidth $L$ of the SH basis used to represent directional signals (such as radiance or color) in volumetric rendering [G]. It balances reconstruction fidelity against memory consumption, as higher degrees increase the number of coefficients quadratically [S1]. It differs from [Spherical Harmonics Approximation](8741-spherical-harmonics-approximation.md) by focusing on the capacity limit (bandwidth) rather than the projection or fitting process itself [G].

## How it works
**Inputs:**
- Target maximum degree $L_{\text{target}}$ (integer $\ge 0$) [G].
- Source directional coefficients, potentially at a higher degree $L_{\text{source}}$ [G].

**Outputs:**
- A coefficient vector truncated or padded to degree $L_{\text{target}}$ [G].
- Metadata indicating the effective bandwidth [G].

**Data Structures:**
- Coefficients are stored as a flat vector of size $N = (L+1)^2$ for a given degree $L$ [Parent].
- For real-valued rendering, coefficients correspond to the real basis functions $Y_{lm}$ [Parent].
- The linear index $i$ maps degree $l$ and order $m$ via a standard scheme [Parent].

**Steps:**
1. **Calculate Capacity:** Compute the required coefficient count for the target degree: $N_{\text{target}} = (L_{\text{target}} + 1)^2$ [G].
2. **Check Compatibility:** Compare $L_{\text{source}}$ with $L_{\text{target}}$ [G].
3. **Pad (if necessary):** If $L_{\text{source}} < L_{\text{target}}$, extend the coefficient vector with zeros up to index $N_{\text{target}} - 1$ [G].
4. **Truncate (if necessary):** If $L_{\text{source}} \ge L_{\text{target}}$, discard all coefficients corresponding to degrees $l > L_{\text{target}}$ by keeping only the first $N_{\text{target}}$ elements of the flat vector [G].
5. **Validate:** Ensure $L_{\text{target}} \ge 0$ [G]. If $L_{\text{target}} = 0$, retain only the DC component ($l=0, m=0$) [G].

**Parameters:**
- $L_{\text{target}}$: Not given in the sources.
- $L_{\text{source}}$: Determined by the data generation method or training configuration [G].

**Edge Cases and Pitfalls:**
- **Memory Scaling:** Memory usage scales as $O(L^2)$ [G]. Higher degrees require significantly more memory [S1].
- **Precision:** Truncation introduces approximation error. SH functions exhibit 'ringing' artifacts (high-frequency noise) near sharp value changes when many coefficients are used [G].
- **Boundary:** $L=0$ represents a constant color field with no directional variation [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_sh_degree(coefficients, l_source, l_target):
    """
    Selects or truncates SH coefficients to a target degree.
    Operates on a flat coefficient array of size (L+1)^2.
    """
    # Calculate required size for target degree
    n_target = (l_target + 1) ** 2
    
    # If source is smaller, pad with zeros
    if l_source < l_target:
        current_size = (l_source + 1) ** 2
        padded_coeffs = list(coefficients) + [0.0] * (n_target - current_size)
        return padded_coeffs
    
    # If source is larger or equal, truncate
    else:
        return list(coefficients[:n_target])
```

## Subtopics

### [SH Degree Truncation](17054-sh-degree-truncation.md)
SH Degree Truncation is a preprocessing step that reduces the spherical harmonics (SH) degree of loaded splats to match a target parameter, thereby lowering memory and compute requirements. It differs from dynamic degree selection methods by performing a static, in-place reduction of the SH coefficients at load time rather than during rendering or training.

### Spherical Harmonic Irradiance Gradients
Spherical Harmonic Irradiance Gradients store the rate at which irradiance changes with respect to translations about a sample point, allowing extrapolation of irradiance to nearby points to reduce error from moving away from the sample location [S2, unofficial]. The technique projects 6 additional irradiance functions into spherical harmonics at small offsets in the +Y, -Y, +X, -X, +Z, and -Z directions relative to the original sample [S2, unofficial]. For each SH coefficient, partial derivatives are computed via central differencing by subtracting the coefficient from the offset sample in the negative direction from the positive direction sample, then dividing by the distance between samples [S2, unofficial]. This yields a 3D gradient vector for each coefficient. At render time, the gradient extrapolates a new irradiance function using $I_i' = I_i + (\nabla I_i \cdot d)$, where $I_i'$ is the $i$-th SH coefficient of the extrapolated irradiance, $I_i$ is the stored sample's coefficient, $\nabla I_i$ is the irradiance gradient for that coefficient, and $d$ is the non-unit vector from the original sample location to the point being rendered [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Selection is $O(N)$ for copying or truncating the coefficient array, where $N = (L+1)^2$ [G]. Computational cost for rendering is not obviously affected by SH degree [S1].
- **Memory Complexity:** $O(L^2)$ [G]. Each coefficient typically requires 3 floats (RGB) or 4 floats (RGBA) [G].
- **Performance:** PSNR generally increases with SH degree. On a 256^3 grid, PSNR rises from 26.13 (Degree 0) to 27.22 (Degree 1), 26.99 (Degree 2), 28.29 (Degree 3), and 27.89 (Degree 4) [S1]. On a 512^3 grid, it rises from 26.99 (Degree 0) to 28.29 (Degree 1), 29.44 (Degree 2), 28.30 (Degree 3), and 29.98 (Degree 4) [S1].
- **Trade-offs:** Higher degrees improve visual fidelity but significantly increase memory usage [S1].
- **When to use:** Use low degrees for memory-constrained environments; use higher degrees for high-fidelity rendering where directional detail is critical [G].

## Background
The concept of limiting spherical harmonic expansions to a finite bandwidth is fundamental to signal processing on the sphere [G]. In volumetric rendering, specifically Direct Volume Ray Casting and Neural Radiance Fields, SH degree selection became a critical hyperparameter for balancing quality and resource usage [G]. Empirical studies on DVGO (Direct Voxel Grid Optimization) demonstrated the quantitative impact of degree selection on PSNR [S1].

## Sources
[S1] Reducing Shape-Radiance Ambiguity in Radiance Fields with a Closed-Form Color Estimation Method, https://doi.org/10.52202/075280-2570, fetched 2026-10-26
[S2] PDF Irradiance Volumes for Games, https://chrisoat.com/papers/Oat_GDC2005_IrradianceVolumesForGames.pdf, fetched 2026-10-28 (unofficial)
