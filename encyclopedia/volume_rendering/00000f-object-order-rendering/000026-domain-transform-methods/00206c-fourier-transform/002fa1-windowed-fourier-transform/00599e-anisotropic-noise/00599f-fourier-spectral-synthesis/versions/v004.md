# Fourier Spectral Synthesis
Also called: Fourier Noise Generation, Fourier Synthesis, Spectral Synthesis

## Overview
Fourier spectral synthesis generates a noise function with a specific target power spectrum by filtering white noise in the frequency domain [S2, S3]. It is used in explicit noise methods such as [Spot Noise](16452-spot-noise.md) and [Anisotropic Noise](16450-anisotropic-noise.md) [S2, S3].

## How it works
**Inputs:** Target power spectrum $P(k)$, domain size $N$, resolution $R$ [G].
**Outputs:** Noise field $n(x)$ [G].
**Data Structures:** Frequency grid $F$ of complex numbers, spatial grid $n$ of real numbers [G].
**Steps:**
1. **Initialize White Noise:** Generate a complex-valued random field $W$ on the frequency grid with Hermitian symmetry to guarantee a real-valued output [G].
2. **Apply Spectral Filter:** Scale the white noise by the square root of the target power spectrum: $F[k] = W[k] \cdot \sqrt{P(k)}$ [G].
3. **Inverse Transform:** Compute the inverse discrete Fourier transform (IDFT) of $F$ to obtain $\tilde{n}(x)$ [G].
4. **Extract Real Part:** Take the real part of $\tilde{n}(x)$ to discard numerical imaginary artifacts [G].
5. **Normalization:** Scale $n(x)$ to zero mean and unit variance if required [G].
**Parameters:** $P(k)$ defines the spectral shape. Not given in the sources for specific standard values.
**Edge Cases and Pitfalls:** Hermitian symmetry must be enforced to prevent complex outputs. The DC component ($k=0$) sets the mean. Aliasing occurs if energy exceeds the Nyquist limit [G].

## Subtopics

### [Surface Noise via Slice-Projection Theorem](16460-surface-noise-via-slice-projection-theorem.md)
Generates band-pass noise on curved surfaces by integrating a 3D band-pass solid noise field perpendicular to the surface. It avoids the low-frequency drift and aliasing artifacts caused by naively slicing 3D noise, and prevents the distortions of planar parametrization. Choose this when anisotropic textures must flow naturally around complex topologies.

## Operating characteristics
*   **Time Complexity:** $O(N^3 \log N)$ for 3D generation using FFT, where $N$ is the grid resolution [G].
*   **Memory Complexity:** $O(N^3)$ for storing the frequency and spatial grids [G].
*   **Strengths:** Exact control over the power spectrum; mathematically rigorous; generates smooth, continuous noise [G].
*   **Limitations:** Global dependency (changing one sample requires recomputing the entire field via FFT); high memory usage for high resolutions; not easily localizable [G].
*   **When to use:** When precise spectral properties are required; for generating reference solutions [S2, S3].
*   **When not to use:** For real-time applications requiring local updates or sparse sampling; when memory is constrained [G].

## Background
Fourier spectral synthesis was introduced in computer graphics by Anjyo, Saupe, and Voss in 1988 to generate random fractals for simulating natural phenomena like terrain [S2, S3]. It is based on the principle that filtering white noise in the frequency domain yields a process with a specified power spectrum [S2, S3]. The method is also related to the mathematical texturing function of Gardner [S2, S3]. It is widely used in explicit noise methods, such as van Wijk's spot noise and Goldberg et al.'s anisotropic noise [S2, S3].

## Sources
[S1] Terrain synthesis using noise, https://trepo.tuni.fi/handle/10024/101043, fetched 2026-10-26 (unofficial)
[S2] PDF A Survey of Procedural Noise Functions, https://www.cs.umd.edu/~zwicker/publications/SurveyProceduralNoise-CGF10.pdf, fetched 2026-10-27
[S3] PDF State of the Art in Procedural Noise Functions, https://www-sop.inria.fr/reves/Basilic/2010/LLCDDELPZ10/LLCDDELPZ10STARPNF.pdf, fetched 2026-10-27
