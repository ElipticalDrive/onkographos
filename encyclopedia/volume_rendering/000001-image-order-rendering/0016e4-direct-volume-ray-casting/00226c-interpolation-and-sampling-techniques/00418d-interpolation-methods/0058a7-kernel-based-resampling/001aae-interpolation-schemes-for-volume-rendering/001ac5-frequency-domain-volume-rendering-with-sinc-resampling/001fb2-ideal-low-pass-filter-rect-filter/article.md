# Ideal Low-Pass Filter (Rect Filter)

Also called: Ideal Kernel Reconstruction, Sinc Filter, Sinc interpolation, Sine function reconstruction, ideal reconstruction filter, rect filter

## Overview
The Ideal Low-Pass Filter (Rect Filter) is the theoretical optimum for signal reconstruction, providing the highest possible quality interpolation by perfectly isolating the baseband spectrum and suppressing all spectral replicas [S2, unofficial]. It is distinguished from its siblings, such as [POCS Resampling Filter Design](6373-pocs-resampling-filter-design.md) which optimizes filter coefficients for specific constraints, and [Frequency Domain Depth Cueing via Impulse Shifting](862-frequency-domain-depth-cueing-via-impulse-shifting.md) which manipulates frequency components for depth effects, by relying on the mathematically perfect rectangular frequency response [S2, unofficial]. While it theoretically reproduces the original signal without error if sampling criteria are met, it is rarely used in practice due to its infinite spatial extent and the severe artifacts introduced by necessary truncation [S1, unofficial][S2, unofficial].

## How it works
The method operates by convolving the sampled volume data with a reconstruction kernel that corresponds to the inverse Fourier transform of a rectangular function in the frequency domain [S2, unofficial].

**Inputs and Outputs**
*   **Input:** A discrete sampled signal $f_s[n]$ (or 3D volume data) sampled at intervals $\Delta x$ [S2, unofficial].
*   **Output:** A continuous reconstructed signal $f(x)$ or resampled discrete values at arbitrary positions [S2, unofficial].

**Data Structures**
*   **Frequency Response $H(\nu)$:** Defined by the Nyquist rate $\nu_N = \frac{1}{2\Delta x}$ [S2, unofficial].
*   **Spatial Kernel $h(x)$:** The sinc function, defined as:
    $$ \text{sinc}(x) = \frac{\sin(\pi x)}{\pi x} $$
    The source [S2, unofficial] defines $\sin(z) = \frac{\sin(\pi z)}{(\pi z)}$, which is the normalized sinc function [S2, unofficial]. We use the normalized form $\frac{\sin(\pi x)}{\pi x}$ consistent with the frequency domain definition where the cutoff is at $\nu_N$ [S2, unofficial].

**Steps**
1.  **Define Frequency Response:** Construct the ideal low-pass filter $H(\nu)$ in the frequency domain. For a signal sampled with Nyquist rate $\nu_N$:
    $$ H(\nu) = \begin{cases} 1, & \text{if } |\nu| < \nu_N \\ 0, & \text{if } |\nu| = \nu_N \\ 0, & \text{otherwise} \end{cases} $$
    This filter passes all frequencies in the passband (below $\nu_N$) without distortion and completely suppresses all frequencies in the stopband (above $\nu_N$) [S2, unofficial].
2.  **Transform to Spatial Domain:** The reconstruction kernel $h(x)$ is the inverse Fourier transform of $H(\nu)$, resulting in the sinc function:
    $$ h(x) = \text{sinc}\left(\frac{x}{\Delta x}\right) = \frac{\sin(\pi x / \Delta x)}{\pi x / \Delta x} $$
    Multiplication in the frequency domain by the rect function is equivalent to convolution in the spatial domain with this sinc function [S2, unofficial].
3.  **Convolution (Theoretical):** The reconstructed value at position $x$ is calculated by convolving the sampled signal with the kernel:
    $$ f(x) = \sum_{n=-\infty}^{\infty} f_s[n] \cdot h(x - n\Delta x) $$
    If the signal was properly sampled (satisfying the Nyquist-Shannon criteria), this process reproduces the original continuous signal without error [S2, unofficial].
4.  **Truncation and Windowing (Practical Implementation):** Since the sinc function has infinite extent, it must be truncated to a finite width for practical computation [S2, unofficial]. To mitigate the ringing artifacts caused by the sharp cutoff of truncation, the kernel is multiplied by a window function $w(x)$:
    $$ h_{practical}(x) = h(x) \cdot w(x) $$
    The window function smoothly goes to zero at the truncation point [S2, unofficial]. The window can be adaptive to local detail [S1, unofficial].

