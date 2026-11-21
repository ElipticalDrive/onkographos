# Regions of Emphasis (Opacity Modulation)
Also called: Focus + context opacity modulation

## Overview
Regions of Emphasis is a focus+context technique that uses opacity modulation to highlight specific density ranges within a volume while preserving the surrounding context [S1, unofficial]. It differs from geometric deformation methods by altering visual transparency rather than spatial geometry, allowing users to select a region of interest based on data values [S1, unofficial].

## How it works
**Inputs:**
*   Volume dataset $V$ with density values $d$.
*   User-defined region of emphasis $R_{emph}$, defined as a range of density values $[d_{min}, d_{max}]$ [S1, unofficial].
*   Base transfer function mapping density to color and opacity [S1, unofficial].

**Outputs:**
*   Rendered image where features within $R_{emph}$ are emphasized and features outside are suppressed [S1, unofficial].

**Data Structures:**
*   The region of emphasis is stored as a pair of scalar values $(d_{min}, d_{max})$ [S1, unofficial].
*   Opacity is modulated per sample along the ray [S1, unofficial].

**Steps:**
1.  **Selection:** The user defines the region of emphasis $R_{emph} = [d_{min}, d_{max}]$ by selecting a range of density values [S1, unofficial].
2.  **Ray Casting:** For each pixel, cast a ray through the volume and sample density values $d_i$ [S1, unofficial].
3.  **Opacity Modulation:** For each sample $i$, determine if $d_i \in R_{emph}$ [S1, unofficial].
    *   If $d_{min} \le d_i \le d_{max}$, the sample retains its base opacity $\alpha_{base}$ [S1, unofficial].
    *   If $d_i < d_{min}$ or $d_i > d_{max}$, the opacity is decreased to $\alpha_{mod} < \alpha_{base}$ [S1, unofficial].
4.  **Compositing:** Composite the samples using standard front-to-back or back-to-front alpha blending with the modulated opacities [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_regions_of_emphasis(volume, camera, region_emphasis):
    """
    volume: 3D array of density values
    camera: defines ray origins and directions
    region_emphasis: tuple (d_min, d_max)
    """
    d_min, d_max = region_emphasis
    image = empty_image(camera.resolution)
    
    for y, x in camera.pixel_coordinates():
        ray = camera.get_ray(x, y)
        samples = volume.sample_ray(ray)
        
        accumulated_color = (0, 0, 0, 0)
        accumulated_alpha = 0
        
        for sample in samples:
            density = sample.density
            base_opacity = sample.base_opacity
            
            # Modulate opacity based on region of emphasis
            if d_min <= density <= d_max:
                modulated_opacity = base_opacity
            else:
                # Decrease opacity for values outside the region
                # The exact reduction factor is not specified in sources [G]
                modulated_opacity = base_opacity * 0.5 
            
            # Standard front-to-back compositing
            alpha = modulated_opacity * (1 - accumulated_alpha)
            color = sample.color * alpha
            accumulated_color += color
            accumulated_alpha += alpha
            
            if accumulated_alpha >= 1.0:
                break
        
        image.set_pixel(x, y, accumulated_color)
    return image
```

**Parameters:**
*   $d_{min}, d_{max}$: The density bounds of the region of emphasis [S1, unofficial].
*   Opacity reduction factor: Not given in the sources.

**Edge Cases:**
*   If the region of emphasis covers the entire density range, no modulation occurs [S1, unofficial].
*   Precision issues may arise if density values are very close to $d_{min}$ or $d_{max}$ [G].

## Subtopics

### ClearView
ClearView is a variant that modulates the opacity of different segmented layers within a user-defined focus region [S2, unofficial]. It renders layers in a first step and then modulates their transparency based on importance measures [S2, unofficial]. The importance of each layer is determined by curvature information and the distance between context and focus layers [S2, unofficial]. This approach is suitable for datasets with distinct segmented structures [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples per pixel, similar to standard volume rendering [S1, unofficial].
*   **Memory Complexity:** Requires storage for the volume data and the region of emphasis parameters [S1, unofficial].
*   **Strengths:** Simple to implement; preserves the spatial structure of the volume; effective for highlighting specific density ranges [S1, unofficial].
*   **Limitations:** Relies on user knowledge of density values to define the region; may not be effective for features defined by spatial structure rather than density [S1, unofficial].
*   **When to use:** When the region of interest is defined by specific data values (e.g., temperature, density) [S1, unofficial].
*   **When not to use:** When the region of interest is defined by spatial location or complex geometric features not correlated with density [S1, unofficial].

## Background
The technique was introduced in the context of illustrative rendering of particle systems, where it was used to focus the viewer on regions of emphasis while maintaining overall volume context [S1, unofficial]. It is categorized as a focus+context technique [S1, unofficial].

## Sources
[S1] Illustrative Rendering of Particle Systems, https://doi.org/10.2312/pe.vmv.vmv13.177-185, fetched 2026-10-26 (unofficial)
[S2] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
