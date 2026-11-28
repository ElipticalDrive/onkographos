# Volumetric Scattering Pass

Also called: Dipole Approximation Scattering, Multiple Scattering Approximation

## Overview
The Volumetric Scattering Pass is an additional rendering pass designed to incorporate indirect illumination and multiple scattering effects into volume rendering, thereby increasing visual realism [S1]. It addresses the limitation of single-scattering approximations by exploiting the insight that scattering influence is local due to the exponential falloff of light [S1]. Unlike its sibling [Depth-Aware Scatter Estimator](2585-depth-aware-scatter-estimator.md), which focuses on depth-aware single-scattering estimation, this method explicitly models the bleeding of color and light from neighboring voxels using a discrete neighborhood sum, allowing for interactive changes to light properties via Spherical Harmonics (SH) projection [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Spherical Harmonics coefficient volumes containing the previously computed in-scattering radiance $L_{mi}(\vec{x}, \vec{\omega}_i)$ for all voxels $\vec{x}$ and light directions $\vec{\omega}_i$ [S1].
*   **Output:** New Spherical Harmonics coefficient volumes storing the result of the scattering pass, denoted as $L_{mi}^0(\vec{x}, \vec{\omega}_i)$ [S1].

**Data Structures:**
*   **SH Coefficient Volumes:** Discrete 3D data structures where each voxel stores SH coefficients representing the angular distribution of light [S1].
*   **Neighborhood $N(\vec{x})$:** A set of voxel positions surrounding a target voxel $\vec{x}$ [S1].

**Parameters:**
*   **$R_d$ (Diffuse Reflectance):** A scalar parameter controlling the amount of scattering. Values are specified by Jensen et al. [JMLH01] for subsurface scattering simulation [S1].
*   **$|N(\vec{x})|$ (Neighborhood Size):** The number of voxels in the neighborhood. Assumed constant throughout the volume. A value of 10% of the maximum volume dimension yields good results [S1].

**Algorithm Steps:**
1. Iterate over every voxel $\vec{x}$ in the volume [S1].
2. Identify the neighborhood $N(\vec{x})$ of voxels surrounding $\vec{x}$ [S1].
3. For each light direction $\vec{\omega}_i$ (represented by SH coefficients):
    a. Initialize a sum accumulator [S1].
    b. Iterate over each neighbor voxel $\vec{x}_i \in N(\vec{x})$ [S1].
    c. Retrieve the pre-computed SH coefficients $L_{mi}(\vec{x}_i, \vec{\omega}_i)$ from the input volumes [S1].
    d. Add the contribution of the neighbor to the accumulator, weighted by the diffuse reflectance term $\frac{1}{\pi} R_d$ [S1].
    e. Multiply the total sum by the neighborhood size $|N(\vec{x})|$ (as per the specific discrete formulation in the source) [S1].
    f. Divide by the neighborhood size $|N(\vec{x})|$ (normalization) [S1].
    g. Store the resulting coefficients as $L_{mi}^0(\vec{x}, \vec{\omega}_i)$ in the output SH volumes [S1].
4. The output volumes $L_{mi}^0$ can now be used for final shading or further processing, allowing interactive light property changes [S1].

**Formula:**
The in-scattering reaching voxel $\vec{x}$ from its neighborhood $N(\vec{x})$ is computed as [S1]:

$$
L_{mi}^0(\vec{x}, \vec{\omega}_i) = \frac{1}{|N(\vec{x})|} \sum_{\vec{x}_i \in N(\vec{x})} \left( \frac{1}{\pi} R_d \right) \cdot L_{mi}(\vec{x}_i, \vec{\omega}_i) \cdot |N(\vec{x})|
$$

*Note: This simplifies to a simple average scaled by $\frac{1}{\pi} R_d$ [S1]. However, the text explicitly states the equation is inspired by Hao and Varshney [HV04] and modified for volumetric scattering [S1]. The implementation should follow the explicit discrete sum structure provided in Equation 4 of [S1].*

**Pitfalls and Edge Cases:**
*   **Cosine Weighting:** Neglect the cosine weighting term present in the original subsurface scattering equation by Hao and Varshney [HV04]. This method targets true volumetric scattering, not subsurface scattering [S1].
*   **Fresnel Term:** Drop the Fresnel term from the original subsurface scattering equation [S1].
*   **Neighborhood Size:** While the source assumes a constant neighborhood size, the resolution and average size of structures in the dataset should be considered when specifying $|N(\vec{x})|$ [S1]. An alternative approach is to adapt the neighborhood based on local material properties, though the constant 10% rule is cited as sufficient for good results [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_neighborhood(center, radius):
    """Returns a list of voxel coordinates within a cubic radius."""
    neighbors = []
    cx, cy, cz = center
    for dx in range(-radius, radius + 1):
        for dy in range(-radius, radius + 1):
            for dz in range(-radius, radius + 1):
                neighbors.append((cx + dx, cy + dy, cz + dz))
    return neighbors

def volumetric_scattering_pass(input_sh_volumes, output_sh_volumes, Rd, neighborhood_size, volume_dims, num_sh_coefficients):
    """
    Computes the volumetric scattering pass using the dipole approximation.
    
    Args:
        input_sh_volumes: 3D array of SH coefficients L_mi(x, omega_i)
        output_sh_volumes: 3D array to store results L_mi^0(x, omega_i)
        Rd: Diffuse reflectance parameter
        neighborhood_size: Number of voxels in the neighborhood |N(x)|
        volume_dims: Dimensions of the volume (max_dim used for default sizing if needed)
        num_sh_coefficients: Number of SH coefficients per voxel
    """
    PI = 3.141592653589793
    diffuse_weight = (1.0 / PI) * Rd
    
    # Estimate cubic radius to approximate the constant neighborhood size [S1]
    radius = int(neighborhood_size ** (1/3) / 2)
    if radius < 1:
        radius = 1
        
    for x in range(volume_dims[0]):
        for y in range(volume_dims[1]):
            for z in range(volume_dims[2]):
                current_voxel = (x, y, z)
                neighbors = get_neighborhood(current_voxel, radius)
                
                for sh_idx in range(num_sh_coefficients):
                    sum_contribution = 0.0
                    for neighbor_voxel in neighbors:
                        l_mi_neighbor = input_sh_volumes[neighbor_voxel][sh_idx]
                        sum_contribution += diffuse_weight * l_mi_neighbor
                    
                    # Explicit discrete formulation from source [S1]
                    result = (1.0 / len(neighbors)) * sum_contribution * len(neighbors)
                    output_sh_volumes[current_voxel][sh_idx] = result
```

## Operating characteristics
*   **Complexity:** The method operates on a per-voxel basis, iterating over a fixed-size neighborhood $N(\vec{x})$. The complexity is $O(V \cdot |N| \cdot D)$, where $V$ is the number of voxels, $|N|$ is the neighborhood size, and $D$ is the number of SH coefficients (light directions) [S1].
*   **Performance:** The approach is designed for interactivity. By storing results in SH coefficient volumes, it allows for efficient storage and real-time modification of light properties [S1].
*   **Strengths:** Incorporates color bleeding and indirect illumination effects that single-scattering approximations miss [S1]. It is computationally cheaper than full multiple scattering integration while providing comparable accuracy to diffusion approximation with volumetric source distribution [S1].
*   **Limitations:** Relies on the assumption that scattering influence is local [S1]. The fixed neighborhood size may not adapt well to varying material properties or resolution changes without manual tuning [S1]. It is an approximation and may not capture complex global scattering paths accurately [S1].

## Background
The method was introduced by Hao and Varshney in the context of subsurface scattering, noting that scattering influence is local due to exponential light falloff [HV04]. The Volumetric Scattering Pass adapts this insight for volume rendering, specifically for participating media [S1]. The diffuse reflectance parameter $R_d$ and the dipole approximation basis are derived from the work of Jensen et al. [JMLH01], who used these parameters for subsurface scattering simulation [S1]. The specific application to volumetric scattering, neglecting cosine weighting and Fresnel terms, was detailed in "Advanced Light Material Interaction for Direct Volume Rendering" [S1].

## Sources
[S1] Advanced Light Material Interaction for Direct Volume Rendering, https://doi.org/10.2312/vg/vg10/101-108, fetched 2026-10-26
