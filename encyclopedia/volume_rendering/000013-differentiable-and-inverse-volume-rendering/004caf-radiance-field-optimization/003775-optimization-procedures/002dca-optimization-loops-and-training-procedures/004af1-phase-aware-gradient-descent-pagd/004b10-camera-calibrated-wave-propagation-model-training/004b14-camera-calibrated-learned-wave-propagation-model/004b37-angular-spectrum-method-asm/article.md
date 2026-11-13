# Angular Spectrum Method (ASM)

Also called: Free-space propagation operator

## Overview
The Angular Spectrum Method (ASM) is a free-space propagation operator used to model the parallel propagation of a wave field, resulting in a wavefront $f(x, y; z)$ [S2]. It serves as an alternative reconstruction method that calculates the wavefield by propagating plane waves in different directions without involving approximations [S1]. Unlike the Fresnel Method, which relies on paraxial approximations, ASM is exact for free-space propagation, making it suitable for high-resolution or non-paraxial scenarios, albeit at higher computational cost [S1].

## How it works
**Inputs and Outputs:**
- Input: Initial optical field $U_0(x_0, y_0)$ at the source plane ($z=0$) [S1].
- Output: Reconstructed wavefield $U_r(x, y)$ at the propagation distance $z$ [S1].
- Parameters: Wavelength $\lambda$, propagation distance $z$ [S1, S2].

**Data Structures:**
The method operates on 2D complex-valued grids representing the optical field. The Fourier domain representation uses spatial frequency coordinates $(k_x, k_y)$ or normalized frequencies $(u, v)$ [S1, S2].

**Steps:**
1. Compute the 2D Fourier Transform of the input optical field $U_0(x_0, y_0)$ to obtain the Angular Spectrum $A_0(k_x, k_y)$ [S1].
   $$ A_0(k_x, k_y) = \mathcal{F} \{ U_0(x_0, y_0) \} $$
   Note: The continuous definition involves a normalization factor of $\frac{1}{2\pi}$ depending on the Fourier transform convention used [S1].

2. Multiply the Angular Spectrum by the propagation phase factor (transfer function) $H(k_x, k_y; z)$ [S1, S2].
   $$ H(k_x, k_y; z) = \exp \left( j k_z z \right) $$
   where $k_z$ is the longitudinal wavenumber:
   $$ k_z = \sqrt{k^2 - k_x^2 - k_y^2} $$
   and $k = \frac{2\pi}{\lambda}$ is the total wavenumber [S1].

3. Compute the Inverse Fourier Transform of the product to obtain the reconstructed wavefield $U_r(x, y)$ [S1].
   $$ U_r(x, y) = \mathcal{F}^{-1} \left\{ A_0(k_x, k_y) \exp \left( j \sqrt{k^2 - k_x^2 - k_y^2} z \right) \right\} $$

**Combined Formula:**
$$ U_r(x, y) = \mathcal{F}^{-1} \left\{ \mathcal{F} \{ U_0 \} \exp \left( j \sqrt{k^2 - k_x^2 - k_y^2} z \right) \right\} $$ [S1]

**Parameters:**
- $\lambda$: Wavelength of the light [S1, S2].
- $z$: Propagation distance [S1, S2].
- $k_x, k_y$: Spatial frequency components [S1].

