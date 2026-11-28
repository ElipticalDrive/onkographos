# Gamma-Corrected Hemispherical Shading

Also called: Cubic Shading in Fourier Domain, Fourier Domain Gamma-Corrected Shading, Gamma-Corrected Shading

## Overview
Gamma-Corrected Hemispherical Shading (GCHS) is an illumination model for Fourier Volume Rendering that approximates the non-linear diffuse operator $\max(0, \mathbf{N} \cdot \mathbf{L})$ by raising the hemispherical term $(1 + \mathbf{N} \cdot \mathbf{L})/2$ to a gamma power [S1, unofficial][S2]. It replaces linear hemispherical shading with a cubic polynomial expansion, which culls back-lit surfaces more effectively and reduces erroneous energy on rear-facing surfaces compared to linear approximations [S1, unofficial][S2]. Unlike the non-linear max operator, GCHS expands into a sum of geometric terms weighted by lighting coefficients, allowing geometric components to be pre-computed in the Fourier domain and re-evaluated for changing light directions via linear combinations [S1, unofficial][S2].

## How it works
**Inputs:**
*   Volume data $f(\mathbf{x})$ [S1, unofficial][S2].
*   Light direction vector $\mathbf{L} = (l_1, l_2, l_3)$, normalized [S1, unofficial][S2].
*   Gamma exponent $g=3$ [S1, unofficial][S2].

**Outputs:**
*   Shaded volume rendering contribution in the Fourier domain, ready for back-projection or further processing [S1, unofficial].

**Data Structures:**
*   $F(\mathbf{\Omega})$: The 3D Fourier transform of the volume data $f$ [S1, unofficial][S2].
*   $\mathbf{\Omega} = (\omega_x, \omega_y, \omega_z)$: The frequency vector [S1, unofficial][S2].
*   Basis images: Twenty 8-bit sample basis images storing the geometric terms $V_i$ [S1, unofficial].
*   Coefficient matrix: A $20 \times 20$ matrix derived from corresponding lighting directions for the coefficients $a_i$ [S1, unofficial].

**Method:**
The method approximates the diffuse shading by expanding the cubic hemispherical term into a polynomial of dot products between the gradient $\nabla f$ and the light vector $\mathbf{L}$, normalized by the gradient magnitude $\|\nabla f\|$ [S1, unofficial][S2].

1.  **Define the Shading Operator:**
    The base hemispherical shading operator is $E = \frac{1}{2}(1 + \mathbf{N} \cdot \mathbf{L})$, where $\mathbf{N}$ is the normalized surface normal [S1, unofficial]. For GCHS, this is raised to the power $g=3$ and scaled:
    $$ E_{GCHS} = \frac{1}{8}(1 + \mathbf{N} \cdot \mathbf{L})^3 $$
    [S2]

2.  **Expand the Polynomial:**
    Substitute the normal $\mathbf{N} = \frac{\nabla f}{\|\nabla f\|}$ into the expansion:
    $$ (1 + \mathbf{N} \cdot \mathbf{L})^3 = \left(1 + \frac{\nabla f \cdot \mathbf{L}}{\|\nabla f\|}\right)^3 = 1 + 3 \frac{\nabla f \cdot \mathbf{L}}{\|\nabla f\|} + 3 \frac{(\nabla f \cdot \mathbf{L})^2}{\|\nabla f\|^2} + \frac{(\nabla f \cdot \mathbf{L})^3}{\|\nabla f\|^3} $$
    [S1, unofficial][S2]
    This expansion results in twenty (20) distinct lighting and geometry terms when fully expanded into components [S1, unofficial]. The coefficients $a_i$ depend only on $\mathbf{L}$, and the geometric terms $V_i$ depend only on unit surface normals $\mathbf{N}$ [S2].

3.  **Compute Linear Term in Fourier Domain:**
    The linear term is $\nabla f \cdot \mathbf{L} = f_x l_1 + f_y l_2 + f_z l_3$ [S1, unofficial][S2]. Using the property that spatial differentiation corresponds to multiplication by $i\mathbf{\Omega}$ in the Fourier domain:
    $$ \mathcal{T}\{\nabla f \cdot \mathbf{L}\} = (i\mathbf{\Omega} \cdot \mathbf{L}) F $$
    where $\mathcal{T}$ denotes the Fourier transform, $F$ is the Fourier transform of $f$, and $\mathbf{\Omega} \cdot \mathbf{L} = \omega_x l_1 + \omega_y l_2 + \omega_z l_3$ [S2] (note: [S1, unofficial] omits the imaginary unit $i$).

