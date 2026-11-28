# Dual-Tree Complex Wavelet Transform

Also called: DTCWT

## Overview
The Dual-Tree Complex Wavelet Transform (DTCWT) is a wavelet transform that employs two separate real Discrete Wavelet Transforms (DWTs) to compute the real and imaginary parts of the complex coefficients [S1]. It is distinguished from its siblings, such as [Laplacian Pyramid Volume Rendering](2261-laplacian-pyramid-volume-rendering.md) and [3D Discrete Wavelet Transform](6083-3d-discrete-wavelet-transform.md), by constructing complex wavelets from pairs of real wavelets to generate six oriented real subbands in 2D [S1].

## How it works
The method decomposes input data into complex coefficients using a dual-tree structure [S1]. The core mechanism involves constructing two real DWTs with distinct filter banks, where the second tree's wavelets are approximately the Hilbert transform of the first tree's wavelets [S1].

### Inputs and Outputs
- **Input:** A discrete signal or volume $V$. For 2D/3D separable implementation, this is processed axis by axis [S1].
- **Output:** A set of complex wavelet coefficients $C$. In 2D, this typically includes one approximation subband and six oriented detail subbands (three real, three imaginary) per decomposition level [S1].

### Data Structures
The transform relies on two parallel filter banks [S1].
1.  **Upper Tree Filters:** Low-pass filter $h_0(n)$ and high-pass filter $h_1(n)$ [S1].
2.  **Lower Tree Filters:** Low-pass filter $g_0(n)$ and high-pass filter $g_1(n)$ [S1].
3.  **Wavelet Definitions:**
    -   Real wavelet from upper tree: $\psi_h(t)$ derived from $\{h_0, h_1\}$ [S1].
    -   Real wavelet from lower tree: $\psi_g(t)$ derived from $\{g_0, g_1\}$ [S1].
    -   Complex 1D wavelet: $\psi(t) = \psi_h(t) + j\psi_g(t)$, where $\psi_g(t)$ is approximately the Hilbert transform of $\psi_h(t)$ [S1].

### Numbered Steps
1.  **Construct Filter Banks:** Define two distinct real DWT filter banks (upper and lower) [S1].
2.  **Compute 1D Complex Wavelets:** For each axis, compute the complex wavelet $\psi(x) = \psi_h(x) + j\psi_g(x)$ and the complex scaling function $\phi(x) = \phi_h(x) + j\phi_g(x)$ [S1].
3.  **Form 2D Complex Wavelets:** Create separable 2D complex wavelets by tensor product: $\Psi(x,y) = \psi(x)\psi(y)$ [S1].
4.  **Expand Complex Wavelets:** Expand the product to separate real and imaginary parts [S1]. The expansion is given by:
    $$ \Psi(x, y) = [\psi_h(x) + j\psi_g(x)][\psi_h(y) + j\psi_g(y)] = \psi_h(x)\psi_h(y) - \psi_g(x)\psi_g(y) + j[\psi_g(x)\psi_h(y) + \psi_h(x)\psi_g(y)] $$
    The source notes that the spectrum of $\psi_h(x)\psi_h(y) - \psi_g(x)\psi_g(y)$ corresponds to the real part of $\psi(x,y)$ and is supported in two quadrants of the 2D frequency plane [S1]. Another expansion in the text for a different combination yields a plus sign [S1].
5.  **Define Separable Bases:** Define two separable 2D wavelet bases $\Psi_{1,i}(x,y)$ and $\Psi_{2,i}(x,y)$ for $i=1,2,3$ using combinations of $\phi_h, \phi_g, \psi_h, \psi_g$ [S1]:
    -   $i=1$: $\Psi_{1,1} = \phi_h(x)\psi_h(y)$, $\Psi_{2,1} = \phi_g(x)\psi_g(y)$
    -   $i=2$: $\Psi_{1,2} = \psi_h(x)\phi_h(y)$, $\Psi_{2,2} = \psi_g(x)\phi_g(y)$
    -   $i=3$: $\Psi_{1,3} = \psi_h(x)\psi_h(y)$, $\Psi_{2,3} = \psi_g(x)\psi_g(y)$ [S1].
    -   For the imaginary part bases $\Psi_{3,i}, \Psi_{4,i}$:
    -   $i=1$: $\Psi_{3,1} = \phi_g(x)\psi_h(y)$, $\Psi_{4,1} = \phi_h(x)\psi_g(y)$
    -   $i=2$: $\Psi_{3,2} = \psi_g(x)\phi_h(y)$, $\Psi_{4,2} = \psi_h(x)\phi_g(y)$
    -   $i=3$: $\Psi_{3,3} = \psi_g(x)\psi_h(y)$, $\Psi_{4,3} = \psi_h(x)\psi_g(y)$ [S1].
6.  **Compute Oriented Real Wavelets:** Construct six oriented real 2D wavelets from the real and imaginary parts using sum/difference operations with normalization factor $\sqrt{1/2}$ [S1]:
    -   From Real Part ($\Psi_{1,i}, \Psi_{2,i}$):
        $$ \psi_i(x, y) = \sqrt{\frac{1}{2}} (\Psi_{1,i}(x, y) - \Psi_{2,i}(x, y)) $$
        $$ \psi_{i+3}(x, y) = \sqrt{\frac{1}{2}} (\Psi_{1,i}(x, y) + \Psi_{2,i}(x, y)) $$
        for $i=1, 2, 3$ [S1].
    -   From Imaginary Part ($\Psi_{3,i}, \Psi_{4,i}$):
        $$ \psi_i(x, y) = \sqrt{\frac{1}{2}} (\Psi_{3,i}(x, y) + \Psi_{4,i}(x, y)) $$
        $$ \psi_{i+3}(x, y) = \sqrt{\frac{1}{2}} (\Psi_{3,i}(x, y) - \Psi_{4,i}(x, y)) $$
        for $i=1, 2, 3$ [S1].
