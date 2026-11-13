# 3D-Filter / Concrete Directions

Also called: 3D-Filter / Nishita, Nishita et al. 3D-Filter, 3D-Filter / N Directions, Concrete Directions Method

## Overview
The 3D-Filter method is a deterministic approximation for rendering multiple scattering in participating media, specifically designed for clouds and snow [S1][S2, book]. It solves the problem of underestimating radiance in optically thick media by precomputing a "contribution-ratio pattern" (the 3D-filter) that encodes the directional distribution of scattered light [S1][S2, book]. Unlike stochastic methods that trace random walks, this approach uses extended form factors in a voxel grid to accumulate second- and third-order scattering contributions efficiently [S1][S2, book]. It sets itself apart from siblings like Most Probable Paths by restricting the scattering order to a small fixed number (typically three) and exploiting the forward-scattering nature of media like clouds to reduce the computational complexity of form factor calculation [S1][S2, book].

## How it works
**Inputs:**
*   A 3D voxel grid representing the participating medium (e.g., cloud or snow).
*   The viewing direction $\omega_v$ (assumed constant or principal axis for the filter generation).
*   The light source direction $\omega_l$ (e.g., sunlight).
*   Optical properties: scattering coefficient $\sigma_s$, absorption coefficient $\sigma_a$, and phase function $p$.
*   Single scattering radiance $L_{ss}$ (computed separately).

**Outputs:**
*   Final radiance $L(\mathbf{x})$ for each voxel $\mathbf{x}$ in the viewing direction.

**Data Structures:**
*   **Voxel Grid:** The space is subdivided into voxels. The grid resolution is determined by the medium's extent.
*   **3D-Filter (Contribution-Ratio Pattern):** A precomputed data structure storing contribution ratios. For a center voxel, this pattern represents the fraction of energy arriving from other voxels that contributes to the radiance in the viewing direction, considering paths with 1 or 2 scatterings (for 2nd and 3rd order total scattering).
*   **Energy Field:** The density or optical thickness distribution of the medium.

**Steps:**

1.  **Voxelization:** Subdivide the space containing the medium into a grid of voxels. Align the grid such that the viewing direction $\omega_v$ serves as a principal axis [S1][S2, book].

2.  **Single Scattering Computation:** Compute the single scattering radiance $L_{ss}(\mathbf{x})$ for all voxels $\mathbf{x}$ separately. This accounts for direct illumination from the light source [S1][S2, book].

3.  **Filter Generation (Precomputation):**
    *   Define a smaller reference space using the mean density of the cloud/medium to simplify calculations [S1][S2, book].
    *   Identify the center voxel $\mathbf{x}_c$ in the viewing direction.
    *   Compute the contribution ratios from other voxels $\mathbf{x}_i$ to $\mathbf{x}_c$. These ratios represent the extended form factors for paths involving one or two scatterings (which contribute to the 2nd and 3rd order scattering terms) [S1][S2, book].
    *   **Optimization:** Since scattering in clouds is predominantly forward, concentrate computational effort on voxels $\mathbf{x}_i$ that effectively contribute to $\mathbf{x}_c$. Use a stochastic method to select these contributing voxels rather than computing form factors for all pairs [S1][S2, book].
    *   Store these ratios in the 3D-filter structure [S1][S2, book].

4.  **Filter Application (Rendering):**
    *   For each voxel $\mathbf{x}$ in the full space, apply the 3D-filter.
    *   Multiply the precomputed form factors (from the filter) by the energy (radiance/density) at the related voxels [S1][S2, book].
    *   Accumulate these products to obtain the multiple scattering contribution $L_{ms}(\mathbf{x})$ for the second and third orders [S1][S2, book].
    *   Note: Only the first three orders of scattering are considered for sunlight multiple scattering to save computation time [S1][S2, book].

5.  **Final Radiance:**
    *   Combine single and multiple scattering contributions:
        $$ L(\mathbf{x}) = L_{ss}(\mathbf{x}) + L_{ms}(\mathbf{x}) $$
    *   Add sky light contributions if applicable (sky light is typically treated as single scattering) [S2, book].

**Parameters:**
*   **Maximum Scattering Order:** Typically 3 (1st, 2nd, 3rd orders). Higher orders are neglected [S1][S2, book].
*   **Voxel Resolution:** Determines the granularity of the filter and the medium representation.