**Edge Cases and Pitfalls:**
- The method assumes $k^2 \ge k_x^2 + k_y^2$ [S1]. If $k_x^2 + k_y^2 > k^2$, $k_z$ becomes imaginary, representing evanescent waves which decay exponentially. Standard implementations often truncate these or handle them via complex arithmetic [S1].
- Computational intensity: ASM requires two Fourier Transforms, making it more computationally intensive than the Fresnel Method [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def angular_spectrum_method(U0, wavelength, distance, dx, dy):
    """
    Propagates an optical field U0 using the Angular Spectrum Method.
    
    Args:
        U0: 2D complex array representing the initial optical field.
        wavelength: Scalar float, wavelength lambda.
        distance: Scalar float, propagation distance z.
        dx: Scalar float, pixel size in x direction.
        dy: Scalar float, pixel size in y direction.
        
    Returns:
        Ur: 2D complex array representing the propagated field.
    """
    import numpy as np
    
    # Constants
    k = 2 * np.pi / wavelength
    
    # Get dimensions
    N, M = U0.shape
    
    # Create frequency grid
    # Note: np.fft.fftfreq returns frequencies in cycles per unit.
    # We need angular spatial frequencies kx, ky.
    # kx = 2 * pi * fx, where fx = n / (N * dx)
    fx = np.fft.fftfreq(N, d=dx)
    fy = np.fft.fftfreq(M, d=dy)
    
    # Create meshgrid for 2D frequencies
    KX, KY = np.meshgrid(2 * np.pi * fx, 2 * np.pi * fy, indexing='ij')
    
    # Calculate kz
    # kz = sqrt(k^2 - kx^2 - ky^2)
    # Handle evanescent waves: if kx^2 + ky^2 > k^2, kz is imaginary.
    # The formula exp(j * kz * z) handles this naturally if we use complex sqrt.
    kz_sq = k**2 - KX**2 - KY**2
    
    # Use complex square root to handle evanescent waves (negative real part inside sqrt)
    # j * sqrt(negative) = j * j * sqrt(positive) = -sqrt(positive)
    # This results in exponential decay.
    kz = np.sqrt(kz_sq)
    
    # Transfer function
    H = np.exp(1j * kz * distance)
    
    # Step 1: Fourier Transform
    A0 = np.fft.fft2(U0)
    
    # Step 2: Multiply by transfer function
    A_propagated = A0 * H
    
    # Step 3: Inverse Fourier Transform
    Ur = np.fft.ifft2(A_propagated)
    
    return Ur
```

## Subtopics

### [Fresnel Method](52-fresnel-method.md)
An approximation of wave propagation that is computationally less intensive than ASM but relies on the paraxial approximation. Choose ASM when high accuracy or non-paraxial propagation is required.

### [Rayleigh-Sommerfeld Diffraction](53-rayleigh-sommerfeld.md)
Another exact propagation method based on Green's functions, often more expensive than ASM due to convolution requirements. ASM is preferred for its efficiency via FFT.

## Operating characteristics
- **Time Complexity:** $O(N^2 \log N)$ due to the two 2D Fast Fourier Transforms (FFTs) [S1].
- **Memory Complexity:** $O(N^2)$ to store the input field, frequency grid, and transfer function [S1].
- **Performance:** More computationally intensive than the Fresnel Method because it requires two full Fourier Transforms rather than a single convolution or multiplication in the spatial domain [S1].
- **Strengths:** No approximations; exact for free-space propagation; handles large angles and near-field propagation effectively [S1].
- **Limitations:** High computational cost; sensitive to aliasing if the sampling rate is insufficient (Nyquist criterion must be met for the highest spatial frequencies) [S1].
- **When to use:** High-resolution hologram reconstruction, non-paraxial scenarios, or when exact phase information is critical [S1].
- **When not to use:** Real-time applications where speed is prioritized over exactness, or when paraxial approximations are valid (Fresnel Method is faster) [S1].

## Background
The Angular Spectrum Method was introduced as an alternative reconstruction method for digital holography that avoids the approximations inherent in the Fresnel Method [S1]. It is based on the principle that any optical field can be decomposed into a sum of plane waves, each propagating in a different direction [S1]. The method has been widely adopted in computer-generated holography and optical reconstruction due to its accuracy [S1, S2]. Recent advances have explored software implementations to improve computational efficiency, especially for high-resolution and color holograms [S1]. It is also used in mesh-represented and learning-empowered hologram synthesis as a fundamental propagation operator [S2].

## Sources
[S1] Recent Advances in the Processing and Rendering Algorithms for Computer-Generated Holography, https://doi.org/10.3390/electronics8050556, fetched 2026-10-26
[S2] Mesh-represented and learning-empowered hologram synthesis for full 3D holographic displays, https://www.nature.com/articles/s41467-026-74557-0, fetched 2026-10-30
