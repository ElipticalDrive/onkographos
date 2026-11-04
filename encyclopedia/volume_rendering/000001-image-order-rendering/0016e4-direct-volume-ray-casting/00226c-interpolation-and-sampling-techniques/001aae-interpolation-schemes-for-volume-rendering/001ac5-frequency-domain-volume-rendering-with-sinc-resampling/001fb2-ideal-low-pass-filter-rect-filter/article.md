# Ideal Low-Pass Filter (Rect Filter)
Also called: Ideal Kernel Reconstruction, Sinc interpolation, Sine function reconstruction, ideal reconstruction filter, rect filter

## Overview
The Ideal Low-Pass Filter (Rect Filter) is a theoretical reconstruction method that suppresses all spectral replicas except the baseband spectrum, allowing exact signal reconstruction if sampling criteria are met [S1, unofficial]. It is distinguished from its siblings by its perfect frequency response in the passband and complete stopband attenuation, corresponding to a sinc function in the spatial domain [S1, unofficial]. Unlike practical approximations, it serves as the theoretical upper bound for interpolation quality, though it is seldom used in practice due to the infinite extent of its spatial kernel [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sampled signal spectrum $F_s(\nu)$ and the Nyquist rate.
*   **Output:** The reconstructed baseband spectrum $F_r(\nu)$, which corresponds to the original continuous signal if properly sampled.

**Data Structures:**
*   The filter is defined by a frequency response function $H(\nu)$. No explicit spatial data structure is used for the ideal case, as the spatial kernel is infinite.

**Formulas:**
The frequency response $H(\nu)$ is defined as:
$$
H(\nu) = \begin{cases} 
1, & \text{if } |\nu| < \text{Nyquist rate}; \\
0, & \text{if } |\nu| = \text{Nyquist rate}; \\
0, & \text{otherwise.}
\end{cases}
$$
The spatial equivalent kernel, the sinc function, is defined as:
$$
\text{sinc}(z) = \frac{\sin(\pi z)}{\pi z}
$$
Note: The source text defines this as $\sin(z) = \frac{\sin(\pi z)}{(\pi z)}$, which is a notation error in the source; the standard sinc function is used here [S1, unofficial].

**Steps:**
1.  Compute the Fourier transform of the sampled signal to obtain $F_s(\nu)$.
2.  Multiply $F_s(\nu)$ by $H(\nu)$ in the frequency domain to isolate the baseband spectrum [S1, unofficial].
3.  Compute the inverse Fourier transform of the result to obtain the reconstructed signal [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ideal_low_pass_filter(sampled_signal_spectrum, nyquist_rate):
    """
    Applies the ideal rect filter in the frequency domain.
    
    Args:
        sampled_signal_spectrum: Array of complex values representing F_s(v).
        nyquist_rate: Float, the maximum frequency of the baseband signal.
        
    Returns:
        Reconstructed baseband spectrum.
    """
    reconstructed_spectrum = []
    for nu, amplitude in enumerate(sampled_signal_spectrum):
        # Map index nu to frequency. Assuming normalized frequency where 
        # Nyquist rate corresponds to a specific index threshold.
        # In practice, nu represents frequency bins.
        freq = nu # Simplified mapping; actual implementation depends on sampling rate
        
        if abs(freq) < nyquist_rate:
            # Passband: Gain is 1
            reconstructed_spectrum.append(amplitude * 1.0)
        else:
            # Stopband: Gain is 0
            reconstructed_spectrum.append(0.0)
            
    return reconstructed_spectrum
```
*Note: The spatial domain implementation involves convolution with $\text{sinc}(z)$. Due to infinite extent, this is not implemented directly [S1, unofficial].*

**Parameters:**
*   **Nyquist Rate:** The cutoff frequency. Not given in the sources as a specific value, but defined as the boundary between passband and stopband [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Infinite Extent:** The spatial kernel (sinc function) has infinite extent, requiring infinite computational work for exact convolution [S1, unofficial].
*   **Truncation Artifacts:** Truncating the sinc function to a finite extent for practical use causes ringing (high-frequency leakage) in the frequency domain, distorting the filter [S1, unofficial].
*   **Ripple Ratio:** Early truncation results in a large ripple ratio (ratio of side lobe amplitude to main lobe amplitude), causing significant artifacts [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Infinite for exact implementation due to the infinite extent of the sinc kernel [S1, unofficial].
*   **Memory Complexity:** Infinite for exact implementation [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Theoretically reproduces the original signal without error if properly sampled; passes all frequencies in the passband without distortion and completely suppresses all frequencies in the stopband [S1, unofficial].
*   **Limitations:** Requires infinite computational work; truncation leads to adverse passband/stopband effects and ringing artifacts [S1, unofficial].
*   **When to use:** Theoretical analysis and benchmarking; seldom used in practice due to the trade-off between computation time and artifact avoidance [S1, unofficial].
*   **When not to use:** Real-time or resource-constrained volume rendering where truncation artifacts are unacceptable [S1, unofficial].

## Background
*   **Origin:** The concept is rooted in signal processing theory, specifically the sampling theorem and ideal reconstruction filters [S1, unofficial].
*   **History:** The rect filter is described in the context of volume rendering interpolation methods, citing Oppenheim [S1, unofficial]. It is discussed as the ideal case against which other interpolation kernels (like linear or cubic) are compared [S1, unofficial].
*   **Usage:** Seldom used in practice in volume rendering due to the computational cost and artifacts associated with truncating the sinc function [S1, unofficial].

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
