# Volume Absorption
Also called: Absorption Rendering
## Overview
Volume Absorption simulates the attenuation of light as it passes through a participating medium, blocking a portion of the light and allowing the rest to pass straight through [S1]. It is primarily used to shade materials such as black smoke, colored glass, or to combine with Volume Scatter for complex media [S1]. Unlike [Volumetric Scattering Model](11410-volumetric-scattering-model.md), which redirects light, absorption simply reduces intensity based on optical depth [S1].
## How it works
### Inputs and Outputs
**Inputs:**
- Incident radiance ($L_{in}$): The light arriving at the volume entry point.
- Absorption coefficient ($\sigma_a$): The rate at which light is absorbed per unit distance. In practical shading nodes, this is often derived from a color map or material property [S2, unofficial].
- Path length ($d$): The distance light travels through the volume.

**Outputs:**
- Transmitted radiance ($L_{out}$): The light exiting the volume after absorption.

### Data Structures
Not given in the sources.

### Steps
1. Calculate the optical depth ($\tau$) along the path:
   $$ \tau = \sigma_a \cdot d $$
2. Compute the transmittance ($T$) using the Beer-Lambert law:
   $$ T = e^{-\tau} $$
3. Calculate the output radiance:
   $$ L_{out} = L_{in} \cdot T $$

### Parameters
- **Color Map**: In visualization contexts (e.g., medical imaging), a `color_map` maps volume scalar values to colors for translucent rendering [S2, unofficial].
- **Absorption Coefficient**: Typically defined per color channel (R, G, B) to simulate colored absorption (e.g., colored glass) [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_absorption(L_in, sigma_a, distance):
    """
    Calculate light transmission through an absorbing volume.
    
    Args:
        L_in: Incident radiance (vector or scalar).
        sigma_a: Absorption coefficient (vector or scalar).
        distance: Path length through the volume.
        
    Returns:
        L_out: Transmitted radiance.
    """
    # Calculate optical depth
    tau = sigma_a * distance
    
    # Calculate transmittance using Beer-Lambert law
    # Note: In practice, clamp tau to avoid underflow if necessary, 
    # though standard float precision usually handles large tau well.
    transmittance = exp(-tau)
    
    # Apply attenuation
    L_out = L_in * transmittance
    
    return L_out
```

### Edge Cases and Pitfalls
- **Colormap Switching**: When switching to absorption rendering mode, especially after adjusting colormaps, the render panel may turn white with only a faint gray outline remaining [S2, unofficial]. This behavior can persist even if `isorange` or `isovalue` are tweaked [S2, unofficial].
- **Noise Sensitivity**: Absorption rendering is more robust to pixel-by-pixel noise than isosurface rendering [S2, unofficial].
- **Default State**: If the color map becomes "out of whack" due to experimentation, there may be no obvious UI button to restore defaults, requiring manual reset [S2, unofficial].

## Subtopics

### Absorption Rendering
A visualization mode that simulates light absorption through the volume to produce a translucent visualization [S2, unofficial]. It uses a `color_map` to map volume scalar values to colors [S2, unofficial]. It is preferred over isosurface rendering when images contain significant noise, as it is less sensitive to pixel-by-pixel noise [S2, unofficial]. However, it can exhibit unexpected UI behavior (white panel) when switching modes or adjusting colormaps [S2, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Strengths**: Robust to noise compared to isosurface rendering [S2, unofficial]. Simple to implement for translucent materials like glass or smoke [S1].
- **Limitations**: Can produce unintuitive visual results (e.g., white panel) if parameters like colormaps are not managed correctly [S2, unofficial]. Does not account for scattering, so it cannot render volumetric effects like fog or clouds alone [S1].
- **When to use**: Shading black smoke, colored glass, or as a component mixed with scattering [S1]. Visualizing noisy scalar fields where isosurfaces fail [S2, unofficial].
- **When not to use**: When scattering effects (light bouncing within the volume) are required [S1].

## Background
Volume Absorption is a fundamental component of volume shading, often implemented as a node in 3D rendering software like Blender [S1]. It is conceptually similar to the transparent BSDF node in surface shading, blocking part of the light while letting the rest pass through [S1]. In scientific visualization, it is used for translucent rendering of volumetric data [S2, unofficial].

## Sources
[S1] Volumes — Blender Manual, https://www.blender.org/manual/nb/3.6/render/materials/components/volume.html, fetched 2026-10-26
[S2] volume.nii, https://github.com/SimonDanisch/GLPlot.jl/issues/40, fetched 2026-10-27 (unofficial)
