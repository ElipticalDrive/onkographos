# Constant Distance Sampling

Also called: Blasi et al, Blasi et al. constant distance sampling

## Overview
Constant Distance Sampling is a Monte Carlo light tracing technique for participating media that advances light bundles in discrete steps of fixed length $\delta$ [S1]. It differs from free-path sampling by decoupling the interaction distance from the exponential distribution, instead using a uniform grid-like progression to simplify the handling of multiple scattering and phase function approximation [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A light bundle (ray) with current position, direction, and flux; medium properties (absorption coefficient $\sigma_a$, scattering coefficient $\sigma_s$, extinction coefficient $\sigma_t = \sigma_a + \sigma_s$); step size $\delta$; Schlick phase function parameters.
*   **Output:** Updated bundle position, direction, and flux; scattered energy stored in voxels.

**Data Structures**
*   **Voxel Grid:** Stores the isotropic component of scattered light. Each voxel holds a scalar value representing the isotropic energy density [S1].
*   **Light Bundle:** Contains position $\mathbf{x}$, direction $\mathbf{d}$, flux $L$, and remaining path length or termination status.

**Method Steps**
1.  **Initialization:** Start a light bundle from a light source or surface interaction. Set initial position $\mathbf{x}_0$ and direction $\mathbf{d}$.
2.  **Step Advancement:** Advance the bundle by a constant distance $\delta$. The new position is $\mathbf{x}_{new} = \mathbf{x}_{old} + \delta \mathbf{d}$ [S1].
3.  **Absorption:** Decrease the bundle's flux $L$ by the transmittance due to absorption over the distance $\delta$. The transmittance $T$ is calculated as:
    $$ T = e^{-\sigma_a \delta} $$
    Update flux: $L_{new} = L_{old} \cdot T$ [S1].
4.  **Scattering Decision:** At the new position, perform a sampling process to decide if scattering occurs. This involves comparing a random number to the scattering probability derived from the medium's properties over the step $\delta$ [S1].
5.  **Scattering Event:** If scattering occurs:
    *   Determine the scatter direction using importance sampling of the Schlick phase function [S1].
    *   The Schlick phase function approximates Mie scattering as a combination of isotropic and forward scattering components [S1].
    *   Extract the isotropic component of the scattered light and store it in the current voxel [S1].
    *   Update the bundle's direction to the sampled scatter direction.
6.  **Termination:** If the bundle exits the medium or its flux falls below a threshold, terminate the bundle. If it exits the medium, record its energy at the border voxel [S1].

**Parameters**
*   $\delta$: Constant step length. Value not given in the sources.
*   $\sigma_a, \sigma_s$: Absorption and scattering coefficients.

**Edge Cases and Pitfalls**
*   **Isotropic Energy Loss:** The isotropic part of the scattered light stored in a voxel is not considered for the illumination of other voxels. This introduces an error, but it is expected to be small because the directional component is much more important [S1]. A progressive refinement strategy can be used if isotropic energy becomes significant [S1].
*   **Mixed Scenes:** In mixed scenes (surfaces and media), bundles hitting diffuse surfaces store energy and terminate, while those hitting specular surfaces reflect using importance sampling [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of steps, which is proportional to the path length divided by $\delta$.
*   **Memory Complexity:** Depends on the voxel grid size for storing isotropic energy.
*   **Strengths:** Simplifies the handling of multiple scattering; inexpensive importance sampling for the Schlick phase function; suitable for mixed scenes with progressive refinement [S1].
*   **Limitations:** Introduces bias due to the fixed step size and the neglect of isotropic energy in voxel-to-voxel illumination; requires careful choice of $\delta$ to balance accuracy and performance [S1].

## Background
Constant Distance Sampling was introduced by Blasi et al. in [9] for single participating media and extended in [10] for mixed scenes [S1]. It is based on a simulation of the particle model of light using Monte Carlo light tracing [S1]. The method uses the Schlick phase function to approximate Mie scattering, combining isotropic and forward scattering components for efficient importance sampling [S1].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
