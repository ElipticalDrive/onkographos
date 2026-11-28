# Hartley Transform
Also called: Real Fourier Transform

## Overview
The Hartley Transform is a real-valued integral transform that maps a real input function to a real spectrum, serving as an alternative to the complex Discrete Fourier Transform (DFT) for real-valued data [S1, unofficial]. In volume rendering, it enables domain-transform methods (such as Fourier volume rendering) to operate entirely in the real domain, avoiding complex arithmetic while preserving the projection-slice theorem [S1, 6324-hartley-transform.md, unofficial]. It is distinguished from its sibling [Fourier Projection-Slice Volume Rendering](824-fourier-projection-slice-volume-rendering.md) by using the Hartley kernel ($\cos + \sin$) instead of the complex exponential ($\cos + i\sin$), which simplifies implementation for real-valued signals and reduces memory overhead for intermediate representations [S1, 895-hartley-transform-for-memory-reduction.md, unofficial].

## How it works
The method follows the three-stage pipeline of [Domain-transform methods](28-domain-transform-methods.md): Global Transformation, Slice Extraction, and Inverse Transformation [S1, 824-fourier-projection-slice-volume-rendering.md, 895-hartley-transform-for-memory-reduction.md, 5396-dct-based-volume-rendering.md, unofficial].

### Inputs and Outputs
*   **Input:** A 3D scalar volume $f(x, y, z)$ with dimensions $N_x \times N_y \times N_z$.
*   **Output:** A 2D projection image $I(u, v)$ corresponding to a viewing direction.

### Data Structures
*   **Volume:** 3D array of real numbers.
*   **Spectrum:** 3D array of real numbers $H(\omega_1, \omega_2, \omega_3)$.
*   **Slice:** 2D array of real numbers $I(u, v)$.

### Steps

1.  **Preprocessing:** Apply zero-padding or vignetting correction to mitigate boundary artifacts caused by periodic boundary assumptions [S3, 841-gpu-based-2d-fft.md, 952-frequency-domain-volume-rendering.md]. Shift the spatial origin to the center of the volume (e.g., via `fftshift` or multiplication by $(-1)^{x+y+z}$) to center the zero-frequency component [S12, 2288-fourier-transform.md, 952-frequency-domain-volume-rendering.md].

2.  **Global Transformation (3D DHT):** Compute the 3D Discrete Hartley Transform (DHT) of the preprocessed volume to obtain the 3D spectrum $H(\omega_1, \omega_2, \omega_3)$. The 1D DHT for a sequence $h_k$ of length $N$ is defined as [S1, unofficial]:
    $$H_n = \sum_{k=0}^{N-1} h_k \cdot \text{cas}\left(\frac{2\pi nk}{N}\right)$$
    where $\text{cas}(x) = \cos(x) + \sin(x)$ [S1, unofficial]. The 3D transform is separable, applied along each dimension.

3.  **Slice Extraction:** For a viewing direction vector $V$, define a 2D plane passing through the origin of the frequency domain and perpendicular to $V$ [S3, 824-fourier-projection-slice-volume-rendering.md, 6324-hartley-transform.md]. Resample the 3D spectrum $H$ onto this 2D plane to obtain the 2D slice $I(u, v)$ [S1, unofficial][S2]. Interpolation (e.g., Trilinear, Windowed Sinc) is required as the plane rarely coincides with the integer lattice [S3, 824-fourier-projection-slice-volume-rendering.md, 6324-hartley-transform.md]. The output resolution must satisfy the Nyquist criterion [S2, 952-frequency-domain-volume-rendering.md, 5396-dct-based-volume-rendering.md].

4.  **Inverse Transformation (2D IDHT):** Compute the 2D Inverse Discrete Hartley Transform of the slice $I(u, v)$ to generate the final projection image [S1, 6324-hartley-transform.md, unofficial][S2]. The 1D IDHT is defined as [S1, unofficial]:
    $$h_k = \frac{1}{N} \sum_{n=0}^{N-1} H_n \cdot \text{cas}\left(\frac{2\pi nk}{N}\right)$$
    The operation is identical to the forward DHT, scaled by $N^{-1}$ [G].

### Conversion to/from Fourier Transform
If complex Fourier coefficients are needed, they can be derived from the Hartley spectrum $H(f)$ using its even $E(f)$ and odd $O(f)$ parts [S1, unofficial]:
$$E(f) = \frac{H(f) + H(-f)}{2}, \quad O(f) = \frac{H(f) - H(-f)}{2}$$
$$S(f) = E(f) + iO(f)$$
Conversely, $H(f) = \text{Re}[S(f)] + \text{Im}[S(f)]$ [S1, unofficial].

