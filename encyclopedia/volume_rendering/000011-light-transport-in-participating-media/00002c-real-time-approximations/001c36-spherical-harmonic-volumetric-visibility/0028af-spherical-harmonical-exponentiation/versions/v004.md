# Spherical Harmonical Exponentiation

Also called: Ren et al. Spherical Harmonical Exponentiation

## Overview
Spherical Harmonical Exponentiation is a technique for encoding directional volumetric visibility using spherical harmonic (SH) basis functions [S1, unofficial]. It solves the problem of real-time dynamic directional occlusion by representing the visibility function as a vector of SH coefficients, allowing for efficient combination of visibility contributions [S1, unofficial]. Unlike [Spherical Harmonics Irradiance Encoding](4336-spherical-harmonics-irradiance-encoding.md) which typically encodes incoming light, this method encodes directional occlusion in the frequency domain [S1, unofficial].

## How it works
**Inputs:**
- A set of $K$ posed camera observations $\{o_k, d_k, c_{v,d_k}\}_{k=1}^K$, where $o_k$ is the camera center, $d_k$ is the normalized direction vector pointing from the camera to the target point $v$, and $c_{v,d_k}$ is the observed color at $v$ [S2].
- A density field $\sigma$ along the ray from each camera center $o_k$ to the target point $v$ [S2].

**Outputs:**
- Spherical harmonic coefficients $\hat{h}_\ell^m$ representing the directional occlusion at point $v$ [S2].

**Data Structures:**
- SH Coefficients: A vector of size $(L+1)^2$ for order $L$, storing real-valued coefficients $\hat{h}_\ell^m$ where $0 \le \ell \le L$ and $-\ell \le m \le \ell$ [S1, unofficial][S2].

**Steps:**
1. **Calculate Transmittance:** For each observation $k$, compute the transmittance $T_{v,k}$ from camera center $o_k$ to target point $v$ along the ray $r(s; v, d_k) = v + s d_k$ [S2].
   $$ T_{v,k} = \exp \left( - \int_0^{\|o_k - v\|} \sigma(r(s; v, d_k)) \, ds \right) $$
   [S2]
2. **Estimate SH Coefficients:** Compute the coefficients $\hat{h}_\ell^m$ using transmittance-weighted Monte Carlo integration over the observed directions [S2]. The probability density function is assumed uniform over the sphere: $p_d(d) \equiv \frac{1}{4\pi}$ [S2].
   $$ \hat{h}_\ell^m = \frac{\sum_{k=1}^K T_{v,k} c_{v,d_k} Y_\ell^m(d_k)}{\sum_{k=1}^K T_{v,k}} $$
   [S2]
3. **Handle Non-Uniform Views:** If views are not uniformly distributed (e.g., half-sphere coverage), the orthogonality of SH bases is violated, causing bias [S2]. A residual estimation step is required to correct for this non-uniformity [S2].

**Parameters:**
- **SH Order ($L$):** Determines the precision of the visibility encoding [S1, unofficial].

**Pitfalls:**
- **Occlusion Blur:** Without transmittance weighting, observations from cameras where $v$ is occluded by nearer surfaces contribute incorrect color data, blurring the SH estimation [S2].
- **View Distribution Bias:** Non-uniformly distributed views lead to biased SH coefficient estimation due to the loss of orthogonality [S2].

## Subtopics

### Transmittance-weighted SH Estimation
A weighting scheme that uses transmittance to exclude occluded observations when estimating SH coefficients [S2]. It prevents blurred results caused by surface points nearer to the camera than the target point contributing incorrect color data [S2]. It assumes a uniform probability density function $p_d(d) \equiv \frac{1}{4\pi}$ for direction sampling [S2].

## Operating characteristics
- **Complexity:** Depends on the number of cameras $K$ and the SH order $L$. Each coefficient requires a sum over $K$ observations [S2].
- **Strengths:** Provides directional occlusion information efficiently, suitable for real-time rendering [S1, unofficial].
- **Limitations:** Sensitive to non-uniform view distributions without residual correction [S2]. Requires accurate density/transmittance calculation [S2].

## Background
The method was introduced by Ren et al. [S1, unofficial]. It is used in the context of dynamic directional occlusion [S1, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] Reducing Shape-Radiance Ambiguity in Radiance Fields with a Closed-Form Color Estimation Method, https://doi.org/10.52202/075280-2570, fetched 2026-10-26
