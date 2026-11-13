# Windowed Fourier Transform

Also called: Gabor Transform

## Overview
The Windowed Fourier Transform (WFT), also known as the Gabor Transform, addresses the lack of spatial localization inherent in the standard Fourier Transform [S1, unofficial]. While the standard Fourier Transform provides a measure of high-frequency irregularities over the entire spatial domain, it makes it difficult to determine the position of these irregularities [S1, unofficial]. The WFT solves this by using a translated window function within the Fourier integral, allowing for the localization of signal features across the domain [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A function $f(x) \in L^2(\mathbb{R})$ representing the signal or volume data.
*   **Input:** A window function $g(x)$, typically a Gaussian or similar localized function.
*   **Input:** Translation parameter $u$ (spatial position) and frequency parameter $w$.
*   **Output:** The complex-valued transform coefficient $Gf(u, w)$.

**Data Structures**
The output is a 2D array (or 3D if considering time $t$ in $Gf(u, t, w)$) storing coefficients indexed by spatial position $u$ and frequency $w$.

**Formulas**
The Windowed Fourier Transform of a function $f(x)$ is defined by [S1, unofficial]:

$$ Gf(u, w) = \int_{-\infty}^{\infty} f(x) g(x - u) e^{-iwx} dx $$

Where:
*   $f(x)$ is the input signal.
*   $g(x - u)$ is the window function centered at position $u$.
*   $e^{-iwx}$ is the complex exponential kernel.
*   $u$ is the translation parameter (spatial location).
*   $w$ is the frequency parameter.

**Steps**
1.  Select a window function $g(x)$ with compact support or rapid decay (e.g., Gaussian).
2.  For each desired spatial position $u$:
    a. Shift the window to $g(x - u)$.
    b. Multiply the input signal $f(x)$ by the shifted window $g(x - u)$.
    c. Compute the Fourier Transform of the product $f(x)g(x - u)$ with respect to frequency $w$.
3.  Store the resulting complex coefficient $Gf(u, w)$ for the specific $u$ and $w$.
4.  Repeat for all $u$ and $w$ to cover the domain.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def windowed_fourier_transform(f, g, u_values, w_values):
    """
    Computes the Windowed Fourier Transform (Gabor Transform).
    
    Args:
        f: Array of input signal values f(x) at spatial points x.
        g: Array of window function values g(x) at spatial points x.
        u_values: Array of translation positions u.
        w_values: Array of frequency values w.
        
    Returns:
        Gf: 2D array of coefficients Gf(u, w).
    """
    # Assume x is defined on a grid corresponding to f and g
    # x = np.linspace(x_min, x_max, len(f))
    
    Gf = np.zeros((len(u_values), len(w_values)), dtype=complex)
    
    for i, u in enumerate(u_values):
        # Shift the window to position u
        # g_shifted[x] = g[x - u]
        # Note: In discrete implementation, this requires interpolation or array indexing
        g_shifted = np.roll(g, int(u / dx)) if u is discrete step else interpolate_shift(g, u)
        
        # Multiply signal by shifted window
        windowed_signal = f * g_shifted
        
        for j, w in enumerate(w_values):
            # Compute Fourier component at frequency w
            # Integral approximation: sum(windowed_signal * exp(-i*w*x) * dx)
            kernel = np.exp(-1j * w * x)
            Gf[i, j] = np.sum(windowed_signal * kernel) * dx
            
    return Gf
```

**Edge Cases and Pitfalls**
*   **Boundary Effects:** The window function must decay to zero at its edges to minimize artifacts when the window moves near the boundaries of the signal domain [S1, unofficial].
*   **Resolution Trade-off:** The width of the window $g(x)$ determines the trade-off between spatial and frequency resolution. A narrow window provides good spatial localization but poor frequency resolution, and vice versa [S1, unofficial].

## Subtopics

### [Aperture Apodization](10803-aperture-apodization.md)
A technique used in numerical holographic reconstruction to suppress side-lobe leakage and diffraction artifacts caused by sharp intensity edges of a rectangular synthetic aperture. It works by smoothly tapering the aperture edges to zero. It is applied after selecting a sub-region of the hologram to define a specific viewpoint.

## Operating characteristics
*   **Complexity:** The computational cost is $O(N \cdot M \cdot K \log K)$ or $O(N \cdot M \cdot K)$ depending on implementation, where $N$ is the number of spatial samples, $M$ is the number of translation positions $u$, and $K$ is the number of frequency samples $w$. This is significantly higher than the standard FFT.
*   **Memory:** Requires storing the 2D (or 3D) coefficient array, which can be large for high-resolution spatial and frequency sampling.
*   **Strengths:** Provides simultaneous time/space and frequency information, allowing for the localization of irregularities [S1, unofficial].
*   **Limitations:** Higher computational cost than standard Fourier methods. Choice of window function affects results.

## Background
The Windowed Fourier Transform was defined by Gabor [15] (cited in [S1, unofficial]) to localize the information provided by the Fourier Transform. It is a classical theory for multiresolution decomposition [S1, unofficial]. It is used in direct rendering of multiresolution decomposed volume data [S1, unofficial].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
