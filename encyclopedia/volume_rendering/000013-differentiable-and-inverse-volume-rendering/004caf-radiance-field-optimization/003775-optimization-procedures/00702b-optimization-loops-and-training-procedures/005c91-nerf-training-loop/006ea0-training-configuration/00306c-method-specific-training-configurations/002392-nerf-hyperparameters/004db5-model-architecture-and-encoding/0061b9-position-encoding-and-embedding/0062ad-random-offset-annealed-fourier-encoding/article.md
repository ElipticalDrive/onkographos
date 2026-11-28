# Random Offset Annealed Fourier Encoding

Also called: ROAFE

## Overview
Random Offset Annealed Fourier Encoding (ROAFE) is a positional encoding scheme for radiance fields that modifies standard Fourier encoding by introducing random Gaussian offsets to frequency basis functions and applying an annealing schedule to the frequencies [S1]. It solves the problem of axis-aligned frequency artifacts (striping) inherent in standard NeRF encoding while maintaining the optimization stability provided by frequency annealing [S1]. It is distinguished from standard NeRF encoding by the addition of random offsets and from BARF/Nerfies by the specific handling of these offsets within the annealing window [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D coordinate vector $\mathbf{x} \in \mathbb{R}^3$.
*   **Parameters:**
    *   $L$: Number of frequency bands. Default: 10 [S1].
    *   $\alpha$: Annealing parameter, ranging from $0$ to $L$. Annealed over 50,000 steps [S1].
    *   $\mathbf{R}$: Random offset matrix of size $L \times 3$, sampled from $\mathcal{N}(0, 0.1)$ [S1].
*   **Output:** An encoded vector $\gamma(\mathbf{x}) \in \mathbb{R}^{3 + 6L}$ [S1].

**Data Structures**
*   $\mathbf{R}$: A static matrix of shape $(L, 3)$ containing Gaussian random values.
*   $\alpha$: A scalar integer or float representing the current training step normalized to the frequency range.

**Formulas**
The encoding is defined as [S1]:
$$ \gamma(\mathbf{x}) = (\mathbf{x}, \Gamma_1, \dots, \Gamma_{L-1}) $$
where each frequency component $\Gamma_k$ for $k \in \{1, \dots, L-1\}$ is computed as [S1]:
$$ \Gamma_k(\mathbf{x}; \alpha) = w_k(\alpha) \left[ \sin(2^k \mathbf{x} + 2^k \mathbf{R}_k), \cos(2^k \mathbf{x} + 2^k \mathbf{R}_k) \right] $$
The annealing weight $w_k(\alpha)$ acts as a truncated Hann window [S1]:
$$ w_k(\alpha) = \frac{1 - \cos(\pi \cdot \text{clamp}(\alpha - k, 0, 1))}{2} $$
where $\text{clamp}(v, 0, 1)$ restricts $v$ to the interval $[0, 1]$ [S1].

**Direction Encoding**
View directions $\mathbf{d}$ are encoded separately using 4 non-annealed and non-offset Fourier frequencies [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def roafe_encode(x, L=10, alpha=0, R=None):
    """
    Encode a 3D coordinate using Random Offset Annealed Fourier Encoding.
    
    Args:
        x: numpy array of shape (3,), the 3D coordinate.
        L: int, number of frequency bands.
        alpha: float, annealing parameter (0 to L).
        R: numpy array of shape (L, 3), random offsets. If None, generated deterministically for reproducibility in this example.
        
    Returns:
        numpy array of shape (3 + 6*(L-1),), the encoded vector.
    """
    if R is None:
        # Deterministic seed for reproducibility in implementation example
        rng = np.random.RandomState(42)
        R = rng.normal(0, 0.1, size=(L, 3))
        
    # Initialize output with raw coordinates
    # Note: The source defines gamma(x) = (x, Gamma_1, ..., Gamma_{L-1})
    # This implies the raw x is included, and frequencies start from index 1 to L-1.
    # However, standard NeRF often uses 0 to L-1. The source explicitly writes Gamma_1 to Gamma_{L-1}.
    # We follow the source notation strictly.
    
    encoded_parts = [x]
    
    for k in range(1, L):
        # Calculate base frequency scaling factor 2^k
        # Source text has garbled '2k' but context implies 2^k or similar scaling.
        # The formula block shows 2^k.
        freq_scale = 2 ** k
        
        # Get random offset for this frequency band
        # R_k is the k-th row of R. Note: R is size Lx3.
        # If k goes 1..L-1, we access R[1]..R[L-1].
        # The source says R is sampled from N(0, 0.1).
        offset = R[k]
        
        # Compute the argument for sin/cos: 2^k * x + 2^k * R_k
        # This is equivalent to 2^k * (x + R_k)
        arg = freq_scale * (x + offset)
        
        # Compute sine and cosine components
        sin_comp = np.sin(arg)
        cos_comp = np.cos(arg)
        
        # Concatenate sin and cos for this band
        band_output = np.concatenate([sin_comp, cos_comp])
        
        # Compute annealing weight w_k(alpha)
        # clamp(alpha - k, 0, 1)
        clamp_val = max(0.0, min(1.0, alpha - k))
        weight = (1.0 - math.cos(math.pi * clamp_val)) / 2.0
        
        # Apply weight
        weighted_band = weight * band_output
        
        encoded_parts.append(weighted_band)
        
    return np.concatenate(encoded_parts)
```

**Edge Cases and Pitfalls**
*   **Annealing Boundary:** When $\alpha < k$, the weight $w_k(\alpha)$ is 0, effectively disabling higher frequencies early in training. When $\alpha \ge k+1$, the weight is 1. The transition occurs between $\alpha=k$ and $\alpha=k+1$ [S1].
*   **Direction Encoding:** Directions use a fixed 4-band encoding without annealing or offsets, distinct from position encoding [S1].
*   **Offset Initialization:** Offsets are sampled from $\mathcal{N}(0, 0.1)$. These should be fixed after initialization to avoid non-deterministic behavior during training [S1].

## Operating characteristics
*   **Complexity:** Time complexity is linear with respect to the number of frequency bands $L$ and the number of samples. Memory complexity is $O(L \cdot D)$ for storing the offset matrix $\mathbf{R}$, where $D=3$ is the input dimension.
*   **Performance:** Improves optimization landscape by breaking axis-alignment, reducing striping artifacts compared to standard NeRF [S1].
*   **Strengths:** Combines the benefits of random frequency rotation (reducing artifacts) with annealing (stabilizing optimization) [S1].
*   **Limitations:** Requires careful tuning of the annealing schedule (50,000 steps in SAMURAI) and offset variance (0.1) [S1].

## Background
ROAFE was introduced in the SAMURAI paper (2022) [S1]. It builds upon the Fourier encoding used in NeRF [S1], the annealing technique from BARF and Nerfies [S1], and the concept of random frequency offsets demonstrated by Tancik et al. [S1]. It was developed to handle unconstrained real-world image collections where axis-aligned artifacts are more pronounced [S1].

## Sources
[S1] SAMURAI: Shape and Material from Unconstrained Real-World Arbitrary Image Collections, https://doi.org/10.52202/068431-1914, fetched 2026-10-29
