# Spherical Harmonics
Also called: SH basis, SH coefficients, Spherical Harmonic Representation of Radiance, Spherical Harmonic Volumetric Rendering

## Overview
Spherical Harmonics (SH) is a family of methods that represent directional functions, such as radiance or lighting, on the unit sphere using a linear combination of orthogonal basis functions [S1, S3, unofficial]. This representation enables efficient rotation, integration, and compression of high-dimensional directional data, solving the problem of expensive Monte Carlo integration in volume rendering by replacing it with algebraic operations on coefficient vectors [S1, unofficial]. Unlike general spherical basis methods, SH uses a fixed, non-learnable basis, distinguishing it from alternatives like Spherical Gaussians (SG) which use learnable kernels [S1, unofficial].

## Shared principles
The family relies on the mathematical properties of spherical harmonics, which form a complete basis of functions mapping the unit sphere $S^2$ to the complex numbers $\mathbb{C}$ [S1, unofficial]. In graphics, real-valued bases are typically used to represent real-valued signals like radiance [S1, unofficial].

### Basis Definition
The complex spherical harmonic of degree $\ell \in \mathbb{N} \cup \{0\}$ and order $m \in \{-\ell, \dots, \ell\}$ is defined as [S1, unofficial]:

$$
Y_\ell^m(\theta, \phi) = \sqrt{\frac{2\ell + 1}{4\pi} \frac{(\ell - m)!}{(\ell + m)!}} P_\ell^m(\cos \theta) e^{im\phi}
$$

where $P_\ell^m$ are the associated Legendre polynomials [S1, unofficial].

The real-valued basis $Y_\ell^m: S^2 \to \mathbb{R}$ is constructed from the complex analogue [S1, unofficial]:

$$
Y_\ell^m = 
\begin{cases} 
\sqrt{2} \cdot (-1)^m \cdot \text{Im}[Y_\ell^{|m|}] & \text{if } m < 0 \\
Y_\ell^0 & \text{if } m = 0 \\
\sqrt{2} \cdot (-1)^m \cdot \text{Re}[Y_\ell^m] & \text{if } m > 0 
\end{cases}
$$

### Coefficient Storage and Indexing
A spherical function $f(\omega)$ is approximated by a vector of coefficients $\mathbf{f} \in \mathbb{R}^n$, where $n$ is the number of coefficients [S1, Parent, unofficial]. The coefficients are stored in a 1D vector. The linear index $i$ maps the degree $l$ and order $m$ using the formula [S1, Parent, unofficial]:

$$
i = l^2 + l + m
$$

where $l \in \{0, \dots, L\}$ and $m \in \{-l, \dots, l\}$. The total number of coefficients is $n = (L+1)^2$ [S1, Parent, unofficial].

### Operations
1.  **Projection:** The coefficients are computed by projecting the function onto the basis [S1, unofficial]:
    $$
    f_i = \int_{\Omega} f(\omega) y_i(\omega) d\omega
    $$
2.  **Reconstruction:** The function is approximated at any direction $\omega$ by [S1, unofficial]:
    $$
    \tilde{f}(\omega) = \sum_{i=1}^{n} f_i y_i(\omega)
    $$
3.  **Product Convolution:** The SH projection of the product of two functions $\tilde{a}(\omega)$ and $\tilde{b}(\omega)$ is computed using a precomputed transfer matrix $C_{ijk}$ [S1, unofficial]:
    $$
    (a * b)_i = \sum_{j=1}^{n} \sum_{k=1}^{n} C_{ijk} a_j b_k
    $$
    where $C_{ijk} = \int_{\Omega} y_i(\omega) y_j(\omega) y_k(\omega) d\omega$ [S1, unofficial].
4.  **Integral of Product:** The integral of the product of two approximated functions is the dot product of their coefficients [S1, unofficial]:
    $$
    \int_{\Omega} \tilde{a}(\omega)\tilde{b}(\omega)d\omega = \mathbf{a} \cdot \mathbf{b}
    $$

## Subtopics

### [Spherical Harmonics Approximation](8741-spherical-harmonics-approximation.md)
Focuses on the projection mechanics and efficient integration of SH coefficients in direct volume ray casting, replacing noisy Monte Carlo integration with algebraic dot products.

### [Volumetric Spherical Harmonics Precomputed Radiance Transfer](5807-volumetric-spherical-harmonics-precomputed-radiance-transfer.md)
Uses SH to store position- and direction-dependent scattered radiance in a 3D grid, allowing fast integration via sparse matrix multiplication during rendering.

### [Spherical Harmonic Volume Shading](5946-spherical-harmonic-volume-shading.md)
Represents the directional distribution of scattered radiance at a position in a 3D grid using SH basis functions, enabling efficient rotation and integration.

### [Ray Tracing Backward Pass](5966-ray-tracing-backward-pass.md)
Utilizes SH for differentiable rendering, allowing gradients to be computed with respect to scene parameters.

### H-Basis Encoding
H-Basis is a basis used to encode a hemispherical signal, requiring fewer coefficients to achieve the same quality as 3rd order spherical harmonics [S4, unofficial]. It requires a local coordinate system to orient the hemisphere, commonly utilizing a tangent frame from uv-parameterization [S4, unofficial]. If encoded in textures, sufficient resolution is needed to adapt to tangent space changes; if two triangles cover the same texel with significantly different tangents, precision in the reconstructed signal is lost [S4, unofficial].

### Spherical Harmonics Degree Selection
The choice of SH degree affects accuracy and memory usage. Higher degrees increase PSNR but require significantly more memory, while computational cost is not obviously affected by degree [S5]. PSNR results for estimated color fields on DVGO density fields are:
- **256^3 grid:** Degree 0 (1 basis): 26.13, Degree 1 (4 bases): 27.22, Degree 2 (9 bases): 26.99, Degree 3 (16 bases): 28.29, Degree 4 (25 bases): 27.89 [S5].
- **512^3 grid:** Degree 0: 26.99, Degree 1: 28.29, Degree 2: 29.44, Degree 3: 28.30, Degree 4: 29.98 [S5].

Switching between max degree 3 (SH-16) and 4 (SH-25) yields negligible differences in metrics or visual quality for many applications [S1, unofficial].

## Background
Spherical Harmonics were introduced in computer graphics for global illumination and environment map compression [S3, unofficial]. They have been used for efficient multiple scattering in hair rendering, storing scattered radiance in 3D grids to avoid high path-tracing costs [S2]. In recent years, SH has become central to neural rendering techniques like NeRF and PlenOctrees, where they represent directional radiance for real-time rendering [S1, unofficial]. They are also used in 3D Gaussian Splatting for representing lighting and reflection [S3, unofficial].

## Sources
[S1] PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs), http://alexyu.net/plenoctrees/, fetched 2026-10-28 (unofficial)
[S2] Efficient multiple scattering in hair using spherical harmonics, https://doi.org/10.1145/1360612.1360630, fetched 2026-10-29
[S3] 3D Gaussian Splatting 入门原创, https://blog.csdn.net/qq_35831906/article/details/143322205, fetched 2026-10-30 (unofficial)
[S4] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S5] Reducing Shape-Radiance Ambiguity in Radiance Fields with a Closed-Form Color Estimation Method, https://doi.org/10.52202/075280-2570, fetched 2026-10-26
