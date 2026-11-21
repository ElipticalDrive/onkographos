# Volumetric Unsharp Masking with Multi-scale Hierarchy

Also called: Darkening Shadowed Regions, Enhanced perception method, Feature enhancement technique, Lightening Salient Features, Volumetric Unsharp Masking, feature enhancement by unsharp masking

## Overview
Volumetric Unsharp Masking with Multi-scale Hierarchy is a feature enhancement technique for volume rendering that improves the perception of local details (edges, superficial features) by increasing local contrast through lightening salient features and darkening shadowed regions [S1, unofficial]. Unlike classical unsharp masking which requires storing full-resolution smoothed radiance volumes, this method reduces memory consumption by using one or two levels of a multi-scale volume hierarchy of the original data to compute the smoothed signals required for the unsharp masking operation [S1, unofficial]. It differs from [Lit Sphere Shading](785-lit-sphere-shading.md) and [Volume Illustration Tone Shading](805-volume-illustration-tone-shading.md) by explicitly decoupling feature enhancement from the base shading model, applying a post-shading contrast modification in the CIELab color space [S1, unofficial].

## How it works
The method operates on the radiance signal computed along viewing rays during ray casting. It enhances local contrast by modifying the shading of each sample using unsharp masking in the CIELab color space, utilizing a multi-scale hierarchy to approximate the smoothed background signal without storing full-resolution auxiliary volumes.

### Inputs and Outputs
*   **Inputs:**
    *   Original volume dataset $V$ with resolution $D_x \times D_y \times D_z$.
    *   Transfer function $TF$ mapping density to color and opacity.
    *   Lighting parameters (light direction $L$, view direction $V$, material properties for Phong shading).
    *   Hierarchy levels $H = \{h_2, h_3, h_4, h_5, h_6, h_8, h_{16}, h_{32}\}$, where $h_i$ represents the volume downsampled by factor $i$.
    *   Parameters: $\gamma$ (lightening intensity), $\lambda_{dark}$ (darkening intensity, implicitly 1 in the final combination formula), level for lightening $L_{lev}$, level for darkening $D_{lev}$.
*   **Outputs:**
    *   Enhanced final color $C_{final}$ for each sample along the ray.

### Data Structures
1.  **Original Volume ($V$):** 3D texture containing the scalar field.
2.  **Multi-scale Hierarchy ($H$):** A set of 3D textures. Each level $h_i$ has dimensions $\lceil D_x/i \rceil \times \lceil D_y/i \rceil \times \lceil D_z/i \rceil$. The hierarchy is computed on the CPU and uploaded to the GPU. Only one or two levels are stored in memory at a time, significantly reducing memory usage compared to storing full-resolution smoothed radiance volumes [S1, unofficial].
3.  **Ray Casting State:** Accumulated color and opacity.

### Numbered Steps

1.  **Preprocessing (CPU):**
    *   Compute the multi-scale volume hierarchy from the original data set $V$.
    *   For each level $i \in \{2, 3, 4, 5, 6, 8, 16, 32\}$, generate a volume where dimensions are divided by $i$. This is done by averaging or filtering the original data over blocks of size $i \times i \times i$ [S1, unofficial].
    *   Upload $V$ and the selected hierarchy levels to GPU 3D textures.

