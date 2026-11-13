# Fourier Transform
Also called: Discrete Fourier Transform, DFT

## Overview
The Fourier Transform method converts a 3D scalar volume into the frequency domain to enable rendering via the Projection-Slice Theorem [Domain-transform methods](28-domain-transform-methods.md). It differs from spatial-domain methods by performing a global transformation of the entire volume, which allows for efficient convolution operations but requires significant preprocessing memory and does not support interactive parameter changes like opacity or lighting [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume $f(x, y, z)$ of dimensions $N_x \times N_y \times N_z$ [S2, unofficial].
*   **Output:** A 2D projection image $I(u, v)$ [Domain-transform methods](28-domain-transform-methods.md).

**Data Structures:**
*   **Volume Data:** A 3D array of real numbers representing the spatial domain [S2, unofficial].
*   **Frequency Data:** A 3D array of complex numbers representing the frequency domain. For real input, the output is Hermitian symmetric, meaning $H_{-u,-v,-w} = H_{u,v,w}^*$, allowing storage of only half the data if optimized [S2, unofficial].

**Parameters:**
*   $N$: Number of samples along a dimension. Assumed to be even for standard frequency point derivation [S2, unofficial].
*   $\Delta$: Sampling interval (spacing between voxels) [S2, unofficial].
*   $f_c$: Nyquist critical frequency, defined as $1/(2\Delta)$ [S2, unofficial].

**Steps:**
1.  **Preprocessing (Centering):** To center the zero-frequency component in the middle of the spectrum and avoid boundary discontinuities in visualization, multiply the input volume by $(-1)^{x+y+z}$ [S2, unofficial]. The source explicitly defines this shift for 1D signals as multiplying by $(-1)^k$ and for 2D signals as multiplying by $(-1)^{x+y}$; the 3D extension follows the same pattern [G].
2.  **Global Transformation (3D DFT):** Compute the 3D Discrete Fourier Transform of the volume. For a 1D sequence $h_k$ of length $N$, the DFT $H_n$ is defined as [S2, unofficial]:
    $$H_n = \sum_{k=0}^{N-1} h_k e^{2\pi i k n/N}$$
    For a 3D volume $f(x, y, z)$, the 3D DFT $F(u, v, w)$ is computed as [S2, unofficial]:
    $$F(u, v, w) = \sum_{x=0}^{N_x-1} \sum_{y=0}^{N_y-1} \sum_{z=0}^{N_z-1} f(x, y, z) e^{2\pi i (ux/N_x + vy/N_y + wz/N_z)}$$
    The source defines the forward DFT with a positive exponent in Eq. 2.13 [S2, unofficial].
3.  **Slice Extraction:** According to the Projection-Slice Theorem, the 2D Fourier transform of a projection of the volume along a viewing direction is equal to a 2D slice through the origin of the 3D Fourier transform, perpendicular to the viewing direction [Domain-transform methods](28-domain-transform-methods.md). Extract a 2D plane from the 3D frequency volume $F(u, v, w)$ corresponding to the viewing direction. The extracted slice lies on a grid that may not align with the discrete 3D lattice of $F$. Interpolation (e.g., trilinear or sinc) is required to resample the frequency data onto the 2D output grid [Domain-transform methods](28-domain-transform-methods.md).
4.  **Inverse Transformation (2D IDFT):** Compute the 2D Inverse Discrete Fourier Transform of the extracted slice to obtain the spatial domain projection image [Domain-transform methods](28-domain-transform-methods.md). The exact normalization factor ($1/N$ or $1/\sqrt{N}$) depends on the specific DFT definition used [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def fourier_transform_volume_rendering(volume, view_direction, output_shape):
    """
    Renders a 3D volume using the Fourier Transform method.
    
    Args:
        volume: 3D numpy array (Nx, Ny, Nz) of real values.
        view_direction: 3D vector (vx, vy, vz) specifying the viewing angle.
        output_shape: Tuple (Ou, Ov) for the 2D output image dimensions.
        
    Returns:
        2D numpy array (Ou, Ov) of real values representing the projection image.
    """
    Nx, Ny, Nz = volume.shape
    
    # Step 1: Preprocessing - Center the zero frequency
    x = np.arange(Nx)
    y = np.arange(Ny)
    z = np.arange(Nz)
    X, Y, Z = np.meshgrid(x, y, z, indexing='ij')
    centered_volume = volume * ((-1) ** (X + Y + Z))
    
    # Step 2: Global Transformation - 3D DFT
    # Using numpy's FFT which uses the standard engineering convention (negative exponent for forward).
    # To match the source's positive exponent definition, one would conjugate the result,
    # but for magnitude-based rendering, standard FFT is sufficient.
    freq_volume = np.fft.fft3(centered_volume)
    
    # Step 3: Slice Extraction
    # Define the plane in frequency domain perpendicular to view_direction.
    # The plane passes through the origin (0,0,0) in the centered frequency domain.
    # Generate 2D grid coordinates in the plane perpendicular to view_direction.
    # Interpolation is required to resample onto this grid.
    # The sources do not provide the resampling algorithm. [G]
    # In practice, one would define a 2D grid in the plane perpendicular to view_direction
    # and interpolate the values from the 3D frequency volume.
    slice_2d = np.zeros(output_shape, dtype=np.complex128)
    
    # Step 4: Inverse Transformation - 2D IDFT
    # Apply 2D inverse FFT
    image_complex = np.fft.ifft2(slice_2d)
    
    # Return the real part of the image
    return np.real(image_complex)
```

**Edge Cases and Pitfalls:**
*   **Boundary Artifacts:** The DFT assumes periodic boundary conditions. Discontinuities at the volume boundaries cause high-frequency artifacts. Mitigation includes zero-padding the volume by a factor of $k_2$ (typically $4/3$) and applying vignetting correction (premultiplying by the reciprocal of the interpolation filter response) [Domain-transform methods](28-domain-transform-methods.md).
*   **Aliasing:** If the output image resolution is too low, aliasing artifacts will appear. The resolution must meet the Nyquist criterion relative to the volume's bandwidth [Domain-transform methods](28-domain-transform-methods.md).
*   **Hermitian Symmetry:** For real-valued input, the DFT output is Hermitian symmetric. This can be exploited to reduce memory usage by storing only half the complex coefficients [S2, unofficial].

## Subtopics

### [Windowed Fourier Transform](6084-windowed-fourier-transform.md)
The Windowed Fourier Transform (WFT), also known as the Gabor Transform, addresses the lack of spatial localization inherent in the standard Fourier Transform. While the standard Fourier Transform provides a global frequency measure by integrating over the entire spatial domain, making it difficult to locate irregularities, the WFT uses a translated window function to cover the signal, allowing for the localization of these irregularities. It builds on [Fourier Transform](2288-fourier-transform.md) by introducing spatial constraints to the frequency analysis.

## Operating characteristics
*   **Time Complexity:** The 3D DFT using FFT algorithms has a complexity of $O(N^3 \log N)$ for a volume of size $N^3$. The 2D IDFT for an image of size $M \times M$ has a complexity of $O(M^2 \log M)$ [Domain-transform methods](28-domain-transform-methods.md).
*   **Memory Complexity:** The 3D frequency volume requires $O(N^3)$ memory [Domain-transform methods](28-domain-transform-methods.md).
*   **Performance:** The method is computationally intensive due to the global 3D transform. It is suitable for static volumes where the viewing direction changes frequently, as the 3D transform is performed once [S1, unofficial].
*   **Strengths:** Efficient for multiple viewing angles once the 3D transform is computed.
*   **Limitations:** High memory usage. Lack of interactivity for display parameters (opacity, lighting) [S1, unofficial]. Boundary artifacts require careful handling [Domain-transform methods](28-domain-transform-methods.md).

## Background
The Fourier Transform is a classical theory for multiresolution decomposition [S1, unofficial]. It provides a measure of high frequency but information is not spatially localized, as the transform covers the whole spatial domain, making it difficult to find the position of irregularities [S1, unofficial].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S2] Fourier volume rendering, https://shiftleft.com/mirrors/www.hpl.hp.com/techreports/95/HPL-95-73.pdf, fetched 2026-10-31 (unofficial)
