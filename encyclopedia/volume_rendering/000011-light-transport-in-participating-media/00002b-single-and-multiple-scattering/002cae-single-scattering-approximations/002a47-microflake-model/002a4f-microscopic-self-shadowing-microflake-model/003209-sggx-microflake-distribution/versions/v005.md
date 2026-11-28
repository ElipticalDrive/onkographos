# SGGX Microflake Distribution

Also called: Micro-flake normal distribution, SGGX Phase Function Representation

## Overview
The SGGX (Single Scattering GGX) microflake distribution is a parametric model for the normal distribution function (NDF) of microflakes in participating media, introduced by Heitz et al. [HDCD15]. It solves the problem of efficiently modeling anisotropic scattering by providing a closed-form representation that supports fast, exact evaluation of attenuation coefficients and efficient sampling of specular phase functions. It sets itself apart from previous distributions by overcoming limitations in computational efficiency and robustness, serving as the foundational component for the [Microscopic Self-Shadowing Microflake Model](4746-microscopic-self-shadowing-microflake-model.md) and its variants.

## How it works
The SGGX distribution models the orientation of microflakes using a single 3x3 symmetric positive definite (SPD) matrix $S$. This matrix defines the shape and orientation of the scattering lobe.

**Inputs and Outputs:**
- Input: A direction vector $m$ (the microflake normal), and the SPD matrix $S$.
- Output: The probability density value $D(m)$ and, when sampled, a new normal vector $m$.

**Data Structures:**
- $S$: A 3x3 symmetric positive definite matrix. In implementation, this is typically stored as 6 floating-point values (upper triangular part) or 9 values if symmetry is not enforced during storage. The matrix must remain SPD to ensure the distribution is valid.

**Formula:**
The probability density function for the microflake normal $m$ is given by [S2]:

$$
D(m) = \frac{1}{\pi \sqrt{|S|} (m^T S^{-1} m)^2}
$$

Where:
- $|S|$ is the determinant of matrix $S$.
- $S^{-1}$ is the inverse of matrix $S$.
- $m$ is a unit vector representing the microflake normal.
- The term $(m^T S^{-1} m)$ represents the quadratic form of the inverse matrix.

**Parameters:**
- $S$: The 3x3 SPD matrix defining the anisotropy and spread of the microflake normals. Typical values depend on the material's roughness and orientation. For isotropic scattering, $S$ is a scaled identity matrix. For anisotropic scattering, $S$ has distinct eigenvalues corresponding to the principal axes of the microflakes [S2].

**Algorithm for Evaluation:**
1. Compute the inverse of $S$, denoted $S^{-1}$.
2. Compute the determinant $|S|$.
3. Compute the quadratic form $q = m^T S^{-1} m$.
4. Compute the denominator $denom = \pi \sqrt{|S|} q^2$.
5. Return $1 / denom$.

**Sampling:**
While the exact sampling algorithm is not detailed in the provided notes, Heitz et al. [HDCD15] provide efficient sampling procedures for SGGX distributions. The sampling typically involves transforming a uniform sample into the space defined by $S$ and then mapping it back to the sphere, ensuring the resulting normal $m$ follows the distribution $D(m)$.

**Edge Cases and Pitfalls:**
- **SPD Constraint:** The matrix $S$ must remain symmetric positive definite. If $S$ becomes singular or indefinite, the distribution is undefined or invalid. This can occur during interpolation or optimization if not properly constrained [S2].
- **Precision:** The computation of $m^T S^{-1} m$ can suffer from precision issues if $S$ is ill-conditioned. Care should be taken in numerical implementations.
- **Normalization:** The formula includes a normalization constant $\frac{1}{\pi \sqrt{|S|}}$. Ensure that the determinant and inverse are computed accurately to maintain proper normalization.

## Subtopics

### [GGX Mixture NDF](10859-ggx-mixture-ndf.md)
The GGX Mixture NDF is a compact representation of the marginal distribution of surface normals, $p_N(n)$, used to model glossiness and anisotropy in participating media and microflake models. It approximates the surface NDF as a weighted sum of Single Scattering GGX (SGGX) distributions. This method sets itself apart from single-lobe SGGX models by allowing the representation of complex, multi-modal normal distributions, and from aggregated microfacet convolutions by providing the base component that is subsequently convolved with isotropic spherical distributions to model surface roughness effects.

## Operating characteristics
- **Time Complexity:** The evaluation of $D(m)$ is $O(1)$ assuming fixed-size matrix operations (3x3). Inversion and determinant computation are also constant time for 3x3 matrices.
- **Memory Complexity:** $O(1)$ per distribution, storing only the 6 unique elements of the SPD matrix $S$.
- **Performance:** The SGGX distribution is computationally efficient compared to state-of-the-art models [Jakob et al. 2010; Zhao et al. 2011] while offering similar representative power for anisotropic scattering [S2].
- **Strengths:** Fast and exact evaluation of attenuation coefficients; efficient evaluation and sampling of specular phase functions; compact representation of anisotropy.
- **Limitations:** Single-lobe representation may not capture complex multi-modal normal distributions without mixing (see [GGX Mixture NDF](10859-ggx-mixture-ndf.md)). Requires careful maintenance of the SPD property for $S$.
- **When to use:** When modeling anisotropic scattering in participating media where computational efficiency is critical and the microflake orientation distribution can be approximated by a single lobe.
- **When not to use:** When the normal distribution is highly multi-modal or requires complex correlations that a single SPD matrix cannot capture.

## Background
The SGGX distribution was introduced by Heitz et al. in 2015 [HDCD15]. It was developed to address limitations in previous microflake normal distributions, specifically regarding the efficiency of evaluating attenuation coefficients and sampling phase functions. The model has since been used in various volume rendering contexts, including downsampling volume data [S1, S2] and implementing simplified self-shadowing models [S1].

## Sources
[S1] A new microflake model with microscopic self-shadowing for accurate volume downsampling, https://doi.org/10.1111/cgf.13346, fetched 2026-10-26
[S2] Downsampling scattering parameters for rendering anisotropic media, https://doi.org/10.1145/2980179.2980228, fetched 2026-10-28