7.  **Apply Transform:** Convolve the input volume with these filters and downsample by 2 at each level to produce the coefficient tree [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def dtcwt_2d_forward(volume, h0, h1, g0, g1):
    """
    Performs a 2D Dual-Tree Complex Wavelet Transform.
    
    Args:
        volume: 2D array of input data.
        h0, h1: Upper tree low-pass and high-pass filters.
        g0, g1: Lower tree low-pass and high-pass filters.
        
    Returns:
        coeffs: Dictionary containing approximation and 6 detail subbands.
    """
    def apply_1d(data, kernel, axis):
        pad = (len(kernel) - 1) // 2
        if axis == 0:
            padded = np.pad(data, ((pad, pad), (0, 0)), mode='reflect')
            return np.array([np.convolve(padded[i, :], kernel, mode='valid') for i in range(data.shape[0])])[::2, :]
        else:
            padded = np.pad(data, ((0, 0), (pad, pad)), mode='reflect')
            return np.array([np.convolve(padded[:, j], kernel, mode='valid') for j in range(data.shape[1])])[:, ::2]

    # Upper tree 1D transforms along axis 0
    U0 = apply_1d(volume, h0, 0)
    U1 = apply_1d(volume, h1, 0)
    # Lower tree 1D transforms along axis 0
    L0 = apply_1d(volume, g0, 0)
    L1 = apply_1d(volume, g1, 0)

    # Upper tree 2D transforms along axis 1
    U00 = apply_1d(U0, h0, 1)
    U01 = apply_1d(U0, h1, 1)
    U10 = apply_1d(U1, h0, 1)
    U11 = apply_1d(U1, h1, 1)

    # Lower tree 2D transforms along axis 1
    L00 = apply_1d(L0, g0, 1)
    L01 = apply_1d(L0, g1, 1)
    L10 = apply_1d(L1, g0, 1)
    L11 = apply_1d(L1, g1, 1)

    # Form 4 complex subbands
    LL = U00 + 1j * L00
    LH = U01 + 1j * L01
    HL = U10 + 1j * L10
    HH = U11 + 1j * L11

    # Combine into 6 oriented complex subbands [S1]
    d1 = np.sqrt(0.5) * (LL + HH)
    d2 = np.sqrt(0.5) * (LH + 1j * HL)
    d3 = np.sqrt(0.5) * (LH - 1j * HL)
    d4 = np.sqrt(0.5) * (LL - HH)
    d5 = np.sqrt(0.5) * (LH - 1j * HL).conj()
    d6 = np.sqrt(0.5) * (LH + 1j * HL).conj()

    return {
        'approx': LL.real,
        'detail_1': d1, 'detail_2': d2, 'detail_3': d3,
        'detail_4': d4, 'detail_5': d5, 'detail_6': d6
    }
```

### Parameters
-   **Wavelet Level:** Set to 1 in experiments to balance performance, training time, and model size [S1]. Increasing levels does not significantly improve performance but increases training time and model size [S1].
-   **Basis Numbers ($R_1, R_2, R_3$):** Used for direction-aware representation in planes (XY-ZT, XZ-YT, YZ-XT) [S1]. Appearance: $R_1=48, R_2=48, R_3=48$ [S1]. Opacity: $R_1=24, R_2=24, R_3=24$ [S1].
-   **Coordinate Boundaries:** Scene modeled using Normalized Device Coordinates (NDC) with min boundaries $[-2.5, -2.0, -1.0]$ and max boundaries $[2.5, 2.0, 1.0]$ [S1].
-   **Mask Loss Weight ($\lambda_m$):** Controls training data sparsity [S1]. Tested values include $-10, -11, 0$ [S1].

### Edge Cases and Pitfalls
-   **Filter Design:** The lower tree filters must be designed such that $\psi_g(t)$ is approximately the Hilbert transform of $\psi_h(t)$ [S1].
-   **Boundary Handling:** Standard DWT boundary handling (extension) can introduce artifacts [G].
-   **Precision:** The use of complex arithmetic doubles the computational cost compared to real DWT [G].

## Operating characteristics
-   **Time Complexity:** Not given in the sources [S1].
-   **Memory Complexity:** Not given in the sources [S1].
-   **Performance:** In the context of dynamic scene representation (DaReNeRF), DTCWT-based methods achieved PSNR of 30.294 (Flame Salmon) and 32.630 (Cook Spinach) on the Plenoptic Video Dataset, and 25.71 (Hell Warrior) and 34.08 (Bouncing Balls) on the D-NeRF Dataset [S1].
-   **Scalability:** Increasing the wavelet level beyond 1 does not significantly improve performance but substantially increases training time and model size [S1].
-   **Strengths:** Enhanced directional selectivity (6 orientations in 2D vs 3 in standard DWT) and approximate shift invariance [S1].
-   **Limitations:** Higher computational cost and memory usage compared to real DWT. Filter design is more complex [S1].

## Background
The Dual-Tree Complex Wavelet Transform (DTCWT) was introduced to address the limitations of the standard Discrete Wavelet Transform (DWT), specifically its lack of shift invariance and limited directional selectivity in 2D and 3D [S1]. The method employs two real DWTs to compute the real and imaginary parts of the transform, allowing for the construction of complex wavelets that are approximately analytic [S1]. It has been applied in various signal processing tasks and, more recently, in volume data representation for dynamic scenes, such as in the DaReNeRF framework [S1].

## Sources
[S1] DaReNeRF: Direction-aware Representation for Dynamic Scenes, https://doi.org/10.1109/cvpr52733.2024.00481, fetched 2026-10-27
