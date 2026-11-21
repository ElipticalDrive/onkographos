# B-spline Wavelet Basis Construction

Also called: Cardinal B-spline wavelet construction, Chui and Wang B-spline wavelets

## Overview
This method constructs the spatial and frequency-domain basis functions for Chui and Wang B-spline wavelets of order $j$, which are used for volume data compression and rendering [S1, unofficial]. It differs from orthogonal wavelet bases by employing semi-orthogonal duals to ensure perfect reconstruction while maintaining the smoothness and vanishing moments of B-splines [S1, unofficial]. It is chosen when analytic expressions in both spatial and frequency domains are required for efficient splatting or filtering operations [S1, unofficial].

## How it works
**Inputs:**
*   Order of the B-spline wavelet $j$ (integer, $j \ge 1$).

**Outputs:**
*   Scaling function $\phi^{[j]}(x)$ in spatial domain.
*   Wavelet function $\psi^{[j]}(x)$ in spatial domain.
*   Frequency-domain representation $\Phi^{[j]}(f)$.
*   Dual functions (implicitly required for decomposition, though explicit construction formulas are not provided in the source text).

**Data Structures:**
*   Coefficients $q_{j,k}$: Array of size $3j-1$ (indices $k=0$ to $3j-2$).
*   Binomial coefficients: Standard combinatorial values.

**Steps:**

1.  **Define the base scaling function (Haar wavelet):**
    For $j=1$, the mother-scaling function is defined as:
    $$
    \phi^{[1]}(x) = \begin{cases} 1 & 0 \leq x < 1 \\ 0 & \text{otherwise} \end{cases}
    $$

2.  **Recursively generate scaling functions for order $j$:**
    For $j > 1$, compute $\phi^{[j]}(x)$ using the recursive relation:
    $$
    \phi^{[j]}(x) = \frac{x}{j-1} \phi^{[j-1]}(x) + \frac{j-x}{j-1} \phi^{[j-1]}(x-1)
    $$
    This generates compactly supported scaling functions in terms of cardinal B-splines.

3.  **Compute frequency-domain representation:**
    The Fourier transform of the scaling function is given by the sinc-polynomial:
    $$
    \Phi^{[j]}(f) = \left( \frac{1 - e^{-i2\pi f}}{i2\pi f} \right)^j
    $$
    where $f$ is the frequency and $i$ is the complex operator. This form is fundamental for splat computation in the Fourier domain [S1, unofficial].

4.  **Construct the wavelet function:**
    The wavelet function $\psi^{[j]}(x)$ is a linear combination of scaled and shifted scaling functions:
    $$
    \psi^{[j]}(x) = \sum_{k=0}^{3j-2} q_{j,k} \phi^{[j]}(2x-k)
    $$
    The support of the wavelet function is $[0, 2j-1]$ [S1, unofficial].

5.  **Calculate wavelet coefficients $q_{j,k}$:**
    Compute the coefficients using the formula:
    $$
    q_{j,k} = \frac{(-1)^k}{2^{j-1}} \sum_{l=0}^j \binom{j}{l} \phi^{[2j]}(k+1-l)
    $$
    Note that this requires the scaling function of order $2j$.

6.  **Extension to 3D:**
    Three-dimensional wavelets are obtained by non-standard tensor-product extensions of the 1D wavelets [S1, unofficial].

