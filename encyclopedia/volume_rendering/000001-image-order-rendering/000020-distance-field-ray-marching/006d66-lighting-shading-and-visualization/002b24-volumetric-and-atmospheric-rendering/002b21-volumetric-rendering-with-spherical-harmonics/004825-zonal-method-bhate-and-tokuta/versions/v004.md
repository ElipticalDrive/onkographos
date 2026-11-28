# Zonal Method (Bhate and Tokuta)
Also called: Bhate et al. Method, Bhate's method

## Overview
The Zonal Method (Bhate and Tokuta) is a global illumination technique for participating media that extends the classical zonal method to handle interactions between volumes and ideal diffuse surfaces while eliminating the assumption of medium isotropy [S1][S2, book]. It distinguishes itself from standard zonal methods by representing the phase function and radiance using spherical harmonics to approximate Mie scattering, and by computing the resulting form factors using an extended hemicube technique [S1][S2, book].

## How it works
**Inputs:** A scene containing $v$ volume elements (voxels) and $s$ ideal diffuse surfaces [S1][S2, book].
**Outputs:** The radiance distribution within the volumes and reflected radiance from the surfaces [S1][S2, book].

**Data Structures:**
1. Spherical harmonic coefficient vectors of size $M$ for each volume and surface to store radiance expansions [S1][S2, book].
2. Form factor matrices $F_{ik}$ and $F_{is_j}$ for volume-volume and volume-surface interactions, computed per spherical harmonic term [S1][S2, book].

**Steps:**
1. Approximate the scattering phase function by the first $M$ terms of its spherical harmonics expansion to approximate Mie scattering [S1][S2, book].
2. Compute the form factors between all volume elements and between volumes and surfaces, accounting for the spherical harmonics, using the extended hemicube technique [S1][S2, book].
3. Set up a system of linear equations consisting of $M \times v$ equations for the $v$ volumes plus $s$ related equations for the $s$ surfaces [S1][S2, book].
4. Solve the resulting system using a Gauss-Seidel iterative technique [S1][S2, book].

**Formulas:** Not given in the sources [S1][S2, book].

**Parameters:** $M$ is the number of spherical harmonic terms; only the first few are necessary for a convincing image, but computational cost scales significantly with $M$ [S1][S2, book].

**Edge Cases and Pitfalls:** The direct application of this method is impractical for large scenes due to the prohibitive cost of computing form factors [S1][S2, book].

## Subtopics

### [Progressive Refinement Zonal Method](12300-progressive-refinement-zonal-method.md)
A hierarchical acceleration technique for the Zonal Method that reduces form factor computation cost by dynamically determining interaction levels and skipping unnecessary refinements. It is chosen for complex scenes where direct computation is prohibitive, distinguishing itself from standard zonal methods by avoiding iterative hierarchy refinement.

## Operating characteristics
* **Time Complexity:** The cost to compute form factors in a regular grid of $v = n^3$ voxels is $O(n^7 + M^2 n^6)$ [S1][S2, book].
* **Memory Complexity:** Not given in the sources [S1][S2, book].
* **Strengths:** Handles anisotropic scattering (Mie approximation) and surface-volume interactions, which standard zonal methods cannot [S1][S2, book].
* **Limitations:** Extremely high computational cost for form factor calculation makes it impractical for large-scale scenes without acceleration techniques [S1][S2, book].
* **When to use:** Not given in the sources [S1][S2, book].
* **When not to use:** Not given in the sources [S1][S2, book].

## Background
The method was proposed by Bhate and Tokuta as an extension to the classical zonal method [S1][S2, book]. It addresses the limitations of previous zonal method implementations which assumed isotropic media and ignored effects between surfaces and volumes [S1][S2, book].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] Global Illumination Techniques for the Simulation of Participating Media, https://inria.hal.science/inria-00510096v1, fetched 2026-10-26 (book)
