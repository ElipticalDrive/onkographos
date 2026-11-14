# Neumann Series Expansion

Also called: Neumann Series, Neumann Series Solution

## Overview
The Neumann Series Expansion solves the Volume Rendering Integral by expressing the incoming radiance at a point as an infinite sum of contributions from light sources that have undergone $0, 1, 2, \dots$ scattering events [S1, unofficial]. It sets itself apart from [Spherical Harmonic Volume Rendering](11250-spherical-harmonic-volume-rendering.md) by avoiding angular basis function approximations, and from [Path Integral Formulation](14634-path-integral-formulation.md) by operating directly on the integral equation operators rather than explicitly sampling path geometries [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The volume rendering equation in operator form, specifically the extinction/transport operator $\mathbf{G}$, the scattering operator $\mathbf{K}$, and the emitted radiance $L_e$.
*   **Output:** The incoming radiance $L_i$ at a specific point and direction.

**Data Structures:**
The method relies on operator notation rather than explicit discrete data structures in its definition.
*   $\mathbf{G}$: The transport operator, representing free-streaming and extinction.
*   $\mathbf{K}$: The scattering operator, representing the integral over the sphere of directions weighted by the phase function and scattering coefficient.
*   $L_e$: The emitted radiance (from volume sources or surface sources entering the volume).

**Formulas:**
The incoming radiance $L_i$ is defined as the sum of terms where light is attenuated by $\mathbf{G}$ and scattered by $\mathbf{K}$:

$$
L_i = \sum_{n=0}^\infty (\mathbf{G}\mathbf{K})^n \mathbf{G} L_e
$$

Expanding the first few terms:

$$
L_i = \mathbf{G} L_e + \mathbf{G}\mathbf{K}\mathbf{G} L_e + (\mathbf{G}\mathbf{K})^2 \mathbf{G} L_e + \cdots
$$

**Symbol Definitions:**
*   $n$: The number of scattering events.
*   $\mathbf{G} L_e$: Represents light that travels from the source to the point without scattering ($n=0$), but still undergoes attenuation.
*   $\mathbf{G}\mathbf{K}\mathbf{G} L_e$: Represents light that scatters once ($n=1$).
*   $(\mathbf{G}\mathbf{K})^n \mathbf{G} L_e$: Represents light that has undergone $n$ scattering events.

**Steps:**
1.  Identify the emitted radiance $L_e$ contributing to the point of interest.
2.  Apply the transport operator $\mathbf{G}$ to $L_e$ to get the direct (unscattered) contribution.
3.  For each subsequent term $n=1, 2, \dots$:
    a. Apply the scattering operator $\mathbf{K}$ to the result of the previous term's $\mathbf{G}$ application.
    b. Apply the transport operator $\mathbf{G}$ to the result of $\mathbf{K}$.
    c. Add this term to the sum.
4.  Sum the terms until convergence or a desired truncation depth $N$ is reached.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neumann_series_expansion(L_e, G, K, max_terms=10):
    """
    Computes the incoming radiance L_i using the Neumann Series Expansion.
    
    Args:
        L_e: Emitted radiance function or value.
        G: Transport operator function (attenuation + streaming).
        K: Scattering operator function (integral over sphere).
        max_terms: Maximum number of scattering events to compute.
        
    Returns:
        L_i: Approximated incoming radiance.
    """
    L_i = 0.0
    # Term n=0: G * L_e
    current_term = G(L_e)
    L_i += current_term
    
    for n in range(1, max_terms):
        # Term n: (G*K)^n * G * L_e
        # Previous term was (G*K)^(n-1) * G * L_e
        # We need to apply K then G to the previous term's result before G?
        # Actually, the formula is sum_{n=0} (G K)^n G L_e.
        # Let T_0 = G L_e
        # Let T_n = G K T_{n-1}
        
        # Apply K to the previous term's "input" to G? 
        # No, the term is (G K)^n G L_e.
        # T_0 = G L_e
        # T_1 = G K T_0
        # T_2 = G K T_1
        
        # So we apply K to the result of the previous G application?
        # Wait, T_0 = G L_e.
        # T_1 = G K (G L_e).
        # So we need to apply K to (G L_e).
        
        # Let's track the "source" for the next G.
        # For n=0, source is L_e. Result is G(L_e).
        # For n=1, source is K(G(L_e)). Result is G(K(G(L_e))).
        
        # Let's rewrite loop to be clearer.
        pass

    # Correct implementation based on L_i = sum_{n=0} (G K)^n G L_e
    L_i = 0.0
    # n=0 term
    term = G(L_e)
    L_i += term
    
    for n in range(1, max_terms):
        # The term is (G K)^n G L_e
        # This is G K [ (G K)^(n-1) G L_e ]
        # So we take the previous term, apply K, then apply G?
        # No. (G K)^n = G K G K ... G K (n times).
        # The formula is (G K)^n G L_e.
        # Let A = G K.
        # Term_n = A^n G L_e.
        # Term_0 = G L_e.
        # Term_1 = A Term_0 = G K Term_0.
        # Term_2 = A Term_1 = G K Term_1.
        
        # So:
        term = G(K(term))
        L_i += term
        
    return L_i
```

**Edge Cases and Pitfalls:**
*   **Convergence:** The series only converges if the spectral radius of the operator $\mathbf{G}\mathbf{K}$ is less than 1. In highly scattering media, convergence may be slow or non-existent [S1, unofficial].
*   **Attenuation:** Even with 0 scatterings, attenuation ($\mathbf{G}$) must be applied. Omitting the initial $\mathbf{G}$ is a common error [S1, unofficial].
*   **Truncation:** Truncating the series introduces bias. The number of terms required depends on the optical thickness of the medium.

## Subtopics

### [Path Integral Formulation](14634-path-integral-formulation.md)
Expresses the integral as a sum over paths of scattering events, allowing for arbitrary sequences of 2D surface locations and 3D positions. It is a prerequisite for constructing participating medium-aware bidirectional integrators. Choose this when explicit path sampling is required for variance reduction in complex scenes.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of terms $N$ if each term is computed independently. However, computing each term involves an integral over the sphere, making it expensive.
*   **Memory Complexity:** Low, as it does not require storing large path histories or basis functions, only the current term's radiance.
*   **Strengths:** Provides a clear physical interpretation of light transport (0 scatter, 1 scatter, etc.). It is exact in the limit of infinite terms.
*   **Limitations:** Convergence can be slow in highly scattering media. It is not directly suitable for Monte Carlo integration without further transformation (e.g., to the Path Integral Formulation).
*   **When to use:** When an analytical understanding of scattering orders is needed, or as a basis for deriving other methods. Not typically used directly for rendering due to slow convergence.

## Background
The Neumann Series Expansion originates from the mathematical theory of integral equations, specifically the solution of Fredholm integral equations of the second kind. In the context of volume rendering, it is derived by recursively substituting the volume rendering equation into itself [S1, unofficial]. It is a fundamental theoretical tool in computational radiative transfer and computer graphics, providing the basis for understanding light transport in participating media.

## Sources
[S1] ボリュームレンダリング方程式(Volume Rendering Equation) 2, https://rayspace.xyz/notes/VLTE2/, fetched 2026-10-30 (unofficial)
