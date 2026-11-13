# Spherical Harmonics
Also called: SH basis, SH coefficients, Spherical Harmonic Representation of Radiance, Spherical Harmonic Volumetric Rendering, Spherical Harmonics (SH) Basis

## Overview
Spherical Harmonics (SH) provide an orthonormal basis for representing directional functions defined on the unit sphere $S^2$, enabling the compact storage and efficient manipulation of volumetric radiance distributions [S2][S3, S4, unofficial]. This family of methods solves the problem of expensive numerical integration for directional signals (such as in-scattering or visibility) by projecting them into a coefficient space where operations like rotation and integration become analytic or linear algebraic [S1, unofficial]. It differs from its siblings, such as [Volumetric Spherical Harmonics Precomputed Radiance Transfer](5807-volumetric-spherical-harmonics-precomputed-radiance-transfer.md) and [Spherical Harmonic Volume Shading](5946-spherical-harmonic-volume-shading.md), by serving as the fundamental mathematical representation layer; while the siblings apply this representation to specific shading or transfer pipelines, the core SH family focuses on the basis definition, coefficient projection, and the algebraic properties (like the product convolution) that make these applications feasible [S1, unofficial][S2][S3, unofficial].

## Shared principles
The SH family relies on the mathematical properties of spherical harmonics to represent a spherical function $f(\omega)$ as a linear combination of basis functions. The following principles are common to all child methods in this family.

### Basis Definition
The spherical harmonics form a complete orthonormal basis over the sphere. The complex basis function $Y_\ell^m$ for degree $\ell \in \mathbb{N} \cup \{0\}$ and order $m \in \{-\ell, \dots, \ell\}$ is defined as [S1, S3, unofficial]:

$$
Y_\ell^m(\theta, \phi) = \sqrt{\frac{2\ell + 1}{4\pi} \frac{(\ell - m)!}{(\ell + m)!}} P_\ell^m(\cos \theta) e^{im\phi}
$$

where $P_\ell^m$ are the associated Legendre polynomials [S1, S3, unofficial].

To avoid imaginary numbers for $m \neq 0$, a real-valued basis $Y_\ell^m: S^2 \to \mathbb{R}$ is constructed piecewise [S1, S3, unofficial]:

$$
Y_\ell^m(\theta, \phi) = 
\begin{cases} 
\sqrt{2} K_{\ell m} \cos(m\phi) P_\ell^m(\cos \theta) & \text{if } m > 0 \\
K_{\ell 0} P_\ell^0(\cos \theta) & \text{if } m = 0 \\
\sqrt{2} K_{\ell |m|} \sin(|m|\phi) P_\ell^{|m|}(\cos \theta) & \text{if } m < 0 
\end{cases}
$$

where $K_{\ell m} = \sqrt{\frac{2\ell + 1}{4\pi} \frac{(\ell - m)!}{(\ell + m)!}}$ [S3, unofficial].

### Coefficient Representation
A directional signal $f(\omega)$ is represented by a vector of coefficients $\mathbf{f} \in \mathbb{R}^n$, where $n = (L+1)^2$ for a target bandwidth $L$ [S1, unofficial]. The linear index $i$ maps the degree $l$ and order $m$ using the formula [S1, unofficial]:

$$
i = l^2 + l + m
$$

The total number of coefficients is $n = (L+1)^2$ [S1, Parent, unofficial].

### Projection and Reconstruction
The coefficients $f_i$ are computed by projecting the function onto the basis [S1, unofficial]:

$$
f_i = \int_{\Omega} f(\omega) y_i(\omega) d\omega
$$

The function is reconstructed at any direction $\omega$ by summing the basis functions weighted by the coefficients [S1, unofficial]:

$$
\tilde{f}(\omega) = \sum_{i=1}^{n} f_i y_i(\omega)
$$

### Product Convolution
To compute the SH projection of the product of two functions $\tilde{a}(\omega)$ and $\tilde{b}(\omega)$, the method uses a precomputed transfer matrix $C_{ijk}$ [S1, unofficial]:

$$
(a * b)_i = \sum_{j=1}^{n} \sum_{k=1}^{n} C_{ijk} a_j b_k
$$

where the transfer matrix elements are [S1, unofficial]:

$$
C_{ijk} = \int_{\Omega} y_i(\omega) y_j(\omega) y_k(\omega) d\omega
$$

The integral of the product of two approximated functions is the dot product of their coefficient vectors [S1, unofficial]:

$$
\int_{\Omega} \tilde{a}(\omega)\tilde{b}(\omega)d\omega = \mathbf{a} \cdot \mathbf{b}
$$

