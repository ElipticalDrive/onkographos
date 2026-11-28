# Digital Hologram Numerical Reconstruction

Also called: ASM, Angular Spectrum Method, Fresnel Method

## Overview
Digital Hologram Numerical Reconstruction computes the complex wavefield at a target plane from a recorded or computed hologram matrix, solving the inverse problem of optical propagation [S1]. It differs from sibling methods by focusing on the mathematical inversion of the diffraction integral (via Angular Spectrum or Fresnel approximations) rather than the physical setup [13993-holographic-reconstruction-system-setup.md] or the generation of the hologram itself [Parent](6884-holographic-computer-generated-holography-cgh-calculation.md).

## Shared principles
The core mechanism propagates a complex field $U_h(x,y)$ defined on the hologram plane to a reconstruction plane at distance $z$. The propagation is modeled in the spatial frequency domain. Let $U_h(u,v)$ be the 2D Fourier Transform of the hologram field, where $u,v$ are spatial frequencies. The transfer function $H(u,v,z)$ depends on the chosen approximation:

1.  **Angular Spectrum Method (ASM):** Exact for scalar diffraction, valid for near-field.
    $$ H_{ASM}(u,v,z) = \exp\left(i 2\pi z \sqrt{\frac{1}{\lambda^2} - u^2 - v^2}\right) $$
    Valid when $\frac{1}{\lambda^2} - u^2 - v^2 \ge 0$. Otherwise, the term is evanescent and decays.

2.  **Fresnel Method:** Approximation for far-field or paraxial regimes.
    $$ H_{Fresnel}(u,v,z) = \exp\left(i \frac{2\pi z}{\lambda}\right) \exp\left(-i \pi \lambda z (u^2 + v^2)\right) $$

The reconstructed field $U_z(x,y)$ is obtained by:
$$ U_z(x,y) = \mathcal{F}^{-1} \left\{ \mathcal{F}\{U_h(x,y)\} \cdot H(u,v,z) \right\} $$

Key shared constraints include:
1.  **Sampling Requirements:** The pixel pitch $\Delta x$ and wavelength $\lambda$ must satisfy the Nyquist criterion for the highest spatial frequency in the transfer function to avoid aliasing [G].
2.  **Complexity:** The FFT-based approach scales as $O(N^2 \log N)$ for an $N \times N$ hologram, significantly faster than direct summation [G].

## Subtopics

### [Holographic Reconstruction System Setup](13993-holographic-reconstruction-system-setup.md)
Describes the physical apparatus (SLM, lenses, cameras) required to convert the numerically reconstructed complex wavefield into a visible optical image [13993-holographic-reconstruction-system-setup.md]. It addresses hardware-specific constraints like phase-only encoding, pixel crosstalk, and diffraction order filtering, which are absent in pure numerical methods.

### Complex Magnitude Extraction & Histogram Enhancement
A post-processing step applied after the absolute value extraction of the reconstructed wavefield. It involves clipping extreme intensity values at a user-defined percentile threshold and applying linear histogram stretching to enhance contrast before final image generation [S1].

## Background
The method is implemented in tools that allow users to configure wavelength, pixel pitch, and reconstruction method (ASM or Fresnel) via external configuration files to avoid re-declaration [S1]. It is used to reconstruct datasets such as "Dices 8K" (using ASM at $z=0.004$ m) and "Skull" (using Fresnel at $z=0.169$ m) [S1]. The output is typically saved as a standard PNG image [S1].

## Sources
[S1] Recent Advances in the Processing and Rendering Algorithms for Computer-Generated Holography, https://doi.org/10.3390/electronics8050556, fetched 2026-10-26
