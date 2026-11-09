# Gamma-Corrected Hemispherical Shading

Also called: Cubic Shading in Fourier Domain, Fourier Domain Gamma-Corrected Shading, Gamma-Corrected Shading

## Overview
Gamma-Corrected Hemispherical Shading (GCHS) is a method for computing diffuse illumination in Fourier Volume Rendering (FVR) that approximates the non-linear `max(0, N·L)` operator with a cubic polynomial of the gradient and light direction, enabling efficient evaluation in the frequency domain [S1, unofficial]. It improves upon standard hemispherical shading by raising the term to the power of three, which effectively culls back-lit surfaces and preserves visual contrast without requiring non-linear operations that break Fourier linearity [S1, unofficial]. The method builds on [Implementation Details for Fourier Volume Rendering](846-implementation-details-for-fourier-volume-rendering.md) and differs from [Adaptive Frequency Domain Filtering](968-adaptive-frequency-domain-filtering.md) and [FDK Pre-Filtering (Ramp Filter + Cone Weighting)](5010-fdk-pre-filtering-ramp-filter-cone-weighting.md) by operating directly on polynomial expansions of the shading operator in the frequency domain.

## How it works
### Inputs and Outputs
- **Input**: Volume data $f(x,y,z)$, light direction vector $L = (l_1, l_2, l_3)$ (normalized) [S1, unofficial].
- **Output**: Shaded volume intensity image [S1, unofficial].

### Data Structures
- **Frequency Domain Data**: $F(\omega_x, \omega_y, \omega_z)$, the 3D Fourier transform of $f$ [G].
- **Frequency Coordinates**: $\Omega = (\omega_x, \omega_y, \omega_z)$ [G].
- **Storage Optimization**: To reduce memory usage for the expanded polynomial terms, the method utilizes twenty 8-bit sample basis images for the geometric terms $V_i$ and a $20 \times 20$ matrix derived from corresponding lighting directions for the coefficients $a_i$ [S1, unofficial].

### Mathematical Formulation
The base hemispherical shading operator is defined as [S1, unofficial]:
$$ E = \frac{1}{2}(1 + N \cdot L) $$
where $N$ is the normalized normal vector and $L$ is the normalized light vector [S1, unofficial]. Ignoring visibility in this linear form results in negative intensity values and reduced contrast, resembling overcast lighting [S1, unofficial].

To improve contrast and cull back-lit surfaces, GCHS raises the hemispherical term to a gamma power $g$ [S1, unofficial]. The gamma-corrected shading operator is [S1, unofficial]:
$$ E = \left( \frac{1 + N \cdot L}{2} \right)^g $$
The exponent $g$ is typically set to 3 [S1, unofficial]. This value is chosen because it is the lowest integer power that produces an inflection point, allowing the curve to pass through both 0 and 1, thereby minimizing error in both zero and non-zero portions of the shading curve [S1, unofficial].

In the context of volume rendering, the normal $N$ is approximated by the gradient $\nabla f$ [G]. The term $(1 + N \cdot L)^3$ is expanded into a polynomial of dot products between the gradient and the light direction, normalized by the gradient magnitude $\|\nabla f\|$ [S1, unofficial]:
$$ (1 + N \cdot L)^3 = \left(1 + 3 \frac{\nabla f \cdot L}{\|\nabla f\|} + 3 \frac{(\nabla f \cdot L)^2}{\|\nabla f\|^2} + \frac{(\nabla f \cdot L)^3}{\|\nabla f\|^3}\right) $$
This expansion results in twenty (20) lighting and geometry terms when fully expanded [S1, unofficial]. The shading is computed by implementing each term of the expanded polynomial separately as convolutions in the Fourier domain [S1, unofficial].

### Implementation Steps
The implementation proceeds by computing the linear, quadratic, and cubic components of $\nabla f \cdot L$ in the Fourier domain [S1, unofficial].

