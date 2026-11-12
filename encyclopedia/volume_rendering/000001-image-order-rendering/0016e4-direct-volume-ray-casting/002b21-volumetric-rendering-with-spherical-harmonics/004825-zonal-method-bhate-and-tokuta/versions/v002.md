# Zonal Method (Bhate and Tokuta)
Also called: Bhate et al. Method, Bhate's method

## Overview
The Zonal Method (Bhate and Tokuta) is a global illumination technique for participating media that extends the classical zonal method by incorporating spherical harmonics to handle anisotropic scattering and surface-volume interactions [S1][S2, book]. Unlike the standard zonal method which assumes isotropic media and ignores surface effects, this method represents the phase function and radiance using spherical harmonics, allowing for the approximation of Mie scattering while maintaining ideal diffuse surfaces [S1][S2, book]. It is distinguished from its siblings by its specific use of an extended hemicube technique to compute spherical harmonic form factors and a Gauss-Seidel solver for the coupled volume-surface system [S1][S2, book].

## How it works
**Inputs:** A scene containing $v$ volume elements (voxels) and $s$ ideal diffuse surfaces. The medium's phase function is assumed to be anisotropic (e.g., Mie scattering).
**Outputs:** The radiance distribution within the volumes and reflected radiance from the surfaces.

**Data Structures:**
1.  **Spherical Harmonic Coefficients:** For each volume $i$, a vector of coefficients $\mathbf{L}_i$ of size $M$ representing the radiance expansion. For each surface $j$, a vector of coefficients $\mathbf{R}_j$ of size $M$ representing the reflected radiance.
2.  **Form Factors:** A tensor or set of matrices $F_{ij}^{(m)}$ representing the form factor between element $i$ and element $j$ for the $m$-th spherical harmonic term.

**Steps:**
1.  **Phase Function Approximation:** Approximate the scattering phase function $p(\Omega)$ by the first $M$ terms of its spherical harmonics expansion [S1][S2, book].
    $$ p(\Omega) \approx \sum_{m=0}^{M-1} a_m Y_m(\Omega) $$
    where $Y_m$ are the spherical harmonic basis functions and $a_m$ are the expansion coefficients derived from the phase function [S1][S2, book].

2.  **Form Factor Computation:** Calculate the form factors between all pairs of volume elements and between volumes and surfaces, taking into account the spherical harmonics [S1][S2, book]. This is done using the **extended hemicube technique** [S1][S2, book]. The standard hemicube is extended to compute the coefficients of the spherical harmonic expansion of the geometric visibility/transfer function rather than just scalar form factors [S1][S2, book].

3.  **System Setup:** Set up a system of linear equations. For each volume $i$, the energy balance equation involves the incoming radiance from all other volumes and surfaces, weighted by the computed form factors and the scattering properties [S1][S2, book].
    The system consists of $M \times v$ equations for the volumes and $M \times s$ equations for the surfaces [S1][S2, book].
    Let $\mathbf{L}_i$ be the radiance vector for volume $i$ and $\mathbf{R}_j$ be the radiance vector for surface $j$. The equation for volume $i$ can be expressed as:
    $$ \mathbf{L}_i = \mathbf{E}_i + \sigma_{s,i} \sum_{k=1}^{v} F_{ik} \mathbf{L}_k + \sigma_{s,i} \sum_{j=1}^{s} F_{is_j} \mathbf{R}_j $$
    where $\mathbf{E}_i$ is the emitted radiance, $\sigma_{s,i}$ is the scattering coefficient, $F_{ik}$ is the form factor matrix between volumes $i$ and $k$, and $F_{is_j}$ is the form factor matrix between volume $i$ and surface $j$ [S1][S2, book].
    The equation for surface $j$ relates the reflected radiance to the incident radiance from volumes and other surfaces, assuming ideal diffuse reflection [S1][S2, book].

4.  **System Solution:** Solve the resulting large sparse linear system using a **Gauss-Seidel iterative technique** [S1][S2, book]. The iterations continue until convergence or a fixed number of iterations is reached.

**Parameters:**
*   $M$: The number of spherical harmonic terms. "Only the first few spherical harmonics are necessary for a convincing image" [S1][S2, book]. Typical values are small integers (e.g., 3-5), but the cost scales with $M$ [S1][S2, book].

**Edge Cases and Pitfalls:**
*   **Prohibitive Cost:** The direct application of this method is impractical for large scenes due to the high computational cost of computing form factors [S1][S2, book].
*   **Precision:** The accuracy depends heavily on the choice of $M$. Too few terms result in inaccurate anisotropic scattering; too many terms increase computational cost significantly [S1][S2, book].

## Subtopics

### [Progressive Refinement Zonal Method](12300-progressive-refinement-zonal-method.md)
A hierarchical acceleration technique for the Zonal Method that reduces computational cost by dynamically determining interaction levels and skipping unnecessary refinements.

## Operating characteristics
*   **Time Complexity:** The cost to compute form factors in a regular grid of $v = n^3$ voxels is $O(n^7 + M^2 n^6)$ [S1][S2, book]. This is considered prohibitive for direct application [S1][S2, book].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Handles anisotropic scattering (Mie approximation) and surface-volume interactions, which standard zonal methods cannot [S1][S2, book].
*   **Limitations:** Extremely high computational cost for form factor calculation makes it impractical for large-scale scenes without acceleration techniques like progressive refinement [S1][S2, book].
*   **When to use:** When accurate anisotropic scattering and surface interactions are required and computational resources are sufficient, or as a baseline for accelerated variants.
*   **When not to use:** For real-time rendering or large scenes where the $O(n^7)$ cost is prohibitive.

## Background
The method was proposed by Bhate and Tokuta as an extension to the classical zonal method [S1][S2, book]. It addresses the limitations of previous zonal method implementations which assumed isotropic media and ignored effects between surfaces and volumes [S1][S2, book]. The method is cited in surveys on participating media rendering techniques [S1][S2, book].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] Global Illumination Techniques for the Simulation of Participating Media, https://inria.hal.science/inria-00510096v1, fetched 2026-10-26 (book)
