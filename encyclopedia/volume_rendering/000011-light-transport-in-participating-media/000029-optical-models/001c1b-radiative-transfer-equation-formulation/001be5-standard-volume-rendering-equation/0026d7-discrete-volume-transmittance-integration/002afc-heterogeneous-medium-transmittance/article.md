# Heterogeneous Medium Transmittance
Also called: Transmittance Computation, computeTransmittance

## Overview
Computes transmittance between two points through a heterogeneous participating medium [S1, unofficial]. It provides the geometric setup required before applying the discrete extinction summation and exponentiation logic of its parent, [Discrete Volume Transmittance Integration](3905-discrete-volume-transmittance-integration.md) [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Start point $p_1$, end point $p_2$, scattering coefficient $\mu_s$, absorption coefficient $\mu_a$ [S1, unofficial].
- Output: Transmittance $T$ (scalar).

**Data Structures:**
- Points $p_1, p_2$: 3D vectors.
- Coefficients $\mu_s, \mu_a$: Scalars.

**Steps:**
1. Compute the direction vector: $\vec{dir} = p_2 - p_1$ [S1, unofficial].
2. Compute the distance: $dist = \|\vec{dir}\|$ [S1, unofficial].
3. Normalize the direction: $\hat{dir} = \vec{dir} / dist$ [S1, unofficial].
4. Compute the extinction coefficient: $\mu_t = \mu_s + \mu_a$ [G].
5. Compute the optical depth: $\tau = \mu_t \times dist$ [G].
6. Compute transmittance: $T = e^{-\tau}$ [G].

**Formulas:**
$$ \vec{dir} = p_2 - p_1 $$
$$ dist = \|\vec{dir}\| $$
$$ \hat{dir} = \frac{\vec{dir}}{dist} $$
$$ \mu_t = \mu_s + \mu_a $$
$$ T = \exp(-\mu_t \cdot dist) $$

**Parameters:**
- $\mu_s$: Scattering coefficient.
- $\mu_a$: Absorption coefficient.
- $p_1, p_2$: Endpoints of the ray segment [S1, unofficial].

**Edge Cases:**
- If $dist = 0$, transmittance is $1.0$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_transmittance(p1, p2, mu_s, mu_a):
    dir_vec = p2 - p1
    dist = dir_vec.length()
    if dist == 0:
        return 1.0
    dir_hat = dir_vec / dist
    mu_t = mu_s + mu_a
    transmittance = math.exp(-mu_t * dist)
    return transmittance
```

## Subtopics

### [Transmittance Mapping](7461-transmittance-mapping.md)
A differentiable method for approximating volume transmittance using bounds derived from the theory of moments. It solves the problem of computing visibility in inverse rendering tasks where gradients with respect to scene geometry are required. Unlike shadow mapping variants, it provides a principled, volume-based approximation that correctly reports constant 1 transmittance in free space. It builds on [Heterogeneous Medium Transmittance](4909-heterogeneous-medium-transmittance.md) by providing a differentiable, moment-based approximation of the transmittance integral.

### Aggregated Interior Visibility (AIV)
A global visibility function $\hat{V}(\omega)$ representing the average visibility from points on the surfaces inside a voxel through the entire scene along direction $\omega$, used to model spatial correlation between voxels [S2, unofficial]. It accumulates voxel contributions using the product of their individual AIV values to correctly handle positive and negative correlations between voxels, which traditional Beer-Lambert law fails to do [S2, unofficial]. Traditional volumetric models assume transmittance is exponential and cannot model spatial correlation, producing incorrect results even if per-voxel visibilities are identical [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S2] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
