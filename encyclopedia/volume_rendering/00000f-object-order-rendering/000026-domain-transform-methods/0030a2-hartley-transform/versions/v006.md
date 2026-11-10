# Hartley Transform
Also called: Real Fourier Transform

## Overview
The Hartley Transform is a real-valued integral transform that maps a real input function to a real spectrum, serving as an alternative to the complex Discrete Fourier Transform (DFT) for real-valued data [S1, unofficial]. In volume rendering, it enables domain-transform methods to operate entirely in the real domain while preserving the projection-slice theorem [S1, unofficial]. It is distinguished from its sibling [Fourier Projection-Slice Volume Rendering](824-fourier-projection-slice-volume-rendering.md) by using the $\cos + \sin$ kernel instead of the complex exponential $\cos + i\sin$, which avoids complex arithmetic for real-valued signals [S1, unofficial].

## How it works
The method follows the three-stage pipeline of [Domain-transform methods](28-domain-transform-methods.md): Global Transformation, Slice Extraction, and Inverse Transformation [G].

**Inputs and Outputs**
*   **Input:** A 3D scalar volume $f(x, y, z)$ with dimensions $N_x \times N_y \times N_z$ [G].
*   **Output:** A 2D projection image $I(u, v)$ corresponding to a viewing direction [G].

**Data Structures**
*   **Volume:** 3D array of real numbers [G].
*   **Spectrum:** 3D array of real numbers $H(\omega_1, \omega_2, \omega_3)$ [G].
*   **Slice:** 2D array of real numbers $I(u, v)$ [G].

**Numbered Steps**
1.  **Global Transformation:** Compute the 3D Discrete Hartley Transform (DHT) of the volume [G]. The 1D DHT for a sequence $h_k$ of length $N$ is applied separably along each dimension [G].
2.  **Slice Extraction:** Define a 2D plane passing through the origin of the frequency domain and perpendicular to the viewing direction vector $V$ [G]. Resample the 3D spectrum onto this 2D plane to obtain the 2D slice $I(u, v)$ [G]. Interpolation is required as the plane rarely coincides with the integer lattice [G]. The output resolution must satisfy the Nyquist criterion to prevent aliasing [G].
3.  **Inverse Transformation:** Compute the 2D Inverse Discrete Hartley Transform (IDHT) of the slice $I(u, v)$ to generate the final projection image [G].

**Formulas**
*   Forward transform: $H(f) = \int_{-\infty}^{\infty} h(t)(\cos ft + \sin ft)dt$ [S1, unofficial].
*   Inverse transform: $h(t) = \int_{-\infty}^{\infty} H(f)(\cos ft + \sin ft)df$ [S1, unofficial].
*   1D DHT: $H_n = \sum_{k=0}^{N-1} h_k \cdot \text{cas}\left(\frac{2\pi nk}{N}\right)$ [S1, unofficial].
*   1D IDHT: $h_k = \frac{1}{N} \sum_{n=0}^{N-1} H_n \cdot \text{cas}\left(\frac{2\pi nk}{N}\right)$ [S1, unofficial].
*   Kernel abbreviation: $\text{cas}(x) = \sin(x) + \cos(x)$ [S1, unofficial].
*   Even part: $E(f) = \frac{H(f) + H(-f)}{2} = \int_{-\infty}^{\infty} h(t) \cos ft dt$ [S1, unofficial].
*   Odd part: $O(f) = \frac{H(f) - H(-f)}{2} = \int_{-\infty}^{\infty} h(t) \sin ft dt$ [S1, unofficial].
*   Conversion to Fourier: $S(f) = E(f) + iO(f)$ [S1, unofficial].
*   Conversion from Fourier: $H(f) = \text{Re}[S(f)] + \text{Im}[S(f)]$ [S1, unofficial].

**Parameters**
*   **Scaling:** $1/N$ for the inverse transform [S1, unofficial].
*   **Interpolation:** Not given in the sources. [G]
*   **Zero-padding factor:** Not given in the sources. [G]

**Edge Cases and Pitfalls**
*   **Boundary Artifacts:** The transform assumes periodic boundary conditions [G]. Discontinuities at volume boundaries cause high-frequency artifacts, mitigated by zero-padding the volume and applying vignetting correction [G].
*   **Aliasing:** Insufficient output resolution violates the Nyquist criterion, causing aliasing [G].
*   **Fixed Parameters:** Once transformed, parameters like opacity and lighting cannot be changed interactively, as the transformation is linear and global [G].

## Subtopics

### [Discrete Hartley Transform](6335-discrete-hartley-transform.md)
The Discrete Hartley Transform (DHT) is a real-valued linear transform that maps a sequence of $N$ real numbers to $N$ real numbers, serving as an alternative to the complex DFT for real input data. It is distinguished from the continuous Hartley Transform by operating on discrete sequences, and from the DFT by avoiding complex arithmetic, which simplifies implementation for real-valued signals. In volume rendering, it is used for domain-transform methods where real-valued convolution theorems are advantageous.

### Hartley Projection Slice Theorem
A theorem stating that the 1-D Hartley transform of a projection of an object at angle $\theta$ equals a slice through the 2-D Hartley transform of the object [S1, unofficial]. The projection is defined as $p_\theta(\hat{u}_1) = \int_{-\infty}^{\infty} x(\hat{u}_1 \cos \theta - \hat{u}_2 \sin \theta, \hat{u}_1 \sin \theta + \hat{u}_2 \cos \theta) d\hat{u}_2$ [S1, unofficial]. The theorem is expressed as $P_\theta(\hat{U}_1) = H(\hat{U}_1 \cos \theta, \hat{U}_1 \sin \theta)$, where $P_\theta$ is the Hartley transform of the projection and $H$ is the 2-D Hartley transform of the object $x(u_1, u_2)$ [S1, unofficial]. This validates the use of the Hartley transform in tomographic reconstruction and volume rendering pipelines [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the 3D DHT and 2D IDHT, typically $O(N^3 \log N)$ using fast algorithms [G].
*   **Memory Complexity:** $O(N^3)$ for the 3D spectrum [G].
*   **Measured Performance:** Not given in the sources. [G]
*   **Scaling:** Not given in the sources. [G]
*   **Strengths:** Avoids complex number overhead; maintains real-valued data throughout the pipeline; supports the projection-slice theorem [S1, unofficial].
*   **Limitations:** Not given in the sources. [G]
*   **When to use:** Not given in the sources. [G]
*   **When not to use:** Not given in the sources. [G]

## Background
The Hartley transform was introduced by R.V.L. Hartley in 1942 as a real-valued alternative to the Fourier transform [S1, unofficial]. It has been applied in computer-aided tomographic (CAT) scanners and other signal processing applications [S1, unofficial].

## Sources
[S1] Fourier volume rendering, https://shiftleft.com/mirrors/www.hpl.hp.com/techreports/95/HPL-95-73.pdf, fetched 2026-10-31 (unofficial)
