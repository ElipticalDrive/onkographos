# 3D-Filter / Concrete Directions

Also called: 3D-Filter / Nishita, Nishita et al. 3D-Filter, 3D-Filter / N Directions, Concrete Directions Method

## Overview
The 3D-Filter method is a deterministic approximation for rendering multiple scattering in participating media, specifically designed for clouds and snow [S1][S2, book]. It solves the problem of computationally expensive multiple scattering integrals by precomputing a "contribution-ratio pattern" (the 3D-filter) that encodes the directional distribution of radiance for second and third-order scattering events [S1][S2, book]. Unlike stochastic methods that trace random walks, this approach uses a grid-based implicit representation of directional distribution, allowing for efficient accumulation of scattered light by multiplying precomputed form factors with local energy values [S1]. It is distinct from sibling methods like Most Probable Paths by focusing on a fixed set of scattering orders and directional contributions rather than path probability densities [S1][S2, book].

## How it works
The method approximates the Radiative Transfer Equation (RTE) by separating single scattering (computed separately) from multiple scattering, considering only the first three orders of multiple scattering to balance accuracy and performance [S1][S2, book].

### Inputs and Outputs
*   **Inputs:**
    *   Medium density field $\rho(x)$, typically defined on a voxel grid [S1][S2, book].
    *   Optical properties: extinction coefficient $\sigma_t$, scattering coefficient $\sigma_s$, and single scattering albedo $\omega_0 = \sigma_s / \sigma_t$ [S1][S2, book].
    *   Phase function $p(\omega_i, \omega_o)$, assumed to be strongly forward-scattering (e.g., Henyey-Greenstein) [S1][S2, book].
    *   Illumination sources: Sunlight direction $\omega_s$ and sky light distribution [S1][S2, book].
    *   Camera viewing direction $\omega_v$ [S1][S2, book].
*   **Outputs:**
    *   Radiance $L(x)$ for each voxel $x$ in the viewing direction, including contributions from single scattering and up to third-order multiple scattering [S1][S2, book].

### Data Structures
1.  **Voxel Grid:** The space containing the medium (e.g., cloud volume) is subdivided into a 3D grid of voxels. The viewing direction $\omega_v$ is aligned with one of the principal axes of this grid to simplify the calculation of contribution ratios [S1][S2, book].
2.  **3D-Filter (Contribution-Ratio Pattern):** A precomputed data structure representing the form factors. It stores the ratio of radiance contribution from neighboring voxels to a center voxel, accounting for 1 and 2 scattering events within the filter's scope [S1][S2, book].
    *   **Layout:** For each voxel, the filter contains weights corresponding to specific relative positions in the grid. These weights are derived from the mean density of the cloud [S1][S2, book].
    *   **Size:** The filter size is determined by the spatial extent of significant scattering contributions, which is small due to the forward-scattering nature of clouds [S1][S2, book].

### Steps

1.  **Grid Subdivision:** Subdivide the medium space into voxels, aligning the viewing direction $\omega_v$ with a principal axis of the grid [S1][S2, book].
2.  **Mean Density Space:** Define a smaller reference space using the mean density of the cloud. This space is used to compute the generic contribution ratios, making the filter applicable to varying densities [S1][S2, book].
3.  **Compute 3D-Filter (Precomputation):**
    *   Calculate the contribution ratios to the radiance of a center voxel (in the viewing direction) from other voxels in the mean density space [S1][S2, book].
    *   These ratios account for the sunlight direction $\omega_s$ and represent the light scattered due to second and third-order scattering events [S1][S2, book].
    *   **Optimization:** Since scattering is mainly forward, concentrate computational effort on voxels that effectively contribute to the center voxel for paths with one or two scatterings. Use a stochastic method to select these contributing voxels efficiently [S1][S2, book].
    *   Store these ratios as the 3D-filter [S1][S2, book].
4.  **Apply Filter (Rendering):**
    *   For each voxel $x$ in the actual medium grid:
        *   Retrieve the local energy (radiance) from neighboring voxels based on the 3D-filter weights [S1][S2, book].
        *   Multiply the filter weights by the energy at the related points [S1][S2, book].
        *   Accumulate these products to obtain the multiple scattering contribution $L_{ms}(x)$ for the second and third orders [S1][S2, book].
