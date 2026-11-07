# Out-Scattering Model

Also called: Out-Scattering, Scattering Loss

## Overview
Out-scattering is the phenomenon where photons are scattered away from their incident direction, preventing them from reaching the observer's eye [S1, unofficial]. It serves as a primary component of extinction, modifying the standard transmittance equation to account for light loss due to scattering rather than just absorption [S1, unofficial]. It is distinct from in-scattering, which describes light entering the viewing direction from other angles [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input**: Position $\mathbf{x}$, direction $\omega$, scattering cross-section $\sigma(\mathbf{x}, \omega)$, true absorption coefficient $\kappa(\mathbf{x})$.
*   **Output**: Total extinction coefficient $\chi(\mathbf{x})$, albedo $W(\mathbf{x})$.

**Data Structures**
No specific data structures are required beyond scalar fields for $\kappa$, $\sigma$, and derived $\chi$.

**Definitions and Formulas**
1.  **Scattering Cross-Section** $\sigma(\mathbf{x}, \omega)$: Defined as the probability of a photon being scattered out of direction $\omega$ per unit length [S2, unofficial].
2.  **Total Extinction** $\chi(\mathbf{x})$: The sum of true absorption and out-scattering [S2, unofficial].
    $$ \chi(\mathbf{x}) = \kappa(\mathbf{x}) + \sigma(\mathbf{x}) $$
    Where $\kappa$ is the true absorption coefficient [S2, unofficial].
3.  **Albedo** $W(\mathbf{x})$: A measure of reflectivity or the ability to scatter relative to total extinction [S2, unofficial].
    $$ W(\mathbf{x}) = \frac{\sigma(\mathbf{x})}{\chi(\mathbf{x})} = \frac{\sigma(\mathbf{x})}{\kappa(\mathbf{x}) + \sigma(\mathbf{x})} $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_out_scattering_params(kappa, sigma):
    """
    Computes total extinction and albedo from true absorption and scattering cross-section.
    
    Args:
        kappa (float): True absorption coefficient.
        sigma (float): Scattering cross-section (probability of scattering out per unit length).
        
    Returns:
        tuple: (chi, W) where chi is total extinction and W is albedo.
    """
    # Total absorption (extinction): true absorption plus out-scattering
    chi = kappa + sigma
    
    # Albedo: measure for reflectivity or ability to scatter
    # Handle division by zero if both kappa and sigma are zero
    if chi == 0:
        W = 0.0
    else:
        W = sigma / chi
        
    return chi, W
```

**Edge Cases and Pitfalls**
*   **Zero Coefficients**: If both $\kappa$ and $\sigma$ are zero, $\chi$ is zero. The albedo formula involves division by $\chi$. Implementation must guard against division by zero [G].
*   **Directionality**: $\sigma$ is defined as $\sigma(\mathbf{x}, \omega)$, implying it may depend on direction, though often simplified to $\sigma(\mathbf{x})$ in isotropic media [S2, unofficial].

## Subtopics

### Anisotropic Out-Scattering
*   **What it is**: Out-scattering where the probability of scattering out depends on the incident direction relative to the outgoing direction.
*   **Difference**: Requires a phase function to model the directional distribution of out-scattering events, unlike isotropic models [S1, unofficial].
*   **When to choose**: When high-fidelity rendering of media like fog or smoke with directional light sources is required.
*   **Link**: [Phase Function](52-phase-function.md)

### Isotropic Out-Scattering
*   **What it is**: Out-scattering where the scattering probability is uniform in all directions.
*   **Difference**: Simplifies the phase function to a constant, reducing computational cost [S1, unofficial].
*   **When to choose**: For general-purpose volume rendering where performance is critical and directional scattering details are less important.
*   **Link**: [Phase Function](52-phase-function.md)

## Operating characteristics
*   **Time Complexity**: $O(1)$ per sample for computing $\chi$ and $W$ given $\kappa$ and $\sigma$.
*   **Memory Complexity**: $O(1)$ additional storage for derived coefficients.
*   **Strengths**: Simple to implement; fundamental to correct light transport in participating media.
*   **Limitations**: Does not account for the directionality of scattered light (handled by in-scattering/phase functions).
*   **When to use**: Always required in participating media rendering to correctly attenuate light along the ray.
*   **When not to use**: Not applicable in vacuum or non-participating media.

## Background
The concept of out-scattering as a loss mechanism in transmittance is a fundamental principle in volumetric rendering, often introduced alongside the Beer-Lambert law [S1, unofficial]. The specific terminology and formulas for scattering cross-section, extinction, and albedo are documented in academic lecture notes on volume rendering [S2, unofficial].

## Sources
[S1] Volumetric rendering fundamental concepts and terminology, https://computergraphics.stackexchange.com/questions/145/volumetric-rendering-fundamental-concepts-and-terminology, fetched 2026-10-27 (unofficial)
[S2] Volume Rendering Volume Rendering, https://docslib.org/doc/1400821/volume-rendering, fetched 2026-10-28 (unofficial)