1. **Compute the Linear Term**: The linear term is $\nabla f \cdot L$ [S1, unofficial]. Differentiation in the spatial domain corresponds to multiplication by a linear ramp in the Fourier domain [S1, unofficial]. Let $F$ be the Fourier transform of $f$ [G]. The Fourier transform of the linear term is [S1, unofficial]:
   $$ T\{\nabla f \cdot L\} = (\Omega \cdot L) F $$
   Explicitly, if $\Omega = (\omega_x, \omega_y, \omega_z)$, then:
   $$ T\{\nabla f \cdot L\} = (\omega_x l_1 + \omega_y l_2 + \omega_z l_3) F $$

2. **Compute the Quadratic Term**: The quadratic term is $(\nabla f \cdot L)^2$ [S1, unofficial]. This requires computing products of derivatives [G]. Using the identity for the product of two functions in the frequency domain:
   $$ f_a f_b = \frac{1}{2}(\mathcal{F}^2)_{ab} - f f_{ab} $$
   where $(\mathcal{F}^2)_{ab}$ represents the convolution of the Fourier transforms of the derivatives, and $f_{ab}$ is the second derivative [S1, unofficial]. The Fourier transform of the quadratic term is [S1, unofficial]:
   $$ T\{(\nabla f \cdot L)^2\} = \frac{1}{2}(\Omega \cdot L)^2(F \otimes F) - (F \otimes ((\Omega \cdot L)^2 F)) $$
   Here, $\otimes$ denotes convolution [S1, unofficial].

3. **Compute the Cubic Term**: The cubic term is $(\nabla f \cdot L)^3$ [S1, unofficial]. Using the identity for the product of three functions:
   $$ f_a f_b f_c = \frac{1}{6}(f^3)_{abc} - \frac{1}{2}f(f^2)_{abc} + \frac{1}{2}f^2 f_{abc} $$
   The Fourier transform of the cubic term is [S1, unofficial]:
   $$ T\{(\nabla f \cdot L)^3\} = \frac{1}{6}(\Omega \cdot L)^3(F \otimes F \otimes F) - \frac{1}{2}(F \otimes ((\Omega \cdot L)^3(F \otimes F))) + \frac{1}{2}(F \otimes F \otimes ((\Omega \cdot L)^3 F)) $$

4. **Normalization**: The terms derived above assume unnormalized gradients [G]. The original expansion includes normalization by $\|\nabla f\|$ [S1, unofficial]. The sources note that normalization with $\frac{1}{\|\nabla f\|}$ can be precomputed [S1, unofficial]. However, the exact Fourier-domain implementation of the division by $\|\nabla f\|$ is not explicitly detailed in the provided equations, which focus on the polynomial expansion of the dot products [S1, unofficial]. In practice, this normalization is handled by precomputing the basis images [S1, unofficial].

5. **Assembly**: Combine the linear, quadratic, and cubic components according to the expansion coefficients (1, 3, 3, 1) and scale by the hemispherical bias factor $\frac{1}{2}$ [S1, unofficial]. The constant term $1$ corresponds to a DC offset in the frequency domain [G].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gamma_corrected_hemispherical_shading(F, Omega, L):
    """
    Compute GCHS shading in the Fourier domain.
    
    Args:
        F: Complex array, Fourier transform of volume f.
        Omega: Tuple of arrays (wx, wy, wz), frequency coordinates.
        L: Tuple of floats (l1, l2, l3), normalized light direction.
        
    Returns:
        Shaded intensity in Fourier domain (before inverse transform).
    """
    l1, l2, l3 = L
    wx, wy, wz = Omega
    omega_dot_L = wx * l1 + wy * l2 + wz * l3
    
    # 1. Linear Term: T{grad f . L}
    linear_term = omega_dot_L * F
    
    # 2. Quadratic Term: T{(grad f . L)^2}
    # Identity: fa fb = 0.5 * (Fa * Fb) - F Fab
    conv_F_F = convolve_3d(F, F)
    second_deriv_F = (omega_dot_L ** 2) * F
    
    term_A = 0.5 * (omega_dot_L ** 2) * conv_F_F
    term_B = convolve_3d(F, second_deriv_F)
    quadratic_term = term_A - term_B
    
    # 3. Cubic Term: T{(grad f . L)^3}
    # Identity: fa fb fc = 1/6(f^3) - 1/2 f(f^2) + 1/2 f^2 f
    conv_F_F_F = convolve_3d(conv_F_F, F)
    omega_dot_L_3 = omega_dot_L ** 3
    
    term_C = (1.0 / 6.0) * omega_dot_L_3 * conv_F_F_F
    
    term_D_inner = omega_dot_L_3 * conv_F_F
    term_D = 0.5 * convolve_3d(F, term_D_inner)
    
    third_deriv_F = omega_dot_L_3 * F
    term_E_inner = convolve_3d(F, third_deriv_F)
    term_E = 0.5 * convolve_3d(F, term_E_inner)
    
    cubic_term = term_C - term_D + term_E
    
    # 4. Assembly
    # Expansion: 1 + 3*(grad.L)/|grad| + 3*(grad.L)^2/|grad|^2 + (grad.L)^3/|grad|^3
    # The constant 1 is a DC offset (delta at origin in frequency domain) [G].
    # Normalization by |grad| is handled via precomputed basis images [S1, unofficial].
    # Combine polynomial components:
    shaded_signal = linear_term + 3.0 * quadratic_term + cubic_term
    
    return shaded_signal

