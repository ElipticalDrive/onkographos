# Spherical Harmonics
Also called: Spherical Harmonic Volumetric Rendering

## Overview
Spherical Harmonics (SH) provide a complete basis of functions mapping the unit sphere $S^2$ to the complex numbers $\mathbb{C}$, used to represent directional functions like radiance in volumetric rendering [S1, unofficial]. Unlike Spherical Gaussians (SGs), SHs constitute a fixed, non-learnable basis, which simplifies implementation by removing the need for optimization of basis parameters [S1, unofficial].

## Shared principles
The method relies on expanding directional signals (such as lighting or view-dependent appearance) into a series of orthogonal basis functions defined on the sphere. The basis is indexed by a degree $\ell \in \mathbb{N} \cup \{0\}$ and an order $m \in \{-\ell, \dots, \ell\}$.

### Complex Basis Definition
The complex spherical harmonic basis function $Y_\ell^m(\theta, \phi)$ is defined as [S1, unofficial]:

$$
Y_\ell^m(\theta, \phi) = \sqrt{\frac{2\ell + 1}{4\pi} \frac{(\ell - m)!}{(\ell + m)!}} P_\ell^m(\cos \theta) e^{im\phi}
$$

where:
*   $\theta$ is the polar angle (inclination).
*   $\phi$ is the azimuthal angle.
*   $P_\ell^m$ are the associated Legendre polynomials.

### Real-Valued Basis Construction
For rendering applications, a real-valued basis $\tilde{Y}_\ell^m: S^2 \to \mathbb{R}$ is typically constructed from the complex analogue to ensure real-valued coefficients and outputs [S1, unofficial]. The mapping is defined as:

$$
\tilde{Y}_\ell^m = 
\begin{cases} 
\sqrt{2} \cdot (-1)^m \cdot \text{Im}[Y_\ell^{|m|}] & \text{if } m < 0 \\
Y_\ell^0 & \text{if } m = 0 \\
\sqrt{2} \cdot (-1)^m \cdot \text{Re}[Y_\ell^m] & \text{if } m > 0 
\end{cases}
$$

### Basis Size and Truncation
The total number of basis functions for a maximum degree $L$ is $(L+1)^2$. Common truncations include:
*   **SH-9**: Degree $L=2$ (9 coefficients).
*   **SH-16**: Degree $L=3$ (16 coefficients).
*   **SH-25**: Degree $L=4$ (25 coefficients).

Empirical ablation on the NeRF-synthetic dataset indicates that switching between max degree 3 (SH-16) and 4 (SH-25) yields negligible differences in metrics (PSNR, SSIM) or visual quality [S1, unofficial].

## Subtopics

### [Spherical Harmonic Volume Shading](5946-spherical-harmonic-volume-shading.md)
Applies SH to encode view-dependent shading within the volume.

### [Ray Tracing Backward Pass](5966-ray-tracing-backward-pass.md)
Uses SH for differentiable rendering gradients.

## Background
Spherical Harmonics are a standard tool in computer graphics for representing spherical functions, originating from physics and mathematics. In the context of neural volume rendering, they were adopted in PlenOctrees to compactly represent the directional radiance field, allowing for real-time rendering of Neural Radiance Fields (NeRFs) [S1, unofficial]. The specific implementation details and ablation studies regarding SH vs. SG in this context are documented in the PlenOctrees paper [S1, unofficial].

## Sources
[S1] PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs), http://alexyu.net/plenoctrees/, fetched 2026-10-28 (unofficial)
