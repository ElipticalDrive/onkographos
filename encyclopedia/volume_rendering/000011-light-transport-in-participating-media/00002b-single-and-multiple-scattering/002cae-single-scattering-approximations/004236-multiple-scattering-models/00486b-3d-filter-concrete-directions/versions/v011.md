# 3D-Filter / Concrete Directions

Also called: 3D-Filter / Nishita, Nishita et al. 3D-Filter, 3D-Filter / N Directions, Concrete Directions Method

## Overview
The 3D-Filter method approximates multiple scattering in participating media by representing the directional distribution of radiance using extended form factors within a voxel grid [S1][S2, book]. It solves the problem of computationally prohibitive multiple scattering integration by precomputing a "contribution-ratio pattern" (the 3D-filter) that encodes how energy from neighboring voxels contributes to the radiance of a center voxel in the viewing direction [S2, book]. Unlike sibling methods that may use diffusion equations or stochastic path tracing for the full angular distribution, this approach restricts the solution to specific scattering orders (typically up to third order) and exploits the forward-scattering nature of media like clouds to accelerate the calculation of these form factors [S1][S2, book].

## How it works
### Inputs and Outputs
**Inputs:**
*   A voxelized representation of the participating medium (e.g., a cloud or snow volume) [S1][S2, book].
*   The viewing direction $\omega_v$ [S2, book].
*   The sunlight direction $\omega_s$ [S2, book].
*   The mean density of the cloud $\bar{\rho}$ [S2, book].
*   Single scattering radiance contribution $L_{ss}$ [S2, book].

**Outputs:**
*   Total radiance $L(x)$ at each voxel $x$ in the viewing direction [S1][S2, book].

### Data Structures
The method relies on a 3D grid of voxels. The space including the medium is subdivided into voxels, with the viewing direction defined as the principal axis of the grid [S2, book].

1.  **Voxel Grid:** Stores the density and optical properties of the medium.
2.  **3D-Filter (Contribution-Ratio Pattern):** A precomputed data structure representing the extended form factors. It stores the contribution ratios to the radiance of a center voxel from other voxels, taking into account the sunlight direction [S2, book].
    *   *Layout:* For a center voxel, the filter contains weights corresponding to neighboring voxels that effectively contribute to the radiance in the viewing direction [S2, book].
    *   *Content:* The filter stores the light scattered due to second and third-order scattering in the viewing direction [S2, book].

### Steps
1.  **Voxelization:** Subdivide the space including the medium into voxels, aligning the viewing direction as the principal axis [S2, book].
2.  **Define Reference Space:** Set up a smaller reference space characterized by the mean density of the cloud $\bar{\rho}$ [S2, book]. This simplifies the computation of form factors by assuming uniform density for the filter generation [S2, book].
3.  **Compute 3D-Filter (Precomputation):**
    *   Calculate the contribution ratios to the radiance of a center voxel (in the viewing direction) from other voxels in the reference space, considering the sunlight direction [S2, book].
    *   This calculation constitutes the "contribution-ratio pattern" or 3D-filter [S2, book].
    *   *Optimization:* Since scattering in media like clouds is mainly forward, energy scattered at a point lies within a small solid angle. Use this fact to compute extended form factors faster by concentrating effort on voxels that effectively contribute to the center voxel for paths with one or two scatterings [S2, book].
    *   *Selection:* Use a stochastic method to select the voxels that effectively contribute to the center voxel [S1][S2, book].
    *   *Orders:* Consider only the first three orders of scattering for multiple scattering of sunlight to save computation time [S2, book]. Second-order scattering is explicitly handled in the filter [S1].
4.  **Apply Filter to Medium:**
    *   Apply the precomputed 3D-filter to the voxels in the whole space [S2, book].
    *   For each voxel, multiply the form factors (from the filter) by the energy at the related points (neighboring voxels) [S1].
    *   Accumulate these results to obtain the multiple scattering radiance component $L_{ms}(x)$ [S1].
    *   Store the light scattered due to second and third-order scattering in the viewing direction for each voxel [S2, book].
5.  **Compute Single Scattering:** Compute the single scattering contribution $L_{ss}(x)$ separately [S2, book].
6.  **Accumulate Radiance:** Combine the single scattering and multiple scattering components to obtain the final radiance $L(x)$ [S1][S2, book].
    $$ L(x) = L_{ss}(x) + L_{ms}(x) $$

### Formulas and Definitions
*   **Radiance:** $L(x)$ is the total radiance at position $x$ in the viewing direction [S1].
*   **Multiple Scattering Orders:** The method considers only the first three orders of scattering for sunlight [S2, book].
    *   Order 1: Single scattering (computed separately) [S2, book].
    *   Order 2 & 3: Handled by the 3D-filter [S2, book].
*   **Contribution Ratio:** The weight $w_{ij}$ in the 3D-filter represents the ratio of radiance contribution from voxel $j$ to voxel $i$ in the viewing direction, given the sunlight direction [S2, book].
*   **Energy Accumulation:**
    $$ L_{ms}(x_i) = \sum_{j} w_{ij} \cdot E_j $$
    where $E_j$ is the energy at voxel $j$ [S1].

### Parameters
*   **Scattering Orders:** Typically limited to the first three orders (single, second, third) to save computation time [S2, book].
*   **Mean Density:** The reference space uses the mean density of the cloud $\bar{\rho}$ [S2, book].
*   **Stochastic Selection:** A stochastic method is used to select contributing voxels [S1][S2, book]. The specific number of directions or samples is not detailed in the sources, but the method relies on selecting voxels that effectively contribute [S1][S2, book].

### Edge Cases and Pitfalls
*   **Forward Scattering Assumption:** The method relies on the fact that scattering is mainly forward (e.g., in clouds) to optimize the form factor calculation [S2, book]. This may be less accurate for media with strong backward scattering.
*   **Mean Density Approximation:** Using a smaller space with mean density for the filter may introduce errors if the density distribution is highly heterogeneous [S2, book].
*   **Limited Scattering Orders:** By considering only the first three orders, the method may underestimate radiance in very thick or highly scattering media where higher-order scattering is significant [S2, book].

## Operating characteristics
*   **Time Complexity:** The precomputation of the 3D-filter is optimized by focusing on contributing voxels and using stochastic selection, reducing the cost compared to computing form factors between all voxel pairs [S2, book]. The application of the filter is a convolution-like operation over the voxel grid.
*   **Memory Complexity:** Requires storage for the voxel grid and the 3D-filter data structure [S1][S2, book].
*   **Strengths:** Efficiently handles multiple scattering in media like clouds and snow by exploiting forward scattering and limiting scattering orders [S1][S2, book].
*   **Limitations:** Approximate nature due to limited scattering orders and mean density assumption; less suitable for media with non-forward scattering characteristics [S2, book].
*   **When to Use:** Suitable for rendering clouds and snow where multiple scattering is significant and forward scattering dominates [S1][S2, book].

## Background
The 3D-Filter method was proposed by Nishita et al. [S1][S2, book]. It was originally developed to display clouds taking into account multiple scattering and sky light [S2, book]. The method computes radiance from a cloud reaching the eye as sunlight multiple scattered plus skylight single scattered by the particles of the cloud [S2, book]. It has also been applied to render snow, which has a high albedo and looks white due to multiple scattering, with shape and density distribution modeled using metaballs [S1]. The approach is part of a broader family of methods that use implicit representations of directional distribution, such as scattering patterns and diffusion equations [S1].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] Global Illumination Techniques for the Simulation of Participating Media, https://inria.hal.science/inria-00510096v1, fetched 2026-10-26 (book)
