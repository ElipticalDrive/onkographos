# Jendersie-d’Eon Phase Function
Also called: Jendersie-d'Eon
## Overview
The Jendersie-d’Eon phase function is a model that approximates actual Mie scattering by water droplets, providing a more realistic forward scattering peak than the Henyey-Greenstein phase function [S1][S2, unofficial]. It is specifically used in Redshift for cloud rendering to model the optical properties of water droplets [S2, unofficial].

## How it works
The Jendersie-d’Eon model replaces the single-anisotropy approximation of Henyey-Greenstein with a formulation derived from Mie theory to better capture the sharp forward peak characteristic of light scattering by spherical water droplets [S1][S2, unofficial].

**Inputs and Outputs:**
*   **Input:** Scattering direction $\omega_s$, incident direction $\omega_i$, and phase function parameters (typically derived from droplet size distribution and refractive index, though specific parameter mappings are not given in the sources).
*   **Output:** Phase function value $p(\omega_s, \omega_i)$ representing the probability density of scattering into $\omega_s$.

**Data Structures:**
Not given in the sources.

**Steps:**
1.  Calculate the scattering angle $\theta$ between the incident direction $\omega_i$ and the outgoing direction $\omega_s$ using the dot product: $\cos \theta = \omega_i \cdot \omega_s$.
2.  Evaluate the Jendersie-d’Eon phase function formula $p(\cos \theta)$ using the appropriate Mie-derived parameters for water droplets. The exact mathematical formula is not provided in the cited sources.
3.  Return the resulting phase value.

**Formulas:**
Not given in the sources.

**Parameters:**
The model is controlled by parameters related to the physical properties of water droplets (size, refractive index) rather than a single empirical anisotropy value like $g$ in Henyey-Greenstein [S1][S2, unofficial].

**Edge Cases and Pitfalls:**
Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def jendersie_deon_phase(cos_theta, droplet_params):
    """
    Evaluate the Jendersie-d'Eon phase function.
    
    Args:
        cos_theta (float): Cosine of the scattering angle (omega_i . omega_s).
        droplet_params (dict): Parameters describing water droplet Mie scattering.
        
    Returns:
        float: Phase function value.
    """
    # The exact formula for the Jendersie-d'Eon phase function is not provided in the sources.
    # Implementation requires the specific Mie approximation equations defined in the original literature.
    raise NotImplementedError("Formula not provided in sources.")
```

## Subtopics

### Henyey-Greenstein Phase Function
The parent model using a single anisotropy parameter $g$. Choose when computational simplicity is prioritized over physical accuracy for water droplets. [Henyey-Greenstein Phase Function](1010-henyey-greenstein-phase-function.md)

### Cornette and Shanks Phase Function
A sibling model for particulate scattering. Choose for different types of particulate media where Jendersie-d’Eon's water-droplet focus is not applicable. [Cornette and Shanks Phase Function](1710-cornette-and-shanks-phase-function.md)

### Multiple Octaves Phase Function
A sibling model using layered scattering. Choose for complex, multi-scale scattering effects. [Multiple Octaves Phase Function](4939-multiple-octaves-phase-function.md)

### HG-Draine Blend Mie Approximation
A sibling model blending Henyey-Greenstein with Mie theory. Choose when a hybrid approach is preferred. [HG-Draine Blend Mie Approximation](10011-hg-draine-blend-mie-approximation.md)

## Operating characteristics
The Jendersie-d’Eon model provides a more realistic forward scattering peak than Henyey-Greenstein, which is critical for accurately rendering the bright edges of clouds illuminated from behind [S1][S2, unofficial]. It is available in Redshift specifically for cloud rendering [S2, unofficial]. Specific time and memory complexity metrics are not given in the sources.

## Background
The Henyey-Greenstein phase function is a classic model dating back to a 1941 astrophysics paper [S1][S2, unofficial]. The Jendersie-d’Eon model is a newer alternative introduced to better approximate the true Mie scattering of water droplets, addressing the limitations of the single-anisotropy Henyey-Greenstein model for cloud rendering [S1][S2, unofficial].

## Sources
[S1] Introduction to Volumetric Rendering, https://help.maxon.net/c4d/2027/en-us/Content/html/What_is_Volumetric_Rendering.html, fetched 2026-10-29
[S2] ボリュームレンダリングとは？雲から爆発まで, https://www.maxon.net/ja/article/volumetric-rendering?srsltid=AYCX9vK-JecQAKHX24X4ZdtRcP9GsOjvBnGwCqB51LzXFIekQMframwI, fetched 2026-10-30 (unofficial)