def convolve_3d(A, B):
    """
    3D Convolution in the frequency domain.
    Implements: A ⊗ B = FFT(IFFT(A) * IFFT(B))
    """
    spatial_a = ifft3d(A)
    spatial_b = ifft3d(B)
    return fft3d(spatial_a * spatial_b)

def fft3d(X):
    """Standard 3D discrete Fourier transform [G]."""
    # Implementation depends on the host language's standard FFT library.
    pass

def ifft3d(X):
    """Standard 3D inverse discrete Fourier transform [G]."""
    # Implementation depends on the host language's standard FFT library.
    pass
```

### Edge Cases and Pitfalls
- **Light Direction Changes**: The equations require convolution with a full volume [S1, unofficial]. This can only be precomputed for a constant light source [S1, unofficial]. If the light direction changes, the terms in the quadratic and cubic equations must be recomputed [S1, unofficial].
- **Normalization**: The expansion includes normalization by $\|\nabla f\|$ [S1, unofficial]. The sources note this can be precomputed as basis images, but the exact Fourier-domain division is not explicitly defined in the core convolution formulas [S1, unofficial].
- **Negative Intensity**: Ignoring visibility (using simple hemispherical shading without gamma correction) results in negative image intensity values and reduced contrast [S1, unofficial].

## Operating characteristics
- **Time Complexity**: The method involves convolutions in the Fourier domain, which are computationally expensive [S1, unofficial]. The rendering speed for a $128^3$ synthetic sphere is 32fps (no depth cueing) and 25fps (with depth cueing) for cubic illumination, compared to 36fps (no depth cueing) and 27fps (with depth cueing) for traditional FVR [S1, unofficial].
- **Memory Complexity**: The method can be optimized using twenty 8-bit sample basis images and a $20 \times 20$ matrix, reducing the memory required for surface normal data [S1, unofficial].
- **Strengths**: Higher accuracy than linear shading by better approximating the `max` function, reducing erroneous energy on back-facing surfaces [S1, unofficial].
- **Limitations**: Lower frame rates than linear shading due to additional calculations (convolutions) [S1, unofficial]. Not suitable for dynamic light directions without recomputation [S1, unofficial].

## Background
Gamma-Corrected Hemispherical Shading was introduced by Scoggins et al. as an improvement over the hemispherical shading method by Totsuka and Levoy [S1, unofficial]. The method was adapted for Fourier Volume Rendering by Entezari, Scoggins, Möller, and Machiraju [S1, unofficial]. The key insight was that culling back-lit surfaces is equivalent to using the maximum operator, and raising the hemispherical term to the cube approximates this behavior while remaining expandable into a polynomial suitable for Fourier domain processing [S1, unofficial].

## Sources
[S1] PDF version2-reza.fm - Scientific Computing and Imaging Institute, https://www.sci.utah.edu/~jmk/papers/others/vis02_forierVolumeShading.pdf, fetched 2026-10-27 (unofficial)
