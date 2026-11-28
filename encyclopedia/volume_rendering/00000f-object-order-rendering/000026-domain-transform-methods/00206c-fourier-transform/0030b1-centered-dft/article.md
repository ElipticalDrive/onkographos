# Centered DFT
Also called: Discrete Fourier Transform (DFT) with zero-padding and frequency shifting, FFT preprocessing

## Overview
Centered DFT is a preprocessing technique that shifts the zero-frequency component of a Discrete Fourier Transform to the center of the frequency domain array. It solves the problem of standard DFT output having the zero frequency at the top-left corner, which creates a visual and analytical discontinuity at the boundaries [S1, unofficial]. This method builds on [Fourier Transform](2288-fourier-transform.md) by providing the specific spatial modulation required to align the spectrum with standard Cartesian conventions where low frequencies are central.

## How it works
**Inputs and outputs:**
- Input: A spatial domain signal $h$ (1D, 2D, or 3D).
- Output: The DFT of the modulated signal, where the zero-frequency component is at the geometric center.

**Data structures:**
- The input signal is stored in a contiguous array or multi-dimensional array.
- For 1D: array of size $N$.
- For 2D: array of size $M \times N$.
- For 3D: array of size $L \times M \times N$.

**Steps:**
1. **Modulation:** Multiply each element of the spatial signal by a checkerboard pattern of signs. This implements the frequency shift property [S1, unofficial].
   - For 1D signal $h_k$ of length $N$:
     $$ h'_k = h_k \cdot (-1)^k $$
   - For 2D signal $h_{x,y}$ of size $M \times N$:
     $$ h'_{x,y} = h_{x,y} \cdot (-1)^{x+y} $$
   - For 3D signal $h_{x,y,z}$ of size $L \times M \times N$:
     $$ h'_{x,y,z} = h_{x,y,z} \cdot (-1)^{x+y+z} $$
2. **Transform:** Compute the standard DFT (or FFT) of the modulated signal $h'$ [S1, unofficial].
3. **Result:** The resulting frequency spectrum $H$ has the zero-frequency component at the center indices ($N/2, M/2$ for 2D, etc.).

**Formulas:**
The standard DFT is defined as:
$$ H_u = \sum_{k=0}^{N-1} h_k e^{-2\pi i k u / N} $$
Multiplying by $(-1)^k = e^{-i \pi k} = e^{-2\pi i k (N/2) / N}$ shifts the frequency origin by $N/2$ [S1, unofficial].

**Parameters:**
- $N, M, L$: Dimensions of the signal. Must be integers.
- $k, x, y, z$: Indices of the signal elements.

**Edge cases and pitfalls:**
- **Precision:** The modulation factor $(-1)^k$ is exactly $\pm 1$, so no floating-point error is introduced during modulation.
- **Boundaries:** The method assumes the signal is periodic. The shift aligns the periodicity such that the transition from high to low frequency occurs at the array boundaries in the centered view, rather than at the original corners [S1, unofficial].
- **Degenerate input:** If $N=1$, the shift is trivial.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def centered_dft_1d(signal):
    """
    Computes the centered DFT of a 1D signal.
    signal: list or array of length N
    returns: complex array of length N
    """
    N = len(signal)
    modulated = [signal[k] * ((-1) ** k) for k in range(N)]
    # Standard DFT implementation (or FFT)
    result = [0] * N
    for u in range(N):
        sum_val = 0
        for k in range(N):
            exponent = -2 * 3.141592653589793 * k * u / N
            sum_val += modulated[k] * complex(cos(exponent), sin(exponent))
        result[u] = sum_val
    return result

def centered_dft_2d(signal):
    """
    Computes the centered DFT of a 2D signal.
    signal: 2D array of size M x N
    returns: 2D complex array of size M x N
    """
    M = len(signal)
    N = len(signal[0])
    modulated = [[0] * N for _ in range(M)]
    for x in range(M):
        for y in range(N):
            modulated[x][y] = signal[x][y] * ((-1) ** (x + y))
    
    # Standard 2D DFT implementation (or FFT)
    result = [[0] * N for _ in range(M)]
    for u in range(M):
        for v in range(N):
            sum_val = 0
            for x in range(M):
                for y in range(N):
                    exponent = -2 * 3.141592653589793 * (x * u / M + y * v / N)
                    sum_val += modulated[x][y] * complex(cos(exponent), sin(exponent))
            result[u][v] = sum_val
    return result
```

## Subtopics

### [2D FFT Shift](20584-2d-fft-shift.md)
The 2D FFT Shift rearranges the quadrants of a 2D frequency-domain array to center the zero-frequency component, swapping the first and fourth quadrants (Q1↔Q4) and the second and third quadrants (Q2↔Q3). It solves the problem of the standard Discrete Fourier Transform (DFT) output having the zero frequency at the top-left corner, which creates a visual and analytical discontinuity at the boundaries. This method builds on [Centered DFT](6328-centered-dft.md) by providing the specific permutation logic required to achieve that centered layout in 2D.

### [3D FFT Shift](20810-3d-fft-shift.md)
The 3D FFT Shift method repositions the zero-frequency component of a 3D Fourier volume or spatial spectrum to the geometric center of the data array by swapping spatial octants. This operation is essential for frequency-domain visualization and processing, as it aligns the spectrum with standard Cartesian conventions where low frequencies are central and high frequencies are at the periphery. It is implemented as a permutation of existing data elements, requiring no arithmetic operations, and serves as a post-processing step for 3D Fast Fourier Transforms (FFTs).

## Operating characteristics
- **Time complexity:** $O(N \log N)$ if using FFT, or $O(N^2)$ for naive DFT in 1D. For 2D, $O(MN \log(MN))$ with FFT. The modulation step is $O(N)$ or $O(MN)$.
- **Memory complexity:** $O(N)$ or $O(MN)$ for the input and output arrays. The modulation can be done in-place.
- **Strengths:** Provides a visually intuitive frequency spectrum where low frequencies are centered. Essential for frequency-domain filtering and visualization [S1, unofficial].
- **Limitations:** Requires an additional multiplication step per element. The resulting spectrum is still periodic and may exhibit artifacts if the signal is not properly windowed or padded.
- **When to use:** When visualizing frequency content or applying frequency-domain filters where the center of the array corresponds to the DC component.
- **When not to use:** When computational resources are extremely limited and the standard DFT layout is acceptable for the application.

## Background
The Centered DFT is a standard technique in signal processing and image analysis. It is derived from the frequency shift property of the Fourier Transform [S1, unofficial]. The method is widely used in scientific computing, particularly in volume rendering and image processing, to facilitate the analysis of spatial frequencies [S1, unofficial].

## Sources
[S1] Fourier volume rendering, https://shiftleft.com/mirrors/www.hpl.hp.com/techreports/95/HPL-95-73.pdf, fetched 2026-10-31 (unofficial)