7.  **Decomposition and Reconstruction:**
    *   **Decomposition:** Volume decomposition is carried out using the **dual** wavelet functions.
    *   **Reconstruction:** Reconstruction is performed using the **primary** scaling and wavelet functions defined in steps 2 and 4 [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import List, Callable

def binomial(n: int, k: int) -> int:
    """Compute binomial coefficient C(n, k)."""
    if k < 0 or k > n:
        return 0
    if k == 0 or k == n:
        return 1
    if k > n // 2:
        k = n - k
    
    res = 1
    for i in range(k):
        res = res * (n - i) // (i + 1)
    return res

def scaling_function_1d(x: float, order: int) -> float:
    """
    Compute the value of the B-spline scaling function of given order at x.
    Uses recursive definition.
    """
    if order == 1:
        # Haar scaling function
        if 0 <= x < 1:
            return 1.0
        else:
            return 0.0
    
    # Recursive step: phi^[j](x) = (x/(j-1)) * phi^[j-1](x) + ((j-x)/(j-1)) * phi^[j-1](x-1)
    prev_order = order - 1
    term1 = (x / prev_order) * scaling_function_1d(x, prev_order)
    term2 = ((order - x) / prev_order) * scaling_function_1d(x - 1, prev_order)
    return term1 + term2

def compute_wavelet_coefficients(order: int) -> List[float]:
    """
    Compute coefficients q_{j,k} for wavelet function of order j.
    Support is k from 0 to 3j-2.
    """
    j = order
    num_coeffs = 3 * j - 1  # indices 0 to 3j-2
    coeffs = []
    
    # Precompute scaling function of order 2j, as it is needed for all k
    # Note: The formula uses phi^[2j](k+1-l). We need to evaluate this function.
    # Since phi is compactly supported, we only need to evaluate it where it is non-zero.
    
    for k in range(num_coeffs):
        sum_val = 0.0
        for l in range(j + 1):  # l from 0 to j
            # Argument for phi^[2j]
            arg = k + 1 - l
            phi_val = scaling_function_1d(arg, 2 * j)
            sum_val += binomial(j, l) * phi_val
        
        q_jk = ((-1) ** k) / (2 ** (j - 1)) * sum_val
        coeffs.append(q_jk)
        
    return coeffs

def wavelet_function_1d(x: float, order: int) -> float:
    """
    Compute the value of the B-spline wavelet function of order j at x.
    psi^[j](x) = sum_{k=0}^{3j-2} q_{j,k} * phi^[j](2x - k)
    """
    j = order
    coeffs = compute_wavelet_coefficients(j)
    result = 0.0
    
    for k, q in enumerate(coeffs):
        # Evaluate phi^[j] at 2x - k
        phi_val = scaling_function_1d(2 * x - k, j)
        result += q * phi_val
        
    return result

def frequency_domain_scaling(order: int, f: float) -> complex:
    """
    Compute the frequency domain representation Phi^[j](f).
    Phi^[j](f) = ((1 - exp(-i*2*pi*f)) / (i*2*pi*f))^j
    """
    if f == 0:
        # Limit as f->0 is 1^j = 1
        return complex(1.0, 0.0)
    
    numerator = 1 - math.exp(-1j * 2 * math.pi * f)
    denominator = 1j * 2 * math.pi * f
    base = numerator / denominator
    return base ** order
```

**Edge Cases and Pitfalls:**
*   **Precision:** The recursive calculation of scaling functions may suffer from numerical instability for high orders $j$. The frequency domain formula involves complex exponentials and division by $f$, requiring careful handling of $f=0$ [S1, unofficial].
*   **Dual Functions:** The source states that decomposition requires dual functions, but does not provide their explicit construction formulas. Implementers must refer to external sources (e.g., Chui and Wang [3]) for the dual basis construction [S1, unofficial].
*   **3D Extension:** The "non-standard tensor-product extension" is not defined in the source. Standard tensor products are common, but "non-standard" implies a specific ordering or filtering strategy. This detail is missing [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The source mentions that linear time algorithms [20] can be more challenging due to additional basis transforms [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Smoothness, vanishing moments, analytic expressions in frequency and spatial domains, semi-orthogonality for $j>1$, linear phase for primary and dual functions, rational coefficients for fast wavelet transform [S1, unofficial].
*   **Limitations:** Requires dual functions for decomposition, which are not explicitly constructed here. Linear time implementations are complex [S1, unofficial].
*   **When to use:** When volume data compression and rendering require smooth bases with analytic frequency-domain representations for splatting [S1, unofficial].
*   **When not to use:** When orthogonal bases are strictly required, or when the complexity of dual function construction is prohibitive.

## Background
*   **Origin:** The method is based on Chui and Wang B-spline wavelet bases [S1, unofficial].
*   **History:** The source cites Chui and Wang [3] as the origin of the B-spline wavelet bases used for volume data compression and rendering [S1, unofficial].
*   **Usage:** Used in compression domain volume rendering, specifically in wavelet splatting [S1, unofficial].

## Sources
[S1] Compression Domain Volume Rendering for Distributed ..., https://cgl.ethz.ch/Downloads/Publications/Tutorials/1998/tu_Gro98-4.pdf, fetched 2026-10-26 (unofficial)
