# Heterogeneous Medium Transmittance
Also called: Transmittance Computation, computeTransmittance

## Overview
Computes the transmittance between two points through a heterogeneous participating medium. It solves the problem of determining the fraction of light that survives extinction along a ray segment in non-uniform media. It differs from its parent, [Discrete Volume Transmittance Integration](3905-discrete-volume-transmittance-integration.md), by explicitly handling the vector geometry and normalization required for heterogeneous sampling, whereas the parent focuses on the discrete summation and exponentiation logic.

## How it works
**Inputs and Outputs:**
- Input: Start point $p_1$, end point $p_2$, scattering coefficient $\mu_s$, absorption coefficient $\mu_a$.
- Output: Transmittance $T$ (scalar).

**Data Structures:**
- Points $p_1, p_2$: 3D vectors.
- Coefficients $\mu_s, \mu_a$: Scalars.

**Steps:**
1. Compute the direction vector: $\vec{dir} = p_2 - p_1$.
2. Compute the distance: $dist = \|\vec{dir}\|$.
3. Normalize the direction: $\hat{dir} = \vec{dir} / dist$.
4. Compute the extinction coefficient: $\mu_t = \mu_s + \mu_a$.
5. Compute the optical depth: $\tau = \mu_t \times dist$.
6. Compute transmittance: $T = e^{-\tau}$.

**Formulas:**
$$ \vec{dir} = p_2 - p_1 $$
$$ dist = \|\vec{dir}\| $$
$$ \hat{dir} = \frac{\vec{dir}}{dist} $$
$$ \mu_t = \mu_s + \mu_a $$
$$ T = \exp(-\mu_t \cdot dist) $$

**Parameters:**
- $\mu_s$: Scattering coefficient.
- $\mu_a$: Absorption coefficient.
- $p_1, p_2$: Endpoints of the ray segment.

**Edge Cases:**
- If $dist = 0$, transmittance is 1.0.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_transmittance(p1, p2, mu_s, mu_a):
    # Step 1: Compute direction
    dir_vec = p2 - p1
    
    # Step 2: Compute distance
    dist = dir_vec.length()
    
    # Edge case: Zero length
    if dist == 0:
        return 1.0
    
    # Step 3: Normalize direction
    dir_hat = dir_vec / dist
    
    # Step 4: Compute extinction coefficient
    mu_t = mu_s + mu_a
    
    # Step 5 & 6: Compute transmittance
    # Note: The source code snippet provided [S1, unofficial] does not show the 
    # actual transmittance calculation, only matrix operations. 
    # The formula is derived from standard volume rendering theory.
    transmittance = exp(-mu_t * dist)
    
    return transmittance
```

## Subtopics

### [Transmittance Mapping](7461-transmittance-mapping.md)
A differentiable method for approximating volume transmittance using bounds derived from the theory of moments. It solves the problem of computing visibility in inverse rendering tasks where gradients with respect to scene geometry are required. Unlike shadow mapping variants, it provides a principled, volume-based approximation that correctly reports constant 1 transmittance in free space. It builds on [Heterogeneous Medium Transmittance](4909-heterogeneous-medium-transmittance.md) by providing a differentiable, moment-based approximation of the transmittance integral.

## Operating characteristics
Not given in the sources.

## Background
Origin: [S1, unofficial] ainagutierrez01/Volumetric-Path-Tracing- (GitHub repository). The method is part of a volumetric path tracing implementation.

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
