# Multiple Octaves Phase Function
Also called: Composite Henyey-Greenstein Phase Function, Multi-Octave HG Phase Function

## Overview
The Multiple Octaves Phase Function is a composite phase function model that combines multiple Henyey-Greenstein lobes with varying asymmetry parameters and weights to create complex scattering patterns, specifically a two-lobed effect [S1, unofficial]. It sets itself apart from its parent [Henyey-Greenstein Phase Function](1010-henyey-greenstein-phase-function.md) by allowing for non-symmetric, multi-modal scattering distributions through a weighted sum of components, rather than relying on a single asymmetry parameter $g$. It differs from the [Dual-Lobe Henyey-Greenstein Phase Function](5231-dual-lobe-henyey-greenstein-phase-function.md) by implementing a specific four-octave iterative accumulation scheme rather than a static two-lobe definition, and from the [Weighted Sum of Phase Functions](6650-weighted-sum-of-phase-functions.md) by using a fixed, hardcoded structure of four iterations with specific decay factors [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: Scattering direction $\omega_s$, incident direction $\omega_i$, optical depth $\tau$, density $\rho$, extinction coefficient $\sigma_t$.
*   Output: Accumulated radiance contribution $L$.

**Data Structures:**
No complex data structures are required. The method relies on scalar accumulators and loop counters.

**Parameters:**
*   Number of octaves: 4 (fixed) [S1, unofficial].
*   Initial weights: $a=1.0, b=1.0, c=1.0$ [S1, unofficial].
*   Decay factors: $a_{decay}=0.2, b_{decay}=0.5, c_{decay}=0.5$ [S1, unofficial].
*   Lobe weights: $w_{forward}=0.7, w_{backward}=0.3$ [S1, unofficial].
*   Lobe asymmetry base: $g_{base\_fwd}=0.3, g_{base\_bwd}=-0.1$ [S1, unofficial].

**Steps:**
1.  Initialize accumulators: $L = 0$, $a = 1.0$, $b = 1.0$, $c = 1.0$.
2.  Loop for $i = 0$ to $3$:
    a. Calculate asymmetry parameters for the current octave:
       $$g_{fwd} = 0.3 \cdot c$$
       $$g_{bwd} = -0.1 \cdot c$$
    b. Evaluate the Henyey-Greenstein phase function $p_{HG}(\omega_s, \omega_i, g)$ for both lobes.
    c. Compute the composite phase value $P$:
       $$P = 0.3 \cdot p_{HG}(\omega_s, \omega_i, g_{bwd}) + 0.7 \cdot p_{HG}(\omega_s, \omega_i, g_{fwd})$$
    d. Calculate the attenuation term:
       $$Att = \exp(-\sigma_t \cdot \rho \cdot a)$$
    e. Add contribution to radiance:
       $$L \leftarrow L + b \cdot P \cdot Att$$
    f. Update factors:
       $$a \leftarrow a \cdot 0.2$$
       $$b \leftarrow b \cdot 0.5$$
       $$c \leftarrow c \cdot 0.5$$
3.  Return $L$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multiple_octaves_phase_function(omega_s, omega_i, sigma_t, density, g_hg):
    """
    Computes radiance contribution using the Multiple Octaves Phase Function.
    Note: This implementation assumes the Henyey-Greenstein phase function
    is available as a helper.
    """
    # Helper: Standard Henyey-Greenstein Phase Function
    def heney_greenstein(omega_s, omega_i, g):
        cos_theta = dot(omega_s, omega_i)
        g_sq = g * g
        denom = 1.0 + g_sq - 2.0 * g * cos_theta
        # Avoid division by zero if denom is extremely small
        if denom < 1e-8:
            denom = 1e-8
        return (1.0 - g_sq) / (4.0 * pi * (denom ** 1.5))

    L = 0.0
    a = 1.0
    b = 1.0
    c = 1.0

    for i in range(4):
        # Calculate asymmetry parameters for current octave
        g_fwd = 0.3 * c
        g_bwd = -0.1 * c

        # Evaluate phase functions
        p_fwd = heney_greenstein(omega_s, omega_i, g_fwd)
        p_bwd = heney_greenstein(omega_s, omega_i, g_bwd)

        # Composite phase: 0.3 weight for backward, 0.7 for forward
        phase = 0.3 * p_bwd + 0.7 * p_fwd

        # Attenuation factor
        attenuation = exp(-sigma_t * density * a)

        # Accumulate radiance
        L += b * phase * attenuation

        # Update factors for next octave
        a *= 0.2
        b *= 0.5
        c *= 0.5

    return L
```

**Edge Cases and Pitfalls:**
*   **Precision:** The exponential term $\exp(-\sigma_t \cdot \rho \cdot a)$ can underflow to zero if $\sigma_t \cdot \rho$ is large and $a$ is not sufficiently small, causing contributions to vanish. Conversely, if $a$ remains large, precision loss may occur.
*   **Boundary Conditions:** The Henyey-Greenstein function is undefined for $g=1$ and $\cos \theta = 1$ (singularity). The pseudocode includes a guard for this [G].
*   **Degenerate Input:** If density or $\sigma_t$ is zero, the attenuation is 1, and the result is purely the weighted sum of phase functions.

## Subtopics

### [Dual-Lobe Henyey-Greenstein Phase Function](5231-dual-lobe-henyey-greenstein-phase-function.md)
Uses a fixed two-lobe structure for visual aesthetics.

### [Weighted Sum of Phase Functions](6650-weighted-sum-of-phase-functions.md)
General framework for any number of lobes.

## Operating characteristics
*   **Time Complexity:** $O(1)$ with a constant factor of 4 Henyey-Greenstein evaluations and arithmetic operations.
*   **Memory Complexity:** $O(1)$, requiring only scalar variables.
*   **Performance:** Very fast due to fixed iteration count and simple math. Suitable for real-time rendering.
*   **Strengths:** Captures complex scattering patterns (forward and backward lobes) with minimal overhead.
*   **Limitations:** The specific weights and decay factors are hardcoded, limiting flexibility. Not physically accurate for all media .
*   **When to use:** When a simple, fast approximation of multi-modal scattering is needed for visual effect.
*   **When not to use:** When physical accuracy is required or when the scattering profile differs significantly from the hardcoded parameters.

## Background
The Multiple Octaves Phase Function originates from the `al-ro/volumetrics` repository, an unofficial implementation [S1, unofficial]. It is not derived from a peer-reviewed paper but is a practical technique used in real-time graphics for volumetric effects. It builds on the [Henyey-Greenstein Phase Function](1010-henyey-greenstein-phase-function.md) by stacking multiple instances with decaying parameters.

## Sources
[S1] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