### Parameters and Pitfalls
*   **Bandwidth:** Typical implementations use the lowest 3-5 levels of SH to keep the representation compact [S3, unofficial]. A bandwidth of $n=8$ is cited as providing good low-frequency approximations [S1, unofficial].
*   **Ringing:** SH approximations containing higher frequencies become too large to be practical and exhibit 'ringing' artifacts (high-frequency noise) near sharp value changes, especially when many coefficients are used [S3, unofficial].
*   **Comparison with SGs:** Unlike Spherical Gaussians (SGs), SHs are a fixed, non-learnable basis [S1, unofficial]. Switching between max degree 3 (SH-16) and 4 (SH-25) yields negligible differences in metrics or visual quality in some contexts [S1, unofficial].

## Subtopics

### [Spherical Harmonics Approximation](8741-spherical-harmonics-approximation.md)
Spherical Harmonics (SH) Approximation is a technique to represent a spherical function $f(\omega)$ as a linear combination of a finite subset of spherical harmonics basis functions. It solves the problem of expensive Monte Carlo integration for directional signals (such as radiance, visibility, or in-scattering) by enabling fast, differentiable, and analytic computation via dot products and precomputed transfer matrices. It differs from its siblings by focusing on the practical application of this basis for inverse problems and real-time rendering, specifically leveraging precomputed coefficients and the product convolution property to avoid numerical integration during runtime.

### [Volumetric Spherical Harmonics Precomputed Radiance Transfer](5807-volumetric-spherical-harmonics-precomputed-radiance-transfer.md)
Not given in the sources.

### [Spherical Harmonic Volume Shading](5946-spherical-harmonic-volume-shading.md)
Not given in the sources.

### [Ray Tracing Backward Pass](5966-ray-tracing-backward-pass.md)
Not given in the sources.

### [Zonal Method (Bhate and Tokuta)](12221-zonal-method-bhate-and-tokuta.md)
Not given in the sources.

### H-Basis Encoding
H-Basis is a basis used to encode a hemispherical signal, requiring fewer coefficients to achieve the same quality as 3rd order spherical harmonics [S5, unofficial]. A coordinate system local to the surface is required to orient the hemisphere, commonly utilizing a tangent frame resulting from uv-parameterization [S5, unofficial]. If encoded in textures, the texture needs sufficient resolution to adapt to changes in the underlying tangent space [S5, unofficial]. If two triangles cover the same texel and have significantly different tangents, it leads to a lack of precision in the reconstructed signal [S5, unofficial].

### Spherical Harmonics Degree Selection
The choice of SH degree affects the accuracy and memory usage of the representation. PSNR of estimated color fields on a $256^3$ grid using DVGO density fields increases with SH degree: Degree 0 (1 basis): 26.13, Degree 1 (4 bases): 27.22, Degree 2 (9 bases): 26.99, Degree 3 (16 bases): 28.29, Degree 4 (25 bases): 27.89. On a $512^3$ grid: Degree 0: 26.99, Degree 1: 28.29, Degree 2: 29.44, Degree 3: 28.30, Degree 4: 29.98 [S6]. Higher SH degrees increase PSNR but require significantly more memory. Computational cost is not obviously affected by SH degree [S6].

## Background
Spherical Harmonics were introduced as a complete basis of functions mapping the unit sphere $S^2$ to the complex numbers $\mathbb{C}$ [S1, unofficial]. In computer graphics, they have been used to represent the directional distribution of scattered radiance $L_s(x, \boldsymbol{\theta})$ at a position $x$ in a 3D grid, allowing for efficient rotation and integration [S2]. They are an infinite set of spherical functions forming an orthonormal basis over the sphere, used to approximate arbitrary spherical functions (e.g., lighting environments) via a sum of basis functions with coefficients [S3, unofficial]. In graphics, they are a set of special functions defined on a sphere, used to efficiently describe arbitrary functions on the sphere surface, representing lighting, reflection, and compressing environment illumination [S4, unofficial].

## Sources
[S1] PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs), http://alexyu.net/plenoctrees/, fetched 2026-10-28 (unofficial)
[S2] Efficient multiple scattering in hair using spherical harmonics, https://doi.org/10.1145/1360612.1360630, fetched 2026-10-29
[S3] PDF A Volumetric Approach to Rendering Microgeometry Using PRT, https://www.cs.umbcm.edu/~ugrad/honors/theses/JohnKloetzli.pdf, fetched 2026-10-28 (unofficial)
[S4] 3D Gaussian Splatting 入门原创, https://blog.csdn.net/qq_35831906/article/details/143322205, fetched 2026-10-30 (unofficial)
[S5] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S6] Reducing Shape-Radiance Ambiguity in Radiance Fields with a Closed-Form Color Estimation Method, https://doi.org/10.52202/075280-2570, fetched 2026-10-26
