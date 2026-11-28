# Neumann Series Expansion

Also called: Neumann Series, Neumann Series Solution

## Overview
The Neumann Series Expansion solves the Volume Rendering Integral by expressing the incoming radiance at a point as an infinite sum of contributions from light sources that have undergone $0, 1, 2, \dots$ scattering events [S1, unofficial]. It operates directly on the integral equation operators rather than explicitly sampling path geometries [S1, unofficial].

## How it works
**Inputs and Outputs:** Input consists of the transport operator $\mathbf{G}$, the scattering operator $\mathbf{K}$, and the emitted radiance $L_e$ [S1, unofficial]. The output is the incoming radiance $L_i$ at a specific point and direction [S1, unofficial].

**Data Structures:** The method relies on continuous operator notation rather than discrete arrays [S1, unofficial]. $\mathbf{G}$ represents free-streaming and extinction [S1, unofficial]. $\mathbf{K}$ represents the integral over the unit sphere weighted by the phase function and scattering coefficient [S1, unofficial]. $L_e$ represents emitted radiance from volume or surface sources [S1, unofficial].

**Numbered Steps:** 1. Compute the direct term by applying $\mathbf{G}$ to $L_e$ [S1, unofficial]. 2. Iteratively compute higher-order terms by applying the composite operator $\mathbf{G}\mathbf{K}$ to the result of the previous step [S1, unofficial]. 3. Accumulate the terms into a sum until a chosen truncation depth $N$ is reached [S1, unofficial].

**Formulas:** The incoming radiance is defined as:
$$L_i = \sum_{n=0}^\infty (\mathbf{G}\mathbf{K})^n \mathbf{G} L_e = \mathbf{G} L_e + \mathbf{G}\mathbf{K}\mathbf{G} L_e + (\mathbf{G}\mathbf{K})^2 \mathbf{G} L_e + \cdots$$ [S1, unofficial].

**Symbol Definitions:** $n$ denotes the scattering order [S1, unofficial]. $\mathbf{G} L_e$ represents light traveling from the source without scattering but still undergoing attenuation [S1, unofficial]. $\mathbf{G}\mathbf{K}\mathbf{G} L_e$ represents light that scatters once [S1, unofficial]. $(\mathbf{G}\mathbf{K})^n \mathbf{G} L_e$ represents light that has undergone $n$ scattering events [S1, unofficial].

**Parameters:** The truncation depth $N$ controls the number of scattering terms computed; typical values are not given in the sources [S1, unofficial].

**Edge Cases and Pitfalls:** Attenuation via $\mathbf{G}$ is applied even at $n=0$ [S1, unofficial]. Truncating the infinite series introduces bias into the result [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neumann_series_expansion(L_e, G, K, max_terms):
    """
    Computes incoming radiance L_i using the Neumann Series Expansion.
    
    Args:
        L_e: Emitted radiance function or value.
        G: Transport operator function (attenuation + streaming).
        K: Scattering operator function (integral over sphere).
        max_terms: Maximum number of scattering events to compute.
        
    Returns:
        L_i: Approximated incoming radiance.
    """
    L_i = 0.0
    term = G(L_e)
    L_i += term
    for _ in range(1, max_terms):
        term = G(K(term))
        L_i += term
    return L_i
```

## Subtopics

### [Path Integral Formulation](14634-path-integral-formulation.md)
Expresses the integral as a sum over paths of scattering events, allowing for arbitrary sequences of 2D surface locations and 3D positions in a participating medium. It transforms the measurement equation into a sum over path lengths $k$, where each term is an integral over a specific path space $\mathcal{P}_k^{\vec{c}}$. It is a prerequisite for constructing participating medium-aware bidirectional integrators. Choose this when explicit path sampling is required for variance reduction in complex scenes.

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Strengths:** Provides a clear physical interpretation of light transport by separating contributions by scattering order [S1, unofficial].
*   **Limitations:** Not given in the sources [S1, unofficial].
*   **When to use:** Not given in the sources [S1, unofficial].

## Background
The Neumann Series Expansion is derived by recursively substituting the volume rendering equation into itself [S1, unofficial]. Other historical details are not given in the sources [S1, unofficial].

## Sources
[S1] ボリュームレンダリング方程式(Volume Rendering Equation) 2, https://rayspace.xyz/notes/VLTE2/, fetched 2026-10-30 (unofficial)