2.  **Ray Casting Loop (GPU):**
    For each sample point $P$ along the viewing ray:
    
    2.1. **Compute Base Radiance:**
    *   Query the density $d$ at $P$ from $V$.
    *   Compute the Phong shading color $C_{phong}$ using the local gradient $\nabla f$, light vector, and view vector.
    *   Transform $C_{phong}$ from RGB to CIELab, yielding $[L, a, b]$ [S1, unofficial].

    2.2. **Compute Lightening Signal ($S_{light}$):**
    *   Query the density $d_{smooth}$ at $P$ from the hierarchy level $L_{lev}$ using trilinear interpolation.
    *   Compute the Phong shading color $C_{smooth}$ for density $d_{smooth}$ using the same lighting model.
    *   Transform $C_{smooth}$ to CIELab, yielding $[L_{level}, a_{level}, b_{level}]$.
    *   **Edge Case:** If the opacity assigned by $TF$ to $d_{smooth}$ is zero, replace $[L_{level}, a_{level}, b_{level}]$ with a color having lower luminance than $[L, a, b]$ to ensure the unsharp masking results in lightening [S1, unofficial].
    *   Calculate the unsharped CIELab color $UCIELab(C)$:
        $$ L_{new} = L + \gamma(L - L_{level}) $$
        $$ k = \frac{L_{new}}{L} $$
        $$ a_{new} = k \cdot a $$
        $$ b_{new} = k \cdot b $$
        $$ UCIELab(C) = [L_{new}, a_{new}, b_{new}] $$
        where $\gamma$ is the user-defined scaling factor for lightening intensity [S1, unofficial].
    *   Transform $UCIELab(C)$ back to RGB to get $C_{light}$ [S1, unofficial].

    2.3. **Compute Darkening Signal ($S_{dark}$):**
    *   Query the density $d_{tf}$ at $P$ from the hierarchy level $D_{lev}$ using trilinear interpolation.
    *   Compute the color $C_{tf}$ assigned by the transfer function for density $d_{tf}$.
    *   Transform $C_{tf}$ to CIELab, yielding $[L_{tf}, a_{tf}, b_{tf}]$.
    *   Calculate the darkened CIELab color $DCIELab(C)$:
        $$ L_{dark} = L + \gamma(L - L_{tf}) $$
        $$ k_{dark} = \frac{L_{dark}}{L} $$
        $$ a_{dark} = k_{dark} \cdot a $$
        $$ b_{dark} = k_{dark} \cdot b $$
        $$ DCIELab(C) = [L_{dark}, a_{dark}, b_{dark}] $$
        *Note: The formula in the source uses the same $\gamma$ for darkening intensity, but applies it to the difference between the sample's luminance and the transfer function's luminance at the hierarchy level [S1, unofficial].*
    *   Transform $DCIELab(C)$ back to RGB to get $C_{dark}$ [S1, unofficial].

    2.4. **Combine Effects:**
    *   Compute the final enhanced color for the sample:
        $$ C_{sample} = 0.5 \cdot (C_{light} + C_{dark}) $$
    *   Use the original opacity $\alpha$ from the transfer function for compositing [S1, unofficial].

    2.5. **Compositing:**
    *   Composite $C_{sample}$ and $\alpha$ into the ray accumulator using standard front-to-back alpha compositing.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_enhanced_shading(sample_pos, volume, hierarchy_levels, transfer_func, lighting_params, gamma, level_light, level_dark):
    """
    Computes the enhanced shading for a single sample point.
    
    Args:
        sample_pos: (x, y, z) coordinates in volume space.
        volume: 3D texture of the original dataset.
        hierarchy_levels: Dictionary mapping level index (int) to 3D texture.
        transfer_func: Function mapping density to (color_rgb, opacity).
        lighting_params: Object containing light direction, view direction, etc.
        gamma: Scaling factor for unsharp masking intensity.
        level_light: Integer, hierarchy level index for lightening.
        level_dark: Integer, hierarchy level index for darkening.
        
    Returns:
        final_color_rgb: RGB color for the sample.
        opacity: Float, opacity for compositing.
    """
    
    # 1. Get original density and compute base Phong color
    density_orig = volume.sample_trilinear(sample_pos)
    color_orig_rgb = compute_phong_shading(density_orig, sample_pos, lighting_params)
    color_orig_lab = rgb_to_cielab(color_orig_rgb)
    L, a, b = color_orig_lab
    
    # 2. Lightening Component
    # Query smoothed density from hierarchy
    density_smooth = hierarchy_levels[level_light].sample_trilinear(sample_pos)
    
    # Compute smoothed Phong color
    color_smooth_rgb = compute_phong_shading(density_smooth, sample_pos, lighting_params)
    color_smooth_lab = rgb_to_cielab(color_smooth_rgb)
    L_smooth, a_smooth, b_smooth = color_smooth_lab
    
    # Edge case: if opacity of smoothed density is 0, use a darker color for L_smooth
    # to ensure lightening effect. Assuming transfer_func provides opacity.
    opacity_smooth = transfer_func.get_opacity(density_smooth)
    if opacity_smooth == 0:
        # Replace with a color of lower luminance than original
        # Simple heuristic: reduce L significantly
        L_smooth = L * 0.5 
        a_smooth = 0
        b_smooth = 0
        
    # Apply Unsharp Masking in CIELab
    L_new = L + gamma * (L - L_smooth)
    # Preserve saturation by scaling a and b by the ratio of new luminance to old luminance
    if L != 0:
        k = L_new / L
    else:
        k = 1.0 # Avoid division by zero, though L=0 is rare in shaded regions
        
    a_new = k * a
    b_new = k * b
    
    color_light_lab = [L_new, a_new, b_new]
    color_light_rgb = cielab_to_rgb(color_light_lab)
    
    # 3. Darkening Component
    # Query density from hierarchy for darkening
    density_tf = hierarchy_levels[level_dark].sample_trilinear(sample_pos)
    
    # Get color assigned by transfer function for this density
    color_tf_rgb, _ = transfer_func.get_color_and_opacity(density_tf)
    color_tf_lab = rgb_to_cielab(color_tf_rgb)
    L_tf, a_tf, b_tf = color_tf_lab
    
    # Apply Unsharp Masking formula for darkening
    # Note: Source uses same gamma for darkening intensity
    L_dark_new = L + gamma * (L - L_tf)
    
    if L != 0:
        k_dark = L_dark_new / L
    else:
        k_dark = 1.0
        
    a_dark_new = k_dark * a
    b_dark_new = k_dark * b
    
    color_dark_lab = [L_dark_new, a_dark_new, b_dark_new]
    color_dark_rgb = cielab_to_rgb(color_dark_lab)
    
    # 4. Combine
    final_color_rgb = 0.5 * (color_light_rgb + color_dark_rgb)
    
    # 5. Get opacity for compositing (from original sample)
    _, opacity = transfer_func.get_color_and_opacity(density_orig)
    
    return final_color_rgb, opacity