4.  **Compute Quadratic Term in Fourier Domain:**
    The quadratic term is $(\nabla f \cdot \mathbf{L})^2$ [S2]. A brute-force expansion yields terms like $l_1^2 f_x^2$, $l_2^2 f_y^2$, $l_3^2 f_z^2$, and cross-terms like $2l_1 l_2 f_x f_y$ [S2]. To compute products of derivatives in the Fourier domain, use the identity $f_a f_b = \frac{1}{2}(\mathcal{F}^{-1}\{F_a F_b\}) - f f_{ab}$ [S1, unofficial]. The source provides the frequency domain implementation for the quadratic term as:
    $$ \mathcal{T}\{(\nabla f \cdot \mathbf{L})^2\} = \frac{1}{2}(\mathbf{\Omega} \cdot \mathbf{L})^2 (F \otimes F) - (F \otimes ((\mathbf{\Omega} \cdot \mathbf{L})^2 F)) $$
    [S1, unofficial][S2]
    where $\otimes$ denotes convolution in the spatial domain [S1, unofficial][S2].

5.  **Compute Cubic Term in Fourier Domain:**
    The cubic term is $(\nabla f \cdot \mathbf{L})^3$ [S1, unofficial]. Using the identity for triple products $f_a f_b f_c = \frac{1}{6}(f^3)_{abc} - \frac{1}{2}f(f^2)_{abc} + \frac{1}{2}f^2 f_{abc}$ [S1, unofficial], the frequency domain implementation is:
    $$ \mathcal{T}\{(\nabla f \cdot \mathbf{L})^3\} = \frac{1}{6}(\mathbf{\Omega} \cdot \mathbf{L})^3 (F \otimes F \otimes F) - \frac{1}{2}(F \otimes ((\mathbf{\Omega} \cdot \mathbf{L})^3 (F \otimes F))) + \frac{1}{2}(F \otimes F \otimes ((\mathbf{\Omega} \cdot \mathbf{L})^3 F)) $$
    [S1, unofficial][S2]

6.  **Normalization:**
    The terms $\frac{1}{\|\nabla f\|}$, $\frac{1}{\|\nabla f\|^2}$, and $\frac{1}{\|\nabla f\|^3}$ must be precomputed and applied to the respective gradient terms [S1, unofficial].

