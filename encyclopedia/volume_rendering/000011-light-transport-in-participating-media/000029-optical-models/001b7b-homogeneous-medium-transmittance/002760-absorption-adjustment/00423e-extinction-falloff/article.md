# Extinction Falloff

Also called: Extinction Falloff Value

## Overview
Extinction Falloff is a rendering shortcut that approximates the visual contribution of hundreds of additional scattering bounces by gently reducing the extinction coefficient with each real bounce [S1, unofficial][S2]. It allows scenes to achieve brighter, more realistic volumetric lighting (such as internal light spill in fire or smoke) while rendering faster than equivalent scenes without the technique, avoiding the need to increase trace depth [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** The current extinction coefficient $\sigma_t$ of the medium, the current bounce depth $n$, and the Extinction Falloff Value parameter $\alpha$.
*   **Output:** The effective extinction coefficient $\sigma_t'$ used for transmittance calculation at the current bounce.

**Data Structures:**
No complex data structures are required. The method operates on scalar values associated with the ray's current state.

**Parameters:**
*   **Extinction Falloff Value ($\alpha$):** A scalar multiplier. The default value is 0.2, described as a "deliberate sweet spot" [S1, unofficial][S2].

**Method Steps:**
1.  Initialize the effective extinction $\sigma_t'$ to the medium's base extinction $\sigma_t$.
2.  For each scattering bounce $n$ (starting from $n=1$ for the first scatter):
    a. Apply the falloff to the extinction. The sources describe this as "gently reducing extinction with each real bounce" [S1, unofficial][S2].
    b. While the exact mathematical formula is not explicitly provided in the cited documentation, the behavior implies a multiplicative reduction or a decay function applied to $\sigma_t$ based on $\alpha$ and $n$. A standard implementation consistent with "reducing extinction" would be:
       $$ \sigma_t' = \sigma_t \times (1 - \alpha)^{n-1} $$
       Or potentially:
       $$ \sigma_t' = \sigma_t \times (1 - \alpha \cdot n) $$
       *Note: The sources do not specify the exact algebraic form, only the effect and the default value [S1, unofficial][S2]. Implementers should verify the specific formula in the host renderer's API or documentation if available.*
    c. Use $\sigma_t'$ to calculate transmittance $T = e^{-\sigma_t' \cdot \Delta s}$ for the current segment.
3.  Proceed to the next bounce with the updated state.

**Edge Cases and Pitfalls:**
*   **Precision:** If $\alpha$ is too high, extinction may drop to zero or negative values, causing numerical instability or incorrect light transmission.
*   **Boundaries:** The falloff applies per bounce. It does not replace the need for proper boundary handling in the volume shader.
*   **Degenerate Input:** If the base extinction is zero, the falloff has no effect.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_effective_extinction(base_extinction, bounce_depth, falloff_value):
    """
    Calculates the effective extinction coefficient for a given bounce.
    
    Args:
        base_extinction (float): The medium's original extinction coefficient.
        bounce_depth (int): The current scattering bounce number (1-based).
        falloff_value (float): The Extinction Falloff Value parameter (default 0.2).
        
    Returns:
        float: The reduced extinction coefficient for this bounce.
    """
    # The exact formula is not specified in the sources [S1, unofficial][S2].
    # This implementation assumes a geometric decay model consistent with
    # "gently reducing extinction with each real bounce".
    
    if base_extinction <= 0:
        return base_extinction
        
    # Apply falloff based on bounce depth
    # Note: This is a plausible implementation; the exact formula is Not given in the sources.
    effective_extinction = base_extinction * ((1.0 - falloff_value) ** (bounce_depth - 1))
    
    return effective_extinction
```

## Subtopics

### Anisotropy Falloff
A companion shortcut that lowers the anisotropy parameter with every bounce to mimic van de Hulst’s similarity principle, making deeper rays scatter more isotropically. See [Anisotropy Falloff](4021-anisotropy-falloff.md).

## Operating characteristics
*   **Time Complexity:** Reduces rendering time by allowing the path tracer to terminate earlier (lower trace depth) while maintaining visual fidelity [S1, unofficial][S2].
*   **Memory Complexity:** No additional memory overhead; it modifies a scalar parameter during ray marching.
*   **Measured Performance:** In Redshift's cloud test, using the default value of 0.2 produced a brighter, more realistic result while rendering faster than the same scene without it [S1, unofficial][S2].
*   **Strengths:** Efficiently approximates multiple scattering effects without the computational cost of tracing hundreds of bounces.
*   **Limitations:** It is an approximation. It does not physically simulate the exact radiative transfer of multiple bounces but mimics their visual result.
*   **When to Use:** When rendering participating media with significant multiple scattering (e.g., clouds, smoke, fire) and performance is a concern.
*   **When Not to Use:** When physical accuracy is required over speed, or when the scene does not benefit from multiple scattering approximation.

## Background
Extinction Falloff is a feature introduced in Redshift Renderer, developed by Maxon. It is part of a suite of shortcuts (including Anisotropy Falloff) designed for experienced artists to achieve realistic volumetric results without excessive trace depths [S1, unofficial][S2]. The technique is inspired by radiative transfer principles, specifically van de Hulst’s similarity principle, which states that in highly multiple scattering media, different combinations of scattering albedo and anisotropy produce nearly indistinguishable appearances [S1, unofficial][S2].

## Sources
[S1] What Is Volumetric Rendering? Volume Shaders Explained, from ..., https://www.maxon.net/en/article/volumetric-rendering, fetched 2026-10-26 (unofficial)
[S2] Introduction to Volumetric Rendering, https://help.maxon.net/c4d/2027/en-us/Content/html/What_is_Volumetric_Rendering.html, fetched 2026-10-29
