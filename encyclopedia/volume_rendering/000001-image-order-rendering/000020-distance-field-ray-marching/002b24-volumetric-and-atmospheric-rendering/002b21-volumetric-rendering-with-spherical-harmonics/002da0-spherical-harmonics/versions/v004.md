# Spherical Harmonics
Also called: Spherical Harmonic Volumetric Rendering

## Overview
Spherical Harmonics (SH) provide a complete basis of functions mapping the unit sphere $S^2$ to the complex numbers $\mathbb{C}$, used to represent directional functions such as radiance, lighting, and reflection [S1, S2, unofficial]. In volumetric rendering, they enable the efficient compression and differentiable evaluation of spherical signals, avoiding expensive Monte Carlo integration by converting integrals of products into linear algebra operations [S1, unofficial]. Unlike Spherical Gaussians (SGs), which use learnable Gaussian kernels, SHs are a fixed, non-learnable basis where switching between standard bandwidths (e.g., SH-16 vs SH-25) yields negligible differences in visual quality or metrics [S1, unofficial].

## Shared principles
The SH family relies on expanding a spherical function $f(\omega)$ into a finite set of orthogonal basis functions.

**Basis Functions:**
The complex SH basis function of degree $\ell \in \mathbb{N} \cup \{0\}$ and order $m \in \{-\ell, \dots, \ell\}$ is defined as [S1, unofficial]:
$$
Y_\ell^m(\theta, \phi) = \sqrt{\frac{2\ell + 1}{4\pi} \frac{(\ell - m)!}{(\ell + m)!}} P_\ell^m(\cos \theta) e^{im\phi}
$$
where $P_\ell^m$ are the associated Legendre polynomials [S1, unofficial].

For graphics applications, real-valued basis functions $Y_\ell^m: S^2 \to \mathbb{R}$ are constructed from the complex analogue [S1, unofficial]:
$$
Y_\ell^m = 
\begin{cases} 
\sqrt{2} \cdot (-1)^m \cdot \text{Im}[Y_\ell^{|m|}] & \text{if } m < 0 \\
Y_\ell^0 & \text{if } m = 0 \\
\sqrt{2} \cdot (-1)^m \cdot \text{Re}[Y_\ell^m] & \text{if } m > 0 
\end{cases}
$$
[S1, unofficial].

**Coefficient Representation:**
A function is represented by a coefficient vector $\mathbf{f} \in \mathbb{R}^n$, where $n = (L+1)^2$ is the number of coefficients for a maximum degree $L$. The index $i$ linearizes the degree $\ell$ and order $m$ using the mapping [S1, Parent, unofficial]:
$$
i = \ell^2 + \ell + m
$$
where $\ell \in \{0, \dots, L\}$ and $m \in \{-\ell, \dots, \ell\}$.

**Projection and Reconstruction:**
The coefficients $f_i$ are obtained by projecting the function onto the basis [S1, Parent, unofficial]:
$$
f_i = \int_{\Omega} f(\omega) y_i(\omega) d\omega
$$
The signal is reconstructed at a direction $\omega$ via [S1, Parent, unofficial]:
$$
\tilde{f}(\omega) = \sum_{i=1}^{n} f_i y_i(\omega)
$$

**Product Convolution:**
The integral of the product of two approximated functions $\tilde{a}(\omega)$ and $\tilde{b}(\omega)$ is the dot product of their coefficients [S1, Parent, unofficial]:
$$
\int_{\Omega} \tilde{a}(\omega)\tilde{b}(\omega)d\omega = \mathbf{a} \cdot \mathbf{b}
$$
The SH coefficients of the product of two functions, denoted $\mathbf{a} * \mathbf{b}$, are computed using a precomputed 3D transfer matrix $C$ of size $n \times n \times n$, where $C_{ijk} = \int_{\Omega} y_i(\omega)y_j(\omega)y_k(\omega) d\omega$ [S1, unofficial]:
$$
(\mathbf{a} * \mathbf{b})_i = \sum_{j=1}^{n} \sum_{k=1}^{n} C_{ijk} a_j b_k
$$
[S1, Parent, unofficial].

**Properties and Pitfalls:**
*   **Gibbs Phenomenon:** Truncating the series causes ringing artifacts for high-frequency signals [Parent].
*   **Smoothness:** The method assumes input functions are sufficiently smooth; discontinuities (e.g., hard shadows) cause slow convergence [Parent].
*   **Rotation:** Rotating an SH function requires applying a linear transformation defined by Wigner D-matrices to the coefficient vector [Parent].

## Subtopics

### [Spherical Harmonics Approximation](8741-spherical-harmonics-approximation.md)
Targets the computational efficiency of the *product* of functions (e.g., visibility $\times$ lighting) within inverse problems. It sets itself apart by precomputing transfer matrices and using linear algebra to evaluate integrals, avoiding Monte Carlo methods. It specifically handles geometry-dependent terms like hemisphere response via rotation matrices to maintain differentiability.

### [Volumetric Spherical Harmonics Precomputed Radiance Transfer](5807-volumetric-spherical-harmonics-precomputed-radiance-transfer.md)
(See sibling article for details).

### [Spherical Harmonic Volume Shading](5946-spherical-harmonic-volume-shading.md)
(See sibling article for details).

### [Ray Tracing Backward Pass](5966-ray-tracing-backward-pass.md)
(See sibling article for details).

### H-Basis Encoding
**Definition:** A basis used to encode a hemispherical signal, requiring fewer coefficients to achieve the same quality as 3rd order spherical harmonics [S3, unofficial].
**Step:** A coordinate system local to the surface is required to orient the hemisphere, commonly utilizing a tangent frame resulting from uv-parameterization [S3, unofficial].
**Pitfall:** If encoded in textures, the texture needs sufficient resolution to adapt to changes in the underlying tangent space. If two triangles cover the same texel and have significantly different tangents, it leads to a lack of precision in the reconstructed signal [S3, unofficial].

## Background
Spherical Harmonics are a set of special functions defined on a sphere, used to efficiently describe arbitrary functions on the sphere surface [S2, unofficial]. In computer graphics, they are widely applied to represent lighting, reflection, and compress environment illumination [S2, unofficial]. They are particularly effective for capturing low-frequency lighting components, such as ambient occlusion and global illumination, allowing for efficient rendering of complex lighting conditions [S2, unofficial]. The use of SH in rendering equations for inverse problems and neural radiance fields (NeRFs) has been popularized in recent years, with implementations like PlenOctrees demonstrating their utility in real-time rendering [S1, unofficial].

## Sources
[S1] PlenOctrees For Real-time Rendering of Neural Radiance Fields (NeRFs), http://alexyu.net/plenoctrees/, fetched 2026-10-28 (unofficial)
[S2] 3D Gaussian Splatting 入门原创, https://blog.csdn.net/qq_35831906/article/details/143322205, fetched 2026-10-30 (unofficial)
[S3] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