7.  **Re-evaluation for Changing Lights:**
    The geometric terms (linear, quadratic, cubic components of $\nabla f$) are computed once [S2]. For a new light direction $\mathbf{L}$, the final shading is obtained by combining these pre-computed geometric terms with the new lighting coefficients $a_i$ derived from $\mathbf{L}$ [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gamma_corrected_hemispherical_shading(F, Omega, L):
    """
    Computes the Fourier domain representation of GCHS shading.
    
    Args:
        F: Complex array, Fourier transform of volume f.
        Omega: Tuple of arrays (omega_x, omega_y, omega_z), frequency coordinates.
        L: Tuple of floats (l1, l2, l3), normalized light direction.
        
    Returns:
        shaded_F: Complex array, Fourier transform of shaded volume.
    """
    l1, l2, l3 = L
    wx, wy, wz = Omega
    i = 1j
    
    # Precompute frequency ramp dot product
    Omega_dot_L = i * (wx * l1 + wy * l2 + wz * l3)
    
    # 1. Linear Term: T{grad f . L}
    linear_term = Omega_dot_L * F
    
    # 2. Quadratic Term: T{(grad f . L)^2}
    # Formula: 0.5 * (Omega . L)^2 * (F conv F) - (F conv ((Omega . L)^2 F))
    # Convolution here denotes spatial convolution, computed via FFT multiplication.
    quad_term = 0.5 * (Omega_dot_L ** 2) * convolve_3d(F, F) - convolve_3d(F, (Omega_dot_L ** 2) * F)
    
    # 3. Cubic Term: T{(grad f . L)^3}
    # Formula: 1/6*(Omega.L)^3*(F conv F conv F) - 1/2*(F conv ((Omega.L)^3*(F conv F))) + 1/2*(F conv F conv ((Omega.L)^3 * F))
    cubic_term = (1/6) * (Omega_dot_L ** 3) * convolve_3d(convolve_3d(F, F), F) \
                 - (1/2) * convolve_3d(F, (Omega_dot_L ** 3) * convolve_3d(F, F)) \
                 + (1/2) * convolve_3d(convolve_3d(F, F), (Omega_dot_L ** 3) * F)
    
    # Combine terms according to expansion: 1 + 3*(linear) + 3*(quadratic) + (cubic)
    # Normalization factors (1/||grad f||, etc.) are applied as precomputed masks outside this function.
    shaded_F = F + 3 * linear_term + 3 * quad_term + cubic_term
    
    return shaded_F

def convolve_3d(A, B):
    """Standard 3D convolution using FFT for efficiency."""
    from numpy.fft import fftn, ifftn
    return ifftn(fftn(A) * fftn(B)).real
```

**Formulas Summary:**
*   **Linear:** $\mathcal{T}\{\nabla f \cdot \mathbf{L}\} = (i\mathbf{\Omega} \cdot \mathbf{L}) F$ [S2] (or $(\mathbf{\Omega} \cdot \mathbf{L}) F$ [S1, unofficial]).
*   **Quadratic:** $\mathcal{T}\{(\nabla f \cdot \mathbf{L})^2\} = \frac{1}{2}(\mathbf{\Omega} \cdot \mathbf{L})^2 (F \otimes F) - (F \otimes ((\mathbf{\Omega} \cdot \mathbf{L})^2 F))$ [S1, unofficial][S2].
*   **Cubic:** $\mathcal{T}\{(\nabla f \cdot \mathbf{L})^3\} = \frac{1}{6}(\mathbf{\Omega} \cdot \mathbf{L})^3 (F \otimes F \otimes F) - \frac{1}{2}(F \otimes ((\mathbf{\Omega} \cdot \mathbf{L})^3 (F \otimes F))) + \frac{1}{2}(F \otimes F \otimes ((\mathbf{\Omega} \cdot \mathbf{L})^3 F))$ [S1, unofficial][S2].

**Parameters:**
*   Gamma exponent $g=3$ [S1, unofficial][S2]. This is the smallest integer power producing an inflection point, minimizing error in both zero and non-zero portions of the shading curve [S1, unofficial][S2].

**Edge Cases and Pitfalls:**
*   **Non-linearity:** The standard $\max(0, \mathbf{N} \cdot \mathbf{L})$ cannot be projected into the Fourier basis [S1, unofficial]. GCHS avoids this by using a polynomial approximation [S1, unofficial].
*   **Negative Intensity:** Ignoring visibility (using simple hemispherical shading) results in negative image intensity values and reduced contrast [S1, unofficial]. GCHS eliminates negative intensity without the max operator [S2].
*   **Light Direction Changes:** The convolution terms in the quadratic and cubic equations require a convolution with the full volume [S1, unofficial]. This can only be precomputed for a constant light source [S1, unofficial]. If the light direction changes, these terms must be recomputed [S1, unofficial].
*   **Performance:** Cubic shading has lower frame rates than linear shading due to additional calculations [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method requires computing convolutions of the volume data in the Fourier domain [S1, unofficial]. The quadratic and cubic terms involve convolutions with the full volume, which are computationally expensive [S1, unofficial]. The source notes that these terms must be recomputed if the light direction changes [S1, unofficial].
*   **Memory Complexity:** Storage optimization uses twenty 8-bit sample basis images for the geometric terms and a $20 \times 20$ matrix for lighting coefficients [S1, unofficial].
*   **Measured Performance:** For a $128^3$ synthetic sphere: Traditional FVR is 36 fps (no depth cueing) and 27 fps (with depth cueing) [S1, unofficial]. Linear and cubic illuminations are 32 fps (no depth cueing) and 25 fps (with depth cueing) [S1, unofficial].
*   **Strengths:** Higher accuracy than linear hemispherical shading by better approximating the $\max$ function [S1, unofficial]. Reduces erroneous energy on surfaces back-facing the light [S1, unofficial][S2]. Produces images visually almost indistinguishable from normal diffuse shading [S2]. Allows re-evaluation for changing lights via linear combinations of pre-computed geometric terms [S2].
*   **Limitations:** Lower frame rates compared to linear shading [S1, unofficial]. Requires recomputation of convolution terms for changing light directions [S1, unofficial]. Not fully linear; the polynomial coefficients depend on the light direction [S1, unofficial].

## Background
*   **Origin:** GCHS was proposed by Scoggins et al. [S1, unofficial][S2] as an improvement over the hemispherical shading method by Totsuka and Levoy [S1, unofficial][S2].
*   **Development:** Totsuka and Levoy replaced the non-linear $\max(0, \mathbf{N} \cdot \mathbf{L})$ with the linear hemispherical term $(1 + \mathbf{N} \cdot \mathbf{L})/2$, which allowed Fourier projection but resulted in poor image quality (reduced contrast, negative intensities) [S1, unofficial]. Scoggins et al. raised this term to the cube ($g=3$) to better approximate the max operator, culling back-lit surfaces more effectively [S1, unofficial]. Entezari et al. adapted this method for Fourier Volume Rendering, providing the Fourier domain implementation formulas [S1, unofficial][S2].
*   **Usage:** Used in Fourier Volume Rendering to incorporate better illumination models, specifically for constant light sources where the convolution terms can be precomputed [S1, unofficial].

## Sources
[S1] PDF version2-reza.fm - Scientific Computing and Imaging Institute, https://www.sci.utah.edu/~jmk/papers/others/vis02_forierVolumeShading.pdf, fetched 2026-10-27 (unofficial)
[S2] Shading For Fourier Volume Rendering, https://eprints.cs.univie.ac.at/5034/1/2002_-_shading_for_fourier.pdf, fetched 2026-10-27