**Edge Cases and Pitfalls:**
*   **Heterogeneity:** The method uses a mean density for filter generation, which may introduce inaccuracies in highly heterogeneous media [S1][S2, book].
*   **Viewing Direction:** The filter is generated for a specific viewing direction (principal axis). Changing the view significantly may require regeneration or interpolation of filters [S1][S2, book].
*   **Backward Scattering:** The method exploits forward scattering. Media with strong backward scattering components may not be rendered accurately [S1][S2, book].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_3d_filter(cloud_voxels, mean_density, viewing_direction, light_direction):
    """
    Precomputes the 3D-filter (contribution-ratio pattern) for a center voxel.
    """
    # Define a smaller reference space with mean density
    reference_space = create_reference_space(mean_density)
    
    # Identify center voxel in viewing direction
    center_voxel = reference_space.get_voxel_in_direction(viewing_direction)
    
    filter_data = {}
    
    # Stochastic selection of contributing voxels for paths with 1 or 2 scatterings
    # due to forward scattering nature
    contributing_voxels = stochastic_select_contributors(
        reference_space, 
        center_voxel, 
        max_scatterings=2,
        light_direction=light_direction
    )
    
    for voxel_i in contributing_voxels:
        # Compute extended form factor / contribution ratio
        # This accounts for light traveling from voxel_i to center_voxel
        # with 1 or 2 scatterings
        ratio = compute_contribution_ratio(
            voxel_i, 
            center_voxel, 
            viewing_direction, 
            light_direction,
            reference_space.optical_properties
        )
        filter_data[voxel_i.id] = ratio
        
    return filter_data

def render_with_3d_filter(cloud_voxels, single_scattering_radiance, filter_data, viewing_direction):
    """
    Applies the precomputed 3D-filter to compute multiple scattering.
    """
    final_radiance = {}
    
    for voxel in cloud_voxels:
        # Start with single scattering
        ms_contribution = 0.0
        
        # Apply filter: multiply form factors by energy at related points
        for related_voxel_id, form_factor in filter_data.items():
            # Retrieve energy/radiance at the related voxel
            # Note: In practice, this might involve interpolation or direct lookup
            energy = get_energy_at_voxel(related_voxel_id, cloud_voxels)
            
            # Accumulate contribution
            ms_contribution += form_factor * energy
            
        # Combine single and multiple scattering (2nd and 3rd orders)
        final_radiance[voxel.id] = single_scattering_radiance[voxel.id] + ms_contribution
        
    return final_radiance
```

## Operating characteristics
*   **Time Complexity:** The precomputation of the 3D-filter is expensive but done once. Rendering is fast as it involves a lookup and multiplication per voxel. The stochastic selection of voxels reduces the number of form factor calculations from $O(N^2)$ to a smaller subset [S1][S2, book].
*   **Memory Complexity:** Requires storing the 3D-filter for each relevant viewing direction and the voxel grid. The filter size depends on the number of contributing voxels selected stochastically [S1][S2, book].
*   **Performance:** Significantly faster than stochastic multiple scattering methods for clouds and snow [S1][S2, book].
*   **Strengths:** Efficient handling of multiple scattering in homogeneous or moderately heterogeneous media; exploits forward scattering [S1][S2, book].
*   **Limitations:** Limited to a small number of scattering orders (typically 3); less accurate for highly heterogeneous media or strong backward scattering; view-dependent [S1][S2, book].
*   **When to use:** Rendering clouds and snow where multiple scattering is important and performance is a concern [S1][S2, book].
*   **When not to use:** Highly heterogeneous media, media with strong backward scattering, or when high accuracy for higher-order scattering is required [S1][S2, book].

## Background
The 3D-Filter method was proposed by Nishita et al. [S1][S2, book]. It was initially developed for rendering clouds, taking into account multiple scattering of sunlight and single scattering of sky light [S2, book]. The method was later applied to render snow, which has a high albedo and appears white due to multiple scattering [S1][S2, book]. The approach is part of a broader class of methods that use implicit representations of directional radiance, such as scattering patterns and diffusion equations [S1][S2, book].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] Global Illumination Techniques for the Simulation of Participating Media, https://inria.hal.science/inria-00510096v1, fetched 2026-10-26 (book)