5.  **Single Scattering:** Compute single scattering contribution $L_{ss}(x)$ separately using standard ray-marching techniques [S1][S2, book].
6.  **Final Radiance:** Combine single and multiple scattering contributions:
    $$ L(x) = L_{ss}(x) + L_{ms}(x) $$
    [S1][S2, book]

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_3d_filter(mean_density, phase_function, sun_direction, num_scatterings=2):
    """
    Precomputes the 3D-filter (contribution-ratio pattern) for a given mean density.
    This represents the form factors for 2nd and 3rd order scattering.
    """
    # Define a small grid around the center voxel for the filter
    filter_grid_size = 5 # Example size, determined by scattering extent
    filter_weights = zeros(filter_grid_size, filter_grid_size, filter_grid_size)
    
    # Stochastic selection of contributing voxels to speed up computation
    # due to forward scattering nature
    contributing_voxels = stochastic_select_contributors(filter_grid_size, phase_function, sun_direction)
    
    for voxel in contributing_voxels:
        # Calculate contribution ratio from this voxel to the center
        # considering the path having 1 or 2 scatterings within the filter scope
        ratio = compute_contribution_ratio(
            voxel.position, 
            mean_density, 
            phase_function, 
            sun_direction
        )
        filter_weights[voxel.index] = ratio
        
    return filter_weights

def render_cloud(voxel_grid, filter_weights, single_scattering_radiance):
    """
    Renders the cloud by applying the precomputed 3D-filter to the voxel grid.
    """
    output_radiance = zeros_like(voxel_grid)
    
    # Apply filter to each voxel
    for x in voxel_grid:
        multiple_scattering_contribution = 0.0
        
        # Iterate over the filter kernel
        for offset in filter_weights.indices():
            neighbor = x + offset
            if neighbor is within bounds:
                # Multiply form factors by energy at related points
                energy = voxel_grid[neighbor].radiance
                weight = filter_weights[offset]
                multiple_scattering_contribution += weight * energy
                
        # Accumulate results
        output_radiance[x] = single_scattering_radiance[x] + multiple_scattering_contribution
        
    return output_radiance
```

### Parameters
*   **Scattering Orders:** Only the first three orders of multiple scattering are considered (i.e., up to third-order) to save computation time [S1][S2, book].
*   **Voxel Size:** Determined by the resolution of the medium and the extent of the 3D-filter [S1][S2, book].
*   **Mean Density:** Used to normalize the filter, allowing it to be applied to media with varying local densities [S1][S2, book].

### Edge Cases and Pitfalls
*   **Forward Scattering Assumption:** The method relies on the fact that scattering in clouds is mainly forward. If the phase function is highly isotropic or backward-scattering, the filter may not capture significant contributions, leading to inaccuracies [S1][S2, book].
*   **Boundary Conditions:** The method assumes an infinite or periodic medium for the filter computation. Edge effects in the actual medium may require special handling or larger filters [S1][S2, book].
*   **Single Scattering Separation:** Single scattering must be computed separately. If not handled correctly, double-counting or missing contributions can occur [S1][S2, book].
*   **Stochastic Selection:** The stochastic method for selecting contributing voxels introduces variance. A sufficient number of samples is needed to reduce noise in the precomputed filter [S1][S2, book].

## Operating characteristics
*   **Time Complexity:** The precomputation of the 3D-filter is done once per mean density configuration. Rendering time is linear with respect to the number of voxels, as it involves a simple convolution-like operation with the filter [S1][S2, book].
*   **Memory Complexity:** Requires storing the voxel grid and the 3D-filter. The filter size is small due to the forward-scattering assumption, making memory usage manageable [S1][S2, book].
*   **Strengths:** Efficient for media with high albedo and strong forward scattering, such as clouds and snow. Provides visually plausible results for multiple scattering without the high cost of full path tracing [S1][S2, book].
*   **Limitations:** Limited to specific scattering orders (up to third). Less accurate for media with isotropic or backward scattering. Requires precomputation, which may not be suitable for dynamic media with rapidly changing densities [S1][S2, book].
*   **When to Use:** Ideal for static or slowly varying cloud and snow rendering where multiple scattering is visually significant and computational resources are limited [S1][S2, book].
*   **When Not to Use:** Not suitable for highly dynamic media, media with complex phase functions, or when higher-order multiple scattering effects are critical [S1][S2, book].

## Background
The 3D-Filter method was proposed by Nishita et al. for rendering clouds with multiple scattering and sky light [S1][S2, book]. It was later applied to render snow, which has a high albedo and appears white due to multiple scattering [S1]. The shape and density distribution of snow in these applications are often modeled using metaballs [S1]. The method is part of a broader family of techniques that use implicit representations of directional distribution, such as scattering patterns and diffusion equations [S1][S2, book].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] Global Illumination Techniques for the Simulation of Participating Media, https://inria.hal.science/inria-00510096v1, fetched 2026-10-26 (book)
