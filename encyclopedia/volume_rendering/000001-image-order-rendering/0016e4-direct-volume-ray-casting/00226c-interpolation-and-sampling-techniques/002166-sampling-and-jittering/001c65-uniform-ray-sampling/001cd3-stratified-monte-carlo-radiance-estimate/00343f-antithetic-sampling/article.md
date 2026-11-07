# Antithetic Sampling

Also called: Antithetic BSDF Sampling

## Overview
Antithetic sampling is a variance reduction technique for estimating the derivative of a BSDF with respect to scene parameters, specifically when the integrand approximates an odd function [S1, unofficial]. It exploits the symmetry of Normal Distribution Functions (NDFs) to make the derivative term an odd function, allowing paired samples to cancel out noise [S1, unofficial]. It sets itself apart from general stratified sampling by targeting the specific symmetry of microfacet derivatives, whereas its sibling [Decomposed sampling](1374-decomposed-sampling.md) addresses cases where derivatives are even functions [S1, unofficial].

## How it works
**Inputs:**
*   Outgoing direction $\boldsymbol{\omega}_o$.
*   Scene parameter $\pi$ (e.g., roughness, position).
*   NDF model $D$ (assumed symmetric, e.g., Beckmann, GGX) [S1, unofficial].

**Outputs:**
*   Two antithetic samples for the derivative estimate [S1, unofficial].

**Data Structures:**
*   Local coordinate system defined by the surface normal $\mathbf{n}$ and tangent $\mathbf{t}$ [S1, unofficial].
*   Half-way vectors $\boldsymbol{\omega}_{h,1}, \boldsymbol{\omega}_{h,2}$ [S1, unofficial].

**Steps:**
1.  **Draw first half-way vector:** Sample a half-way vector $\boldsymbol{\omega}_{h,1}$ based on the NDF $D$ [S1, unofficial].
2.  **Generate antithetic half-way vector:** Set $\boldsymbol{\omega}_{h,2}$ to be symmetric to $\boldsymbol{\omega}_{h,1}$ with respect to the z-axis in the local coordinate system [S1, unofficial].
3.  **Compute incident directions:** Calculate the corresponding incident directions $\boldsymbol{\omega}_{i,1}$ and $\boldsymbol{\omega}_{i,2}$ using the reflection relation $\boldsymbol{\omega}_{h} = (\boldsymbol{\omega}_i + \boldsymbol{\omega}_o) / \|\boldsymbol{\omega}_i + \boldsymbol{\omega}_o\|$ [S1, unofficial].
4.  **Evaluate derivative:** Compute the BSDF derivative $\frac{\partial f_s^{(0)}}{\partial \pi}$ for both samples [S1, unofficial].

**Formulas:**
The microfacet BSDF is decomposed as [S1, unofficial]:
$$
f_s^{(0)}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = D(\boldsymbol{\omega}_h) f_s^{(0)}(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o)
$$
where $f_s^{(0)}$ represents other terms (e.g., Fresnel) and $\boldsymbol{\omega}_h := (\boldsymbol{\omega}_i + \boldsymbol{\omega}_o) / \|\boldsymbol{\omega}_i + \boldsymbol{\omega}_o\|$ [S1, unofficial].