**Parameters**
*   **Kernel Width:** The number of samples over which the sinc function is evaluated [S2, unofficial]. [S2, unofficial] notes that the width directly affects the computation required for convolution.
*   **Window Function:** Determines the tapering at the edges of the truncated kernel [S2, unofficial]. The sources mention windowing but do not specify common window types or parameters [S1, unofficial][S2, unofficial].

**Edge Cases and Pitfalls**
*   **Infinite Extent:** The sinc function has infinite spatial extent, requiring infinite computational work for exact implementation [S2, unofficial].
*   **Ringing Artifacts:** Truncating the sinc function in the spatial domain produces high-frequency leakage (ringing) in the frequency domain, distorting the filter's passband and stopband performance [S2, unofficial].
*   **Large Ripple Ratio:** Early truncation results in a large ripple ratio, defined as the ratio of side lobe amplitude to main lobe amplitude, causing significant visual artifacts [S2, unofficial].
*   **Derivative Conditions:** For visualization algorithms based on derivatives (e.g., volume gradients), the reconstruction filter must satisfy six specific conditions to avoid visible artifacts [S3, unofficial]:
    1.  The function should be continuous [S3, unofficial].
    2.  The derivative of the function should be continuous [S3, unofficial].
    3.  The function should be symmetrical [S3, unofficial].
    4.  The value should be one at $x=0$ and zero at $x=nx$ (locations of other samples) for interpolating filters [S3, unofficial].
    5.  When data is constant, the reconstructed value should be the same constant [S3, unofficial].
    6.  When sample values are a linear function of place, reconstructed values should also be a linear function of place [S3, unofficial].
    Using a filter that does not satisfy these conditions leads to visible artifacts, especially in 3D visualization [S3, unofficial]. These conditions apply to 1D filters applied repeatedly in multi-dimensional reconstruction [S3, unofficial].

## Subtopics

### Reconstruction Filter Conditions
A set of six necessary conditions for reconstruction filters to be suitable for visualization algorithms based on derivatives [S3, unofficial]. These conditions apply to 1D filters applied repeatedly in multi-dimensional reconstruction [S3, unofficial]. Violating these conditions leads to visible artifacts in 3D visualization [S3, unofficial]. The conditions are: 1) continuity of the function, 2) continuity of the derivative, 3) symmetry, 4) interpolating property (1 at center, 0 at other sample locations), 5) preservation of constant data, and 6) preservation of linear data [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot W)$, where $N$ is the number of samples and $W$ is the kernel width [G]. Because the ideal kernel has infinite extent, exact implementation is $O(\infty)$ [S2, unofficial]. Practical implementations require truncation, making complexity dependent on the chosen kernel width [S2, unofficial].
*   **Measured Performance:** The sources do not provide specific benchmark numbers [S1, unofficial][S2, unofficial]. However, it is noted that higher filter quality comes at the price of wider spatial extent and larger computational effort [S1, unofficial].
*   **Strengths:** Theoretically provides the best possible interpolation quality with no aliasing if the signal is band-limited and properly sampled [S2, unofficial]. It is the "best filter possible in the numerical sense" [S1, unofficial].
*   **Limitations:** Infinite spatial extent makes it impractical [S2, unofficial]. Truncation causes significant ringing artifacts due to the large ripple ratio [S2, unofficial]. It is seldom used in practice due to the trade-off between computation time and artifact avoidance [S2, unofficial].
*   **When to Use:** Only when theoretical maximum quality is required and computational resources are unlimited, or as a baseline for comparing other filters [S2, unofficial]. In practice, first-order (linear) or cubic filters are preferred for better cost-quality trade-offs [S1, unofficial].

## Background
The Ideal Low-Pass Filter is rooted in classical signal processing theory, specifically the Nyquist-Shannon sampling theorem [S2, unofficial]. The concept of the rectangular filter in the frequency domain and its sinc function counterpart in the spatial domain is detailed in standard texts such as Oppenheim [S2, unofficial]. In the context of volume rendering, it is discussed as the theoretical upper bound for interpolation quality, with practical limitations highlighted in works by Mitchell [S2, unofficial] and Müller et al. [S1, unofficial]. The specific conditions for reconstruction filters suitable for visualization algorithms based on derivatives were formalized in research [S3, unofficial].

## Sources
[S1] Overview of Volume Rendering, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
[S3] Iso-Surface Volume Rendering : speed and accuracy for medical applications, https://doi.org/10.3990/1.9789036513982, fetched 2026-10-26 (unofficial)