def compute_phong_shading(density, pos, lighting_params):
    """
    Computes standard Phong shading color.
    This is a placeholder for the specific Phong implementation used in the parent method.
    """
    # Gradient approximation
    grad = compute_gradient(pos)
    normal = normalize(grad)
    
    # Lighting calculation
    # ... standard Phong model ...
    # Return RGB color
    pass

def compute_gradient(pos):
    """
    Computes the gradient of the volume at pos.
    """
    # Central differences or similar
    pass

def rgb_to_cielab(rgb):
    """Converts RGB (0-1) to CIELab."""
    pass

def cielab_to_rgb(lab):
    """Converts CIELab to RGB (0-1)."""
    pass
```

### Parameters
*   $\gamma$: User-defined scaling factor controlling the intensity of the lightening/darkening effect. Typical values are not explicitly given, but it is a user-tunable parameter [S1, unofficial].
*   $L_{lev}$ (Level for lightening): Determines the scale of features to enhance. Levels 2 to 5 are typically used. Low scaling factors (e.g., level 2) detect small features; higher factors (e.g., level 8) enhance larger features [S1, unofficial].
*   $D_{lev}$ (Level for darkening): Determines the scale of shadowed regions to darken. Levels 2 or 3 are typically used. Using lower resolution levels (higher scaling factors) for darkening produces extra, unpleasant shadowed regions [S1, unofficial].

### Edge Cases and Pitfalls
*   **Zero Opacity in Hierarchy:** If the opacity assigned by the transfer function to the average density (from the hierarchy level) is zero, the Phong color of the average must be replaced by a color with lower luminance than the sample's to ensure the unsharp masking results in lightening [S1, unofficial].
*   **Darkening Level Selection:** Using lower resolution levels (higher scaling factors) for darkening produces extra, unpleasant shadowed regions. Levels 2 or 3 are recommended [S1, unofficial].
*   **Isosurface Sampling:** Applying unsharp masking only at the isosurface level produces poor results because the contribution of a single sample to the final color may be quite low, especially when semi-transparent materials are traversed before reaching the isosurface. Performance gain is only 2% to 10% [S1, unofficial].
*   **Performance with Different Levels:** Using different levels for lightening and darkening reduces texture coherence, causing frame rates to decrease up to 50% compared to using the same level (which decreases up to 20%) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Hierarchy building is $O(N)$ where $N$ is the number of voxels, but the constant factor is low. Building times for various datasets range from 192ms (192x180x168) to 4559ms (512x512x512) [S1, unofficial].
*   **Memory Complexity:** The method stores the original volume and one or two levels of the hierarchy. In the worst case (using levels 2 and 3), memory consumption increases by less than a sixth of the original volume, compared to triple storage in Tao et al.'s method [S1, unofficial].
*   **Measured Performance:** On an Intel Core2 DUO CPU at 3.0 GHz and nVidia GeForce GTX 280 GPU with 1GB RAM:
    *   Using the same level for lightening and darkening: Frame rates decrease up to 20% compared to standard Phong shading [S1, unofficial].
    *   Using different levels: Frame rates decrease up to 50% due to less texture coherence [S1, unofficial].
    *   This is an improvement over Tao et al.'s method, which saw up to 70% frame rate decay [S1, unofficial].
*   **Strengths:**
    *   Significantly reduced memory requirements compared to full-resolution smoothed radiance volumes.
    *   Effective enhancement of local features in both opaque and semi-transparent volumes.
    *   User studies show 72% preference for this method over Phong and Tao's method [S1, unofficial].
*   **Limitations:**
    *   Performance penalty when using different hierarchy levels for lightening and darkening.
    *   Requires careful selection of hierarchy levels based on feature size.
    *   Not suitable for real-time applications requiring very high frame rates on low-end hardware if different levels are used.

## Background
The method was proposed to overcome the high memory consumption limitations of the volumetric unsharp masking technique by Tao et al. [105], which required storing two additional full-resolution volumes (radiance and smoothed radiance) [S1, unofficial]. The multi-scale volume hierarchy approach allows for non-power-of-two levels and sparse storage, making it more efficient [S1, unofficial]. The technique was evaluated in a user study involving 252 participants, where it was preferred for feature perception [S1, unofficial].

## Sources
[S1] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