The derivative with respect to parameter $\pi$ is [S1, unofficial]:
$$
\frac{\partial f_s^{(0)}}{\partial \pi} = \frac{\partial D}{\partial \boldsymbol{\omega}_h} \frac{\partial \boldsymbol{\omega}_h}{\partial \pi} f_s^{(0)} + D \frac{\partial f_s^{(0)}}{\partial \pi}
$$
The first term dominates for glossy BSDFs. If $D$ is symmetric, $\frac{\partial D}{\partial \boldsymbol{\omega}_h}$ is an odd function [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def antithetic_bsdf_sample(outgoing_direction, parameter, ndf_model, local_basis):
    """
    Generates two antithetic samples for BSDF derivative estimation.
    
    Args:
        outgoing_direction: Vector omega_o.
        parameter: Scene parameter pi (e.g., roughness).
        ndf_model: Function D(omega_h) returning NDF value.
        local_basis: Tuple (normal, tangent, bitangent) defining local coords.
        
    Returns:
        List of two tuples (incident_direction, derivative_estimate).
    """
    # Step 1: Draw first half-way vector based on NDF
    # Note: Specific sampling method for NDF is implementation-dependent.
    # We assume a function sample_ndf(ndf_model, local_basis) exists.
    omega_h1 = sample_ndf(ndf_model, local_basis)
    
    # Step 2: Set second half-way vector symmetric to omega_h1 w.r.t z-axis
    # In local coordinates, z-axis is the normal. Symmetry w.r.t z-axis
    # implies flipping the x and y components (tangent and bitangent)
    # while keeping the z component (normal) unchanged.
    omega_h2 = flip_xy_in_local_basis(omega_h1, local_basis)
    
    # Step 3: Compute incident directions
    # omega_h = (omega_i + omega_o) / ||omega_i + omega_o||
    # Therefore, omega_i = 2 * (omega_h . omega_o) * omega_h - omega_o
    # This assumes perfect reflection geometry for the half-way vector definition.
    omega_i1 = reflect_across_halfway(omega_h1, outgoing_direction)
    omega_i2 = reflect_across_halfway(omega_h2, outgoing_direction)
    
    # Step 4: Evaluate derivative for both samples
    # The derivative calculation uses the formula:
    # dF/dpi = dD/dwh * dwh/dpi * F_other + D * dF_other/dpi
    # We assume a function compute_bsdf_derivative exists that handles
    # the specific BSDF model (Fresnel, Geometry, etc.)
    deriv1 = compute_bsdf_derivative(omega_i1, outgoing_direction, parameter, ndf_model, local_basis)
    deriv2 = compute_bsdf_derivative(omega_i2, outgoing_direction, parameter, ndf_model, local_basis)
    
    return [(omega_i1, deriv1), (omega_i2, deriv2)]

def flip_xy_in_local_basis(vector, local_basis):
    """
    Flips the components of a vector along the tangent and bitangent axes
    of the local basis, keeping the normal component unchanged.
    """
    normal, tangent, bitangent = local_basis
    # Project vector onto local basis
    z = dot(vector, normal)
    x = dot(vector, tangent)
    y = dot(vector, bitangent)
    # Flip x and y
    return normal * z - tangent * x - bitangent * y

def reflect_across_halfway(halfway, outgoing):
    """
    Computes the incident direction omega_i given omega_o and omega_h
    assuming omega_h is the normalized sum.
    omega_i = 2 * dot(omega_h, omega_o) * omega_h - omega_o
    """
    return 2.0 * dot(halfway, outgoing) * halfway - outgoing
```

**Edge Cases and Pitfalls:**
*   **Exponential Path Growth:** Applying antithetic sampling to every light bounce on glossy surfaces in scenes with multiple glossy surfaces causes exponential growth in the number of light paths [S1, unofficial].
*   **Even Functions:** Antithetic sampling is not effective if the BSDF derivative is an even function (e.g., derivative w.r.t. roughness of GGX), as it produces high variance in such cases [S1, unofficial].

## Subtopics

### [Decomposed sampling](1374-decomposed-sampling.md)
A technique for cases where BSDF derivatives are even functions, where antithetic sampling fails.

### [Stratified Monte Carlo Radiance Estimate](1373-stratified-monte-carlo-radiance-estimate.md)
The parent method that provides the stratified framework within which antithetic sampling is applied to reduce variance.

## Operating characteristics
*   **Variance Reduction:** Significantly lowers variance when estimating the interior integral for glossy BSDFs where the NDF is symmetric [S1, unofficial].
*   **Complexity:** Doubles the number of samples per bounce if applied naively, leading to exponential path counts in complex scenes with multiple glossy surfaces [S1, unofficial].
*   **Limitations:** Ineffective for even-function derivatives (e.g., GGX roughness derivative) [S1, unofficial].
*   **Usage Strategy:** For complex scenes, apply antithetic sampling only once per light path, merge branched paths after a few bounces (when the surface is no longer glossy), and repeat the path $k$ times to sample $k$ different glossy surfaces [S1, unofficial].

## Background
Antithetic sampling for BSDF derivatives was formally discussed by Zhang et al. [ZDDZ21], building on earlier mentions in [LHJ19] and [ZSGJ21] [S1, unofficial]. It is a key technique in differentiable rendering to stabilize gradients for glossy materials [S1, unofficial].

## Sources
[S1] A Survey on Physics-based Differentiable Rendering, https://arxiv.org/pdf/2504.01402, fetched 2026-10-29 (unofficial)
