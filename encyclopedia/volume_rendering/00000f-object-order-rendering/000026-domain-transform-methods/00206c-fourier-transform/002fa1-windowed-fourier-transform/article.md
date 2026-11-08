# Windowed Fourier Transform

Also called: Gabor Transform

## Overview
The Windowed Fourier Transform (WFT), also known as the Gabor Transform, addresses the lack of spatial localization inherent in the standard Fourier Transform. While the standard Fourier Transform provides a global frequency measure by integrating over the entire spatial domain, making it difficult to locate irregularities [S1, unofficial], the WFT uses a translated window function to cover the signal, allowing for the localization of these irregularities [S1, unofficial]. It builds on [Fourier Transform](2288-fourier-transform.md) by introducing spatial constraints to the frequency analysis.

## How it works
**Inputs and Outputs:**
*   **Input:** A signal $f(x)$ belonging to $L^2(\mathbb{R})$ [S1, unofficial].
*   **Output:** A 2D representation of the signal in the time-frequency (or space-frequency) domain, denoted as $G_f(u, \omega)$, where $u$ is the translation parameter and $\omega$ is the frequency parameter [S1, unofficial].

**Data Structures:**
The output is typically stored as a 2D array or matrix where one axis represents the translation (position) and the other represents the frequency. The values are complex numbers representing the amplitude and phase of the signal component at that specific position and frequency.

**Method:**
1.  **Define the Window Function:** Select a window function $g(x)$ that is localized in space. The window is typically symmetric and decays to zero at its boundaries.
2.  **Translate and Modulate:** For a specific position $u$ and frequency $\omega$, the window function is translated by $u$ and modulated by a complex exponential $e^{-i\omega x}$ [S1, unofficial].
3.  **Compute the Integral:** The transform value at $(u, \omega)$ is computed by integrating the product of the signal $f(x)$, the translated window $g(x-u)$, and the complex exponential over the entire domain [S1, unofficial].

**Formula:**
The Windowed Fourier Transform of a function $f(x)$ is defined by [S1, unofficial]:

$$ G_f(u, \omega) = \int_{-\infty}^{\infty} f(x) g(x - u) e^{-i\omega x} \, dx $$

Where:
*   $G_f(u, \omega)$ is the WFT coefficient at translation $u$ and frequency $\omega$.
*   $f(x)$ is the input signal.
*   $g(x)$ is the window function.
*   $u$ is the translation parameter (center of the window).
*   $\omega$ is the frequency parameter.
*   $i$ is the imaginary unit.

**Parameters:**
*   **Window Function $g(x)$:** Common choices include Gaussian, Hanning, or Hamming windows. The choice affects the trade-off between spatial and frequency resolution.
*   **Step Sizes:** In discrete implementations, step sizes for $u$ and $\omega$ determine the resolution of the output grid.

**Edge Cases and Pitfalls:**
*   **Boundary Effects:** If the signal is finite, the window may extend beyond the signal boundaries, requiring zero-padding or window adjustment.
*   **Resolution Trade-off:** The width of the window function determines the trade-off between spatial and frequency resolution. A narrower window provides better spatial localization but poorer frequency resolution, and vice versa [S1, unofficial].

## Subtopics

### Aperture Apodization
Aperture apodization is a technique used to suppress side lobe leakage and aperture diffraction artifacts that arise from sharp intensity edges of a rectangular aperture during numerical reconstruction, such as in holography [S2]. It works by smoothly bringing the aperture edges to zero, which prevents the side lobes that would otherwise be produced upon propagation [S2]. A common implementation uses a 1D Hanning window function defined as [S2]:

$$ h_n(x) = \cos^2\left(\frac{\pi x}{2\alpha}\right) $$

where $\alpha$ is the full width at half maximum (FWHM) [S2]. A potential limitation is a decrease in resolution due to the limited signal area after apodization [S2].

## Operating characteristics
*   **Time Complexity:** The direct computation of the WFT for all positions and frequencies is $O(N^2)$ for a signal of length $N$, similar to the standard Fourier Transform, but with an additional factor due to the windowing. Fast algorithms using the Fast Fourier Transform (FFT) can reduce this to $O(N \log N)$ if the window is fixed and the translation is uniform.
*   **Memory Complexity:** $O(N \cdot M)$ where $N$ is the signal length and $M$ is the number of frequency bins, as the output is a 2D representation.
*   **Strengths:** Provides simultaneous time (or space) and frequency information, allowing for the localization of signal features [S1, unofficial].
*   **Limitations:** The choice of window function and its width involves a trade-off between spatial and frequency resolution (Heisenberg-Gabor limit). It is less efficient than the standard Fourier Transform for global frequency analysis.
*   **When to Use:** When the signal is non-stationary and it is necessary to analyze how the frequency content changes over time or space [S1, unofficial].
*   **When Not to Use:** When only global frequency information is needed, or when computational resources are severely limited and the signal is stationary.

## Background
The Windowed Fourier Transform was introduced by Dennis Gabor in the context of signal analysis [S1, unofficial]. Gabor proposed the use of a spatial window in the Fourier integral to localize the information provided by the Fourier transform, addressing the limitation that the standard Fourier transform covers the whole spatial domain and is not spatially localized [S1, unofficial]. This method has been applied in various fields, including volume rendering of multiresolution decomposed data [S1, unofficial] and computer-generated holography [S2].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S2] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