### Parameters
*   **Interpolation:** Nearest Neighbor, Trilinear, Tri-cubic, or Windowed Sinc [S2, 952-frequency-domain-volume-rendering.md].
*   **Scaling:** $1/N$ for inverse transform [S1, 952-frequency-domain-volume-rendering.md, unofficial].

### Edge Cases and Pitfalls
*   **Boundary Artifacts:** The DHT assumes periodicity. Without zero-padding or windowing, discontinuities at volume boundaries cause high-frequency artifacts [S3, 841-gpu-based-2d-fft.md, 952-frequency-domain-volume-rendering.md].
*   **Aliasing:** Insufficient resolution in the slice extraction step violates the Nyquist criterion, causing aliasing [S2, 952-frequency-domain-volume-rendering.md, 5396-dct-based-volume-rendering.md].
*   **Precision:** Real-valued arithmetic avoids complex number overhead but requires careful handling of the even/odd symmetry for conversion to Fourier domain if needed [S1, unofficial].

## Subtopics

### [Discrete Hartley Transform](6335-discrete-hartley-transform.md)
The Discrete Hartley Transform (DHT) is a real-valued linear transform that maps a sequence of $N$ real numbers to $N$ real numbers, serving as an alternative to the complex Discrete Fourier Transform (DFT) for real input data. It is distinguished from its sibling, the Hartley Transform, by operating on discrete sequences rather than continuous functions, and from the DFT by avoiding complex arithmetic, which simplifies implementation for real-valued signals. In volume rendering, it is used for domain-transform methods where real-valued convolution theorems are advantageous.

### Hartley Projection Slice Theorem
A theorem stating that the 1-D Hartley transform of a projection of an object at angle $\theta$ equals a slice through the 2-D Hartley transform of the object [S1, unofficial]. The projection $p_\theta(\hat{u}_1)$ is defined as [S1, unofficial]:
$$p_\theta(\hat{u}_1) = \int_{-\infty}^{\infty} x(\hat{u}_1 \cos \theta - \hat{u}_2 \sin \theta, \hat{u}_1 \sin \theta + \hat{u}_2 \cos \theta) d\hat{u}_2$$
The theorem is expressed as [S1, unofficial]:
$$P_\theta(\hat{U}_1) = H(\hat{U}_1 \cos \theta, \hat{U}_1 \sin \theta)$$
where $P_\theta$ is the Hartley transform of the projection and $H$ is the 2-D Hartley transform of the object $x(u_1, u_2)$ [S1, unofficial]. This validates the use of the Hartley transform in tomographic reconstruction and volume rendering pipelines [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the 3D DHT and 2D IDHT, typically $O(N^3 \log N)$ for FFT-like algorithms or $O(N^3)$ for direct DHT [S1, 6324-hartley-transform.md, unofficial].
*   **Memory Complexity:** $O(N^3)$ for the 3D spectrum. Since the Hartley transform is real-valued, it uses half the memory of a complex DFT for the same data size [S1, 895-hartley-transform-for-memory-reduction.md, unofficial].
*   **Performance:** Real-valued arithmetic is generally faster and less memory-intensive than complex arithmetic [S1, 895-hartley-transform-for-memory-reduction.md, unofficial].
*   **Strengths:** Avoids complex number overhead; maintains real-valued data throughout the pipeline; supports the projection-slice theorem [S1, 895-hartley-transform-for-memory-reduction.md, unofficial].
*   **Limitations:** Less mature library support compared to FFT; requires careful handling of boundary conditions [S1, 895-hartley-transform-for-memory-reduction.md, unofficial].
*   **When to use:** When memory bandwidth is a bottleneck or when complex arithmetic is undesirable [S1, 895-hartley-transform-for-memory-reduction.md, unofficial].
*   **When not to use:** When complex-valued data is inherent or when standard FFT libraries are preferred for performance optimization [S1, 895-hartley-transform-for-memory-reduction.md, unofficial].

## Background
The Hartley transform was introduced by R.V.L. Hartley in 1942 as a real-valued alternative to the Fourier transform [S1, unofficial]. It has been applied in computer-aided tomographic (CAT) scanners and other signal processing applications [S1, unofficial]. In volume rendering, it has been explored for memory reduction and efficient domain-transform methods [S1, 895-hartley-transform-for-memory-reduction.md, 6324-hartley-transform.md, unofficial].

## Sources
[S1] Fourier volume rendering, https://shiftleft.com/mirrors/www.hpl.hp.com/techreports/95/HPL-95-73.pdf, fetched 2026-10-31 (unofficial)
