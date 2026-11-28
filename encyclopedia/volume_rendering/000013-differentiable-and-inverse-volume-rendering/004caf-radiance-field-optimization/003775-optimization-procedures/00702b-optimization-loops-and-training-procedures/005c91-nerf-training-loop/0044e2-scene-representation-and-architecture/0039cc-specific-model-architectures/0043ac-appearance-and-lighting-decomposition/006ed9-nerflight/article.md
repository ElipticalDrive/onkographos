# NeRFLight


## Overview
NeRFLight is a grid-based neural radiance field architecture that achieves real-time view synthesis with a significantly reduced memory footprint compared to standard grid-based methods like Instant-NGP or KiloNeRF [S1, S2]. It solves the trade-off between rendering speed and model size by splitting the 3D scene volume into $N$ distinct spatial regions, each utilizing a separate density decoder but sharing a single, compact feature grid [S1]. Unlike Instant-NGP, which relies on multiresolution hash grids that can suffer from rendering speed losses due to multiple linear interpolations, NeRFLight avoids hash grids entirely and uses multiple density decoders to handle feature collisions, ensuring high frame rates [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Training images with known camera poses $(o_j, d_j)$ defining rays $r_j(t) = o_j + d_j t$, and a 3D scene volume bounded by a region of interest [S1].
*   **Output:** Rendered image pixel color $C(\mathbf{r})$ for a given ray, and a compact model representation consisting of a shared feature grid and decoder weights suitable for inference [S1].

**Data Structures:**
1.  **Shared Feature Grid:** A 3D grid of latent features arranged symmetrically across the $N$ regions, forcing features to learn a compact representation valid for different parts of the scene [S1].
2.  **Density Decoders:** $N$ separate Multi-Layer Perceptrons (MLPs), denoted as $\theta_i$ for region $i \in [1, N]$, which take features from the shared grid and output density $\sigma$ and an intermediate latent representation $h$ [S1].
3.  **Color Decoder:** A single MLP that takes the intermediate representation $h$, the viewing direction $\mathbf{d}$, and positional encodings to output the final color $\mathbf{c}$ [S1, S2].

**Numbered Steps:**
1.  **Scene Partitioning:** Divide the 3D scene volume into $N$ distinct spatial regions [S1].
2.  **Feature Grid Initialization:** Create a single shared feature grid with features arranged symmetrically across the regions to favor feature pruning and smooth gradient transitions between neighboring voxels [S1].
3.  **Ray Sampling:** For a given ray $\mathbf{r}$, sample points $\mathbf{x}_j$ along the ray using a coarse-to-fine strategy [S1].
4.  **Region Identification:** Determine which of the $N$ regions a sample point $\mathbf{x}_j$ falls into, assigning it an index $i$ [S1].
5.  **Feature Lookup:** Retrieve the latent features $\hat{f}(\mathbf{x}_j)$ from the shared feature grid at position $\mathbf{x}_j$ via trilinear interpolation [S1].
6.  **Density and Latent Prediction:** Pass the features $\hat{f}(\mathbf{x}_j)$ to the region-specific density decoder $\theta_i$ to compute $\sigma_j$ and $h_j$ [S1].
7.  **Color Prediction:** Pass the intermediate representation $h_j$, the viewing direction $\mathbf{d}$, and positional encodings to the color decoder to obtain the color $\mathbf{c}_j$ [S1, S2].
8.  **Volumetric Rendering:** Compute the pixel color $C(\mathbf{r})$ using deterministic volume integration instead of Monte Carlo integration [S1].

**Formulas:**
$$
\alpha_j = 1 - \exp(-\sigma_j \delta_j)
$$
$$
T_j = \prod_{k=1}^{j-1} (1 - \alpha_k)
$$
$$
C(\mathbf{r}) = \sum_{j=1}^{M} T_j \alpha_j \mathbf{c}_j
$$
where $\alpha_j$ is the opacity, $T_j$ is the accumulated transmittance, $\delta_j$ is the distance between samples, and $M$ is the total number of samples along the ray [Parent, S1].

**Parameters:**
*   $N$: The number of regions the volume is split into. In the canonical implementation, $N=8$ [S1, S2].
*   Positional Encoding Bands ($L$): Not explicitly specified in the sources for NeRFLight, but the parent article defines the general form. Typical values in NeRF variants are $L=10$ for position and $L=4$ for direction [Parent].

**Edge Cases and Pitfalls:**
*   **Feature Collisions:** Sharing the feature grid forces compactness but may lead to ambiguity if distinct features in different regions map to the same grid location; NeRFLight mitigates this by using multiple density decoders [S1].
*   **Boundary Artifacts:** The symmetric feature configuration is designed to favor smooth gradient transitions between neighboring voxels, reducing artifacts at region boundaries [S1].
*   **Empty Space:** Empty space pruning is applied to accelerate model inference by removing features that do not contribute to the visible scene [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def positional_encoding(x, num_bands):
    """Computes positional encoding gamma(x) with L bands."""
    freq_bands = [2.0 ** i for i in range(num_bands)]
    scaled_x = x.unsqueeze(-1) * freq_bands * 3.141592653589793
    return torch.cat([torch.sin(scaled_x), torch.cos(scaled_x)], dim=-1)

def trilinear_interpolate(grid, x):
    """Trilinearly interpolates features from a 3D grid at continuous coordinates x."""
    # Standard trilinear interpolation implementation
    pass

def get_region_index(x, num_regions):
    """Determines which of the N regions a 3D point x belongs to based on grid partitioning."""
    # Maps continuous coordinates to region index i in [0, num_regions-1]
    pass

def render_ray(ray_origin, ray_direction, shared_feature_grid, density_decoders, color_decoder, num_regions=8, step=0.5, max_dist=100.0):
    """Renders a single ray using the NeRFLight architecture."""
    # 1. Sample points along the ray with fixed step size
    t_values = torch.arange(0.0, max_dist, step)
    samples = ray_origin + ray_direction.unsqueeze(0) * t_values.unsqueeze(-1)
    deltas = torch.full_like(t_values, step)
    
    accumulated_color = torch.zeros(3)
    accumulated_transmittance = 1.0
    
    for j in range(len(t_values)):
        x_j = samples[j]
        delta_j = deltas[j]
        
        # 2. Determine region index
        region_index = get_region_index(x_j, num_regions)
        
        # 3. Lookup features
        features = trilinear_interpolate(shared_feature_grid, x_j)
        
        # 4. Density and latent prediction
        sigma_j, h_j = density_decoders[region_index](features)
        sigma_j = torch.clamp(sigma_j, min=0.0)
        
        # 5. Compute opacity and transmittance
        alpha_j = 1.0 - torch.exp(-sigma_j * delta_j)
        
        # 6. Color prediction
        gamma_x_j = positional_encoding(x_j, num_bands=10)
        gamma_d_j = positional_encoding(ray_direction, num_bands=4)
        color_j = color_decoder(h_j, gamma_x_j, gamma_d_j)
        
        # 7. Accumulate
        weight = accumulated_transmittance * alpha_j
        accumulated_color += weight * color_j
        accumulated_transmittance *= (1.0 - alpha_j)
        
        # Early termination
        if accumulated_transmittance < 1e-5:
            break
            
    return accumulated_color
```

## Operating characteristics
*   **Rendering Speed:** Achieves real-time performance of up to 181 FPS [S1].
*   **Model Size:** The model size is approximately 14MB [S1].
*   **Efficiency:** The FPS/MB ratio is 2x larger than the closest state-of-the-art approach at the time of publication [S1].
*   **Memory Footprint:** The shared feature grid reduces the total number of stored feature values by a factor of $N$ (8x reduction compared to separate grids) [S1, S2].
*   **Strengths:** High rendering speed, small model size, real-time capability, seamless reconstruction due to symmetric feature configuration [S1].
*   **Limitations:** The method relies on a fixed number of regions ($N=8$ in the paper), which may not adapt well to scenes of varying complexity without retraining [S1]. It does not use hash grids or multiresolution schemes, which might limit its ability to capture fine details in very large scenes compared to Instant-NGP [S1].
*   **When to use:** When real-time rendering and low memory footprint are critical, and the scene can be effectively partitioned into a small number of regions [S1].
*   **When not to use:** For scenes requiring extremely high detail across a vast volume where a fixed partitioning scheme is insufficient, or when dynamic resolution adaptation is needed [S1].

## Background
NeRFLight was introduced by Rivas-Manzaneque et al. in 2023 [S1]. It extends grid-based NeRF approaches by addressing the memory footprint issues that prevent deployment on bandwidth-constrained applications [S1]. The method builds on the idea of splitting the density field into multiple regions with separate decoders, similar to KiloNeRF, but avoids baking a large number of MLPs by using a shared feature grid [S1]. It also leverages deterministic volume integration from [48] to improve accuracy and speed [S1]. The approach was evaluated on synthetic and real data, demonstrating state-of-the-art FPS/MB ratios [S1].

## Sources
[S1] NeRFLight: Fast and Light Neural Radiance Fields using a Shared Feature Grid, https://doi.org/10.1109/cvpr52729.2023.01195, fetched 2026-10-26
[S2] NeRFLight: Fast and Light Neural Radiance Fields using a ..., https://www.iri.upc.edu/files/scidoc/2861-NeRFLight%3A-Fast-and-Light-Neural-Radiance-Fields-using-a-Shared-Feature-Grid.pdf, fetched 2026-10-27
