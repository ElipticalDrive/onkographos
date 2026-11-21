# Spherical Harmonical Exponentiation

Also called: Ren et al. Spherical Harmonical Exponentiation

## Overview
Spherical Harmonical Exponentiation is a technique for encoding directional volumetric visibility using spherical harmonic (SH) basis functions [S1, unofficial]. It solves the problem of real-time dynamic directional occlusion by representing the visibility function as a vector of SH coefficients, allowing for efficient combination of visibility contributions [S1, unofficial]. Unlike [Spherical Harmonics Irradiance Encoding](4336-spherical-harmonics-irradiance-encoding.md) which typically encodes incoming light, this method encodes the *absence* of light (occlusion) in the frequency domain [S1, unofficial].

## How it works
**Inputs:**
- A set of $K$ posed camera observations.
- For each camera $k$, the camera center $o_k$, the normalized direction vector $d_k$ pointing from the camera to the target point $v$, and the observed color $c_{v,d_k}$ [S2].
- The density field $\sigma$ along the ray from $o_k$ to $v$ [S2].

**Outputs:**
- A set of spherical harmonic coefficients $\hat{h}_\ell^m$ representing the directional occlusion at point $v$ [S2].

**Data Structures:**
- SH Coefficients: A vector of size $(L+1)^2$ for order $L$, storing real-valued coefficients $\hat{h}_\ell^m$ where $0 \le \ell \le L$ and $-\ell \le m \le \ell$ [S1, unofficial][S2].

**Steps:**
1. **Calculate Transmittance:** For each observation $k$, compute the transmittance $T_{v,k}$ from the camera center $o_k$ to the target point $v$ along the ray $r(s; v, d_k) = v + s d_k$ [S2].
   $$ T_{v,k} = \exp \left( - \int_0^{\|o_k - v\|} \sigma(r(s; v, d_k)) \, ds \right) $$
   [S2]

2. **Estimate SH Coefficients:** Compute the coefficients $\hat{h}_\ell^m$ using transmittance-weighted Monte Carlo integration over the observed directions [S2]. The probability density function $p_d(d)$ is assumed uniform over the sphere: $p_d(d) \equiv \frac{1}{4\pi}$ [S2].

   $$ \hat{h}_\ell^m = \frac{\sum_{k=1}^K T_{v,k} c_{v,d_k} Y_\ell^m(d_k)}{\sum_{k=1}^K T_{v,k}} $$
   [S2]

   *Note: The source [S2] presents Eq. (7) with a denominator $\sum T_{v,k}$ and numerator terms involving $c_{v,d_k}$. The standard SH projection usually involves dividing by the integral of the basis squared or the PDF. The source text implies a weighted average where the weights are transmittance. The formula in [S2] Eq. (7) is:*
   $$ \hat{h}_\ell^m = \frac{\sum_{k=1}^K T_{v,k} c_{v,d_k} Y_\ell^m(d_k)}{\sum_{k=1}^K T_{v,k}} $$
   *However, Eq. (5) in [S2] shows the integral approximation as $\sum \frac{c F Y}{p_d}$. Eq. (7) modifies this with transmittance. The exact denominator in Eq. (7) of [S2] is written as $\sum_{k=1}^K T_{v,k}$. This acts as a normalization for the weighted sum.*

3. **Handle Non-Uniform Views (Residual):** If views are not uniformly distributed (e.g., half-sphere coverage), the orthogonality of SH bases is violated, causing bias. A residual estimation step is required to correct for this non-uniformity [S2].

**Parameters:**
- **SH Order ($L$):** Determines the precision of the visibility encoding. Higher $L$ provides more precise directional occlusion but increases computational cost [S1, unofficial].

**Pitfalls:**
- **Occlusion Blur:** Without transmittance weighting, observations from cameras where the target point $v$ is occluded by nearer surfaces will contribute incorrect color data, blurring the SH estimation [S2].
- **View Distribution Bias:** Non-uniformly distributed views (e.g., limited camera angles) lead to biased SH coefficient estimation due to the loss of orthogonality [S2].

## Subtopics

### Transmittance-weighted SH Estimation
A weighting scheme that uses transmittance to exclude occluded observations when estimating SH coefficients [S2]. It prevents blurred results caused by surface points nearer to the camera than the target point contributing incorrect color data [S2]. It assumes a uniform probability density function $p_d(d) \equiv \frac{1}{4\pi}$ for direction sampling [S2].

## Operating characteristics
- **Complexity:** Depends on the number of cameras $K$ and the SH order $L$. Each coefficient requires a sum over $K$ observations.
- **Strengths:** Provides directional occlusion information efficiently, suitable for real-time rendering [S1, unofficial].
- **Limitations:** Sensitive to non-uniform view distributions without residual correction [S2]. Requires accurate density/transmittance calculation [S2].

## Background
The method was introduced by Ren et al. [S1, unofficial]. It is used in the context of real-time ray-traced voxel global illumination and dynamic directional occlusion [S1, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] Reducing Shape-Radiance Ambiguity in Radiance Fields with a Closed-Form Color Estimation Method, https://doi.org/10.52202/075280-2570, fetched 2026-10-26
