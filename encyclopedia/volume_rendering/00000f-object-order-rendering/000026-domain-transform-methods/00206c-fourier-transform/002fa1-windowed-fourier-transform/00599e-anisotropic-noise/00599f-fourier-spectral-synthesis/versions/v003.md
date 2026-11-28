# Fourier Spectral Synthesis
Also called: Fourier Noise Generation, Spectral Synthesis

## Overview
Fourier spectral synthesis generates a noise function with a specific target power spectrum by filtering white noise in the frequency domain [S1, S2]. It sets itself apart from spatial-domain methods (like [Sparse Convolution Noise](16455-sparse-convolution-noise.md)) by enforcing spectral properties directly, making it the foundational technique for [Anisotropic Noise](16450-anisotropic-noise.md) and [Spot Noise](16452-spot-noise.md).

## How it works
**Inputs:**
1.  Target power spectrum $P(k)$, where $k$ is the spatial frequency vector.
2.  Domain size $N$ (for discrete implementation).
3.  Resolution $R$ (number of samples per dimension).

**Outputs:**
1.  Noise field $n(x)$ defined on the spatial domain.

**Data Structures:**
*   **Frequency Grid:** A 3D array of complex numbers of size $N \times N \times N$ (or 2D/1D as needed). Let this be $F$.
*   **Spatial Grid:** A 3D array of real numbers of size $R \times R \times R$. Let this be $n$.

**Steps:**
1.  **Initialize White Noise:** Generate a complex-valued random field $W$ on the frequency grid. For each frequency bin $k$, draw the real and imaginary parts from a normal distribution $\mathcal{N}(0, 1)$ or uniform distribution, ensuring Hermitian symmetry ($W[-k] = \overline{W[k]}$) to guarantee a real-valued output [S1, S2].
2.  **Apply Spectral Filter:** Scale the white noise by the square root of the target power spectrum.
    $$ F[k] = W[k] \cdot \sqrt{P(k)} $$
    Here, $P(k)$ is the desired power spectral density at frequency $k$. For isotropic noise, $P(k)$ depends only on $|k|$. For anisotropic noise (as in [Anisotropic Noise](16450-anisotropic-noise.md)), $P(k)$ is shaped by oriented subbands [S1].
3.  **Inverse Transform:** Compute the Inverse Fast Fourier Transform (IFFT) of $F$.
    $$ \tilde{n}(x) = \text{IFFT}(F) $$
4.  **Extract Real Part:** Since numerical errors may introduce small imaginary components, take the real part.
    $$ n(x) = \text{Re}(\tilde{n}(x)) $$
5.  **Normalization (Optional):** Scale $n(x)$ to have zero mean and unit variance if required by the application [G].

**Parameters:**
*   $P(k)$: The shape of the spectrum (e.g., $1/|k|^2$ for Brownian motion, flat for white noise). Not given in the sources for specific standard values, but generally defined by the user.

**Edge Cases and Pitfalls:**
*   **Hermitian Symmetry:** Failure to enforce $W[-k] = \overline{W[k]}$ results in a complex-valued noise field, which is invalid for scalar noise functions [G].
*   **DC Component:** The zero-frequency component ($k=0$) determines the mean. It should be set to 0 for zero-mean noise.
*   **Aliasing:** If the target spectrum has significant energy at frequencies higher than the Nyquist limit of the spatial resolution, aliasing will occur. Pre-filtering or anti-aliasing techniques are required [G].

## Subtopics

### [Surface Noise via Slice-Projection Theorem](16460-surface-noise-via-slice-projection-theorem.md)
Generates band-pass noise on curved surfaces by integrating a 3D band-pass solid noise field perpendicular to the surface. It avoids the low-frequency drift and aliasing artifacts caused by naively slicing 3D noise, and prevents the distortions of planar parametrization. Choose this when anisotropic textures must flow naturally around complex topologies.

## Operating characteristics
*   **Time Complexity:** $O(N^3 \log N)$ for 3D generation using FFT, where $N$ is the grid resolution.
*   **Memory Complexity:** $O(N^3)$ for storing the frequency and spatial grids.
*   **Strengths:** Exact control over the power spectrum; mathematically rigorous; generates smooth, continuous noise.
*   **Limitations:** Global dependency (changing one sample requires recomputing the entire field via FFT); high memory usage for high resolutions; not easily localizable.
*   **When to use:** When precise spectral properties are required (e.g., for [Anisotropic Noise](16450-anisotropic-noise.md) or [Spot Noise](16452-spot-noise.md)); for generating reference solutions.
*   **When not to use:** For real-time applications requiring local updates or sparse sampling; when memory is constrained.

## Background
Fourier spectral synthesis was introduced in computer graphics by Anjyo [Anj88], Saupe [Sau88], and Voss [Vos88] to generate random fractals for simulating natural phenomena like terrain [S1, S2]. It is based on the principle that filtering white noise in the frequency domain yields a process with a specified power spectrum [S1, S2]. The method is also related to the mathematical texturing function of Gardner [Gar84] [S1]. It is widely used in explicit noise methods, such as van Wijk's spot noise [vW91] and Goldberg et al.'s anisotropic noise [GZD08] [S1, S2].

## Sources
[S1] A Survey of Procedural Noise Functions, https://www.cs.umd.edu/~zwicker/publications/SurveyProceduralNoise-CGF10.pdf, fetched 2026-10-27
[S2] State of the Art in Procedural Noise Functions, https://www-sop.inria.fr/reves/Basilic/2010/LLCDDELPZ10/LLCDDELPZ10STARPNF.pdf, fetched 2026-10-27
