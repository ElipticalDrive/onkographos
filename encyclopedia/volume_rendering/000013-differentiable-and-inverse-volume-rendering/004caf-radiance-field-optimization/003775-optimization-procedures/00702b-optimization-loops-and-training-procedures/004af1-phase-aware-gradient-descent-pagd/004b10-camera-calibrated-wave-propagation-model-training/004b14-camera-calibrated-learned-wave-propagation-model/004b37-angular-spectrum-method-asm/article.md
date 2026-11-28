# Angular Spectrum Method (ASM)

Also called: Fourier Optics Propagation, Free-space propagation operator, Helmholtz equation solution in Fourier space

## Overview
The Angular Spectrum Method (ASM) is a free-space propagation operator used to model the parallel propagation of a wave field from one plane to another [S3]. It solves the problem of accurately reconstructing or propagating a complex optical field $U_0(x_0, y_0)$ to a target plane at distance $z$ by treating the field as a superposition of plane waves propagating in different directions [S1]. Unlike the Fresnel Method, which relies on paraxial approximations, ASM operates in the frequency domain to solve the Helmholtz equation without approximations, avoiding the computational cost of first-principles methods like FDTD [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The complex optical field $U_0(x_0, y_0)$ at the source plane ($z=0$), the propagation distance $z$, and the wavelength $\lambda$ [S1][S2, unofficial].
*   **Output:** The reconstructed complex wavefield $U_r(x, y)$ at the target plane ($z$) [S1].

**Data Structures:**
*   The input field is represented as a 2D grid of complex numbers [G].
*   The angular spectrum is represented as a 2D grid of complex numbers in the frequency domain $(k_x, k_y)$ [G].
*   The transfer function is a 2D grid of complex numbers with the same dimensions as the angular spectrum [G].

**Numbered Steps and Formulas:**
1.  **Fourier Transform:** Compute the 2D Fourier Transform of the input field $U_0(x_0, y_0)$ to obtain the Angular Spectrum $A_0(k_x, k_y)$ [S1].
    $$ A_0(k_x, k_y) = \mathcal{F} \{ U_0(x_0, y_0) \} $$
2.  **Apply Transfer Function:** Multiply the Angular Spectrum $A_0(k_x, k_y)$ by the propagation phase factor (transfer function) $H(k_x, k_y; z)$ [S1].
    $$ H(k_x, k_y; z) = \exp \left( j k_z z \right) $$
    where
    $$ k_z = \sqrt{k^2 - k_x^2 - k_y^2} $$
    The source assumes $k^2 \ge k_x^2 + k_y^2$ [S1].
3.  **Inverse Fourier Transform:** Compute the 2D Inverse Fourier Transform of the product to obtain the reconstructed wavefield $U_r(x, y)$ [S1].
    $$ U_r(x, y) = \mathcal{F}^{-1} \left\{ A_0(k_x, k_y) \exp \left( j \sqrt{k^2 - k_x^2 - k_y^2} z \right) \right\} $$

**Parameters:**
*   $\lambda$: Wavelength of the light [S2, unofficial].
*   $k = \frac{2\pi}{\lambda}$: Wavenumber [S1][S2, unofficial].
*   $z$: Propagation distance [S1].
*   $\Delta x, \Delta y$: Spatial sampling intervals in the source plane [G].
*   $\Delta k_x, \Delta k_y$: Frequency sampling intervals, where $\Delta k_x = \frac{2\pi}{N_x \Delta x}$ and $\Delta k_y = \frac{2\pi}{N_y \Delta y}$ for an $N_x \times N_y$ grid [G].
*   Typical values: Not given in the sources [S1][S2, unofficial][S3, S4].

**Edge Cases and Pitfalls:**
*   **Evanescent Waves:** If $k_x^2 + k_y^2 > k^2$, $k_z$ becomes imaginary, causing exponential decay of high-frequency components [S1]. In practice, these components are handled via complex $k_z$ or zeroed out [G].
*   **Sampling:** The spatial sampling $\Delta x, \Delta y$ must satisfy the Nyquist criterion for the highest spatial frequency present in the field to avoid aliasing [G]. The frequency grid must cover the relevant bandwidth [G].
*   **Computational Cost:** ASM requires two 2D Fourier Transforms, making it more computationally intensive than the Fresnel Method [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def angular_spectrum_method(U0, z, wavelength, dx, dy):
    """
    Propagates a complex optical field U0 using the Angular Spectrum Method.
    
    Args:
        U0: 2D numpy array (complex128) representing the input field.
        z: float, propagation distance.
        wavelength: float, wavelength of light.
        dx: float, spatial sampling interval in x.
        dy: float, spatial sampling interval in y.
        
    Returns:
        Ur: 2D numpy array (complex128) representing the propagated field.
    """
    Ny, Nx = U0.shape
    k = 2 * np.pi / wavelength
    
    # Create frequency grid (angular frequencies in rad/m)
    kx = np.fft.fftfreq(Nx, d=dx) * 2 * np.pi
    ky = np.fft.fftfreq(Ny, d=dy) * 2 * np.pi
    
    # Create 2D meshgrid
    KX, KY = np.meshgrid(kx, ky, indexing='xy')
    
    # Calculate kz
    kz_sq = k**2 - KX**2 - KY**2
    
    # Handle evanescent waves: if kz_sq < 0, kz is imaginary.
    # exp(j * kz * z) will decay exponentially.
    kz = np.sqrt(kz_sq)
    
    # Transfer function
    H = np.exp(1j * kz * z)
    
    # Step 1: Fourier Transform
    A0 = np.fft.fft2(U0)
    
    # Step 2: Multiply by transfer function
    A_propagated = A0 * H
    
    # Step 3: Inverse Fourier Transform
    Ur = np.fft.ifft2(A_propagated)
    
    return Ur
```

## Subtopics

### [Camera-Calibrated Differentiable Holographic Model](17687-camera-calibrated-differentiable-holographic-model.md)
A differentiable computational model that simulates the physical propagation of light from a Spatial Light Modulator (SLM) to a camera sensor, incorporating learnable parameters to account for system-specific imperfections such as polarization, SLM curvature, beam profiles, higher diffraction orders, crosstalk, and Lookup Table (LUT) non-linearity. It sets itself apart from standard Angular Spectrum Method (ASM) implementations by explicitly modeling the superposition of zero and first-order diffraction patterns through a neural network (U-Net) and calibrating these parameters against experimental data, thereby compensating for unmodeled physical effects in compact optical setups.

### [Modified High Order Angular Spectrum Method (HOASM)](17758-modified-high-order-angular-spectrum-method-hoasm.md)
HOASM is a differentiable wave propagation model that separates the propagation of zero-order and first-order diffraction terms to handle them differently before feeding them into a neural network. It builds on [Angular Spectrum Method (ASM)](12941-angular-spectrum-method-asm.md) by introducing a specific preprocessing step for the first-order term involving grid patterning and sinc weighting, distinguishing it from standard ASM which propagates all orders together.

### Fresnel Propagation
A paraxial approximation for propagating electric fields, used here as the propagation function $f_{prop}$ [S4]. It is nearly identical to the Angular Spectrum Method (ASM) for SLMs with pixel pitch > 3 µm and diffraction angles < 5° [S4]. Fresnel propagation in Fourier space is defined as $f_{fresnel}(g, z, \lambda) = \mathcal{F}^{-1} \{ \mathcal{F} \{g\} \cdot H(z, \lambda) \}$, where $\mathcal{F}$ is the 2D Fourier transform, $g$ is the input electric field, $z$ is the propagation distance, and $H$ is the Fresnel kernel [S4]. The Fresnel propagation kernel is $H(z, \lambda) = \exp \left( i \pi \lambda z (f_x^2 + f_y^2) \right)$, where $f_x, f_y$ are spatial frequency coordinates [S4]. A key pitfall is Wavelength-Depth Ambiguity: in the Fresnel kernel, $\lambda$ and $z$ appear as the product $\lambda z$, creating an ambiguity where an image formed at distance $z_0$ under red illumination ($\lambda_r$) will also appear at distances $z_0 \lambda_g / \lambda_r$ (green) and $z_0 \lambda_b / \lambda_r$ (blue), creating 'depth replicas' [S4].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1][S2, unofficial][S3, S4]. The method requires two 2D Fourier Transforms, making it more computationally intensive than the Fresnel Method [S1].
*   **Memory Complexity:** Not given in the sources [S1][S2, unofficial][S3, S4].
*   **Measured Performance:** Fourier Optics is computationally less taxing than first-principles FDTD simulations, which model dielectric refractive indices and Snell's Law explicitly, but Fourier Optics may not capture all physical details [S2, unofficial].
*   **Strengths:** Exact solution for free-space propagation without paraxial approximations [S1].
*   **Limitations:** Computationally more expensive than Fresnel or Fresnel-Kirchhoff approximations; sensitive to sampling rates; requires handling of evanescent waves [S1].
*   **When to use:** When high accuracy is required, especially for near-field propagation or large angles where Fresnel approximations fail [S1].
*   **When not to use:** When computational speed is critical and the paraxial approximation is valid (use Fresnel Method instead) [S1].

## Background
The method was developed as an exact solution to the Helmholtz equation for free-space propagation [S2, unofficial]. Specific origin details, authors, and dates are not given in the sources [S1][S2, unofficial][S3, S4]. It is cited in recent advances in processing and rendering algorithms for CGH [S1] and in mesh-represented hologram synthesis [S3].

## Sources
[S1] Recent Advances in the Processing and Rendering Algorithms for Computer-Generated Holography, https://doi.org/10.3390/electronics8050556, fetched 2026-10-26
[S2] Fourier Vases | Almost looks like work, https://jasmcole.com/2015/04/26/fourier-vases/, fetched 2026-10-29 (unofficial)
[S3] Mesh-represented and learning-empowered hologram synthesis for full 3D holographic displays, https://www.nature.com/articles/s41467-026-74557-0, fetched 2026-10-30
[S4] Simultaneous Color Computer Generated Holography, https://doi.org/10.48550/arxiv.2303.11287, fetched 2026-10-30
