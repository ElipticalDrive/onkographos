# Sky Atmosphere
Also called: Rayleigh + Mie + ozone, physically based sky

## Overview
An Earth-like atmosphere model that bakes single-scattered sunlight (Rayleigh, Mie, and ozone scattering) into an equirectangular map sampled by `environment_light`, with sun irradiance attenuated by air along its direction [S1, unofficial]. It differs from its parent [Sky Radiance Model](3256-sky-radiance-model.md) by explicitly modeling the spectral attenuation and scattering physics of an Earth-like atmosphere rather than relying solely on a precomputed view texture and simple sun disk addition [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Sun direction vector, atmospheric parameters (Rayleigh, Mie, ozone coefficients), and the geometry of the atmosphere (implied Earth-like spherical shell).
*   **Output:** An equirectangular environment map containing the baked sky radiance.

**Data Structures:**
*   **Equirectangular Map:** A 2D texture mapping longitude to the x-axis and latitude to the y-axis. The resolution is not specified in the sources [S1, unofficial].
*   **Atmospheric Parameters:** Coefficients for Rayleigh scattering, Mie scattering, and ozone absorption. The specific bit-packing or storage format is not detailed, though the implementation notes mention "one byte per cell" in a different context (height fog) [S1, unofficial].

**Steps:**
1.  **Calculate Single-Scattered Sunlight:** Compute the contributions from Rayleigh, Mie, and ozone scattering for sunlight interacting with an Earth-like atmosphere [S1, unofficial].
2.  **Bake Contributions:** Store these computed radiance values into an equirectangular map [S1, unofficial].
3.  **Sample Map:** Use an `environment_light` sampler to retrieve sky radiance based on the ray direction [S1, unofficial].
4.  **Attenuate Sun Irradiance:** Modulate the sun's direct irradiance by the optical depth of the air along the sun's direction. This causes low sun angles to appear redder due to increased path length through the atmosphere [S1, unofficial].

**Formulas and Parameters:**
*   The sources do not provide the explicit mathematical formulas for the Rayleigh, Mie, or ozone scattering coefficients or the integration over the atmospheric shell [S1, unofficial].
*   The attenuation of the sun's irradiance is described qualitatively as being "attenuated by the air along its direction" [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Low Sun Angles:** The method specifically handles the reddening effect of low suns by attenuating irradiance based on the air path [S1, unofficial].
*   **Precision:** No specific precision requirements are given for the baking process [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bake_sky_atmosphere(sun_direction, atmosphere_params):
    """
    Bakes single-scattered sunlight into an equirectangular map.
    
    Args:
        sun_direction: Vector3, direction of the sun.
        atmosphere_params: Object containing Rayleigh, Mie, and ozone coefficients.
        
    Returns:
        Texture2D: Equirectangular map of sky radiance.
    """
    # Resolution of the equirectangular map is not specified in sources [S1, unofficial]
    width = 2048
    height = 1024
    sky_map = empty_texture(width, height)
    
    # Iterate over all directions in the equirectangular map
    for y in range(height):
        for x in range(width):
            # Convert pixel coordinates to spherical coordinates
            # Longitude: 0 to 2*pi
            # Latitude: -pi/2 to pi/2
            longitude = (x / width) * 2 * PI
            latitude = (y / height) * PI - PI / 2
            
            # Convert to Cartesian direction vector
            dir_vec = spherical_to_cartesian(longitude, latitude)
            
            # Calculate single-scattered sunlight contributions
            # This involves Rayleigh, Mie, and ozone scattering [S1, unofficial]
            # The exact integration over the atmosphere is not provided in sources [S1, unofficial]
            radiance = calculate_single_scattering(dir_vec, sun_direction, atmosphere_params)
            
            # Store in the equirectangular map
            sky_map.set_pixel(x, y, radiance)
            
    return sky_map

def sample_sky_atmosphere(ray_direction, sky_map, sun_direction, atmosphere_params):
    """
    Samples the baked sky atmosphere for a given ray direction.
    
    Args:
        ray_direction: Vector3, direction of the ray.
        sky_map: Texture2D, the baked equirectangular map.
        sun_direction: Vector3, direction of the sun.
        atmosphere_params: Object containing atmospheric coefficients.
        
    Returns:
        Color3: The sky radiance for the given direction.
    """
    # Convert ray direction to equirectangular coordinates
    longitude, latitude = cartesian_to_spherical(ray_direction)
    
    # Map to texture coordinates
    x = int((longitude / (2 * PI)) * sky_map.width)
    y = int(((latitude + PI / 2) / PI) * sky_map.height)
    
    # Sample the baked sky radiance
    sky_radiance = sky_map.get_pixel(x, y)
    
    # Attenuate sun irradiance by air along its direction
    # This causes low suns to turn orange [S1, unofficial]
    sun_attenuation = calculate_sun_attenuation(sun_direction, atmosphere_params)
    
    # Combine sky radiance and attenuated sun contribution
    # The exact combination formula is not specified in sources [S1, unofficial]
    final_radiance = sky_radiance + sun_attenuation
    
    return final_radiance

def calculate_sun_attenuation(sun_direction, atmosphere_params):
    """
    Calculates the attenuation of the sun's irradiance due to atmospheric scattering.
    
    Args:
        sun_direction: Vector3, direction of the sun.
        atmosphere_params: Object containing atmospheric coefficients.
        
    Returns:
        Color3: The attenuated sun irradiance.
    """
    # The exact formula for attenuation is not provided in sources [S1, unofficial]
    # It depends on the optical depth along the sun's direction
    return Color3(1.0, 1.0, 1.0) # Placeholder [G]
```

## Subtopics

### A-cube Model
A general illumination model for atmospheric effects (blue skies, sunsets, rainbows, shafts of light) based on the A-cube, modeling the atmosphere by particles of different sizes and using volume sampling techniques [S2]. The A-cube or atmospheric cube limits the modeling space to a finite volume [S2]. The atmospheric model handles absorption, scattering (Rayleigh and Mie) and geometric optics (refractive dispersion) but operating in the RGB colour model [S2]. Density and phase functions of the scattering particles inside the cube can vary from point to point [S2]. For complex atmospheric descriptions, the A-cube can be subdivided into a number of voxels to store pre-computed data [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The baking step is offline, and the sampling step is O(1) per ray if the map is precomputed [S1, unofficial].
*   **Memory Complexity:** Depends on the resolution of the equirectangular map. The sources do not specify the resolution or memory footprint [S1, unofficial].
*   **Strengths:** Physically based representation of Earth-like sky colors and sun attenuation [S1, unofficial].
*   **Limitations:** Limited to single scattering and Earth-like atmospheres [S1, unofficial]. Does not handle complex atmospheric phenomena like rainbows or shafts of light natively (unlike the A-cube model) [S2].
*   **When to use:** When a physically based, Earth-like sky is required with minimal runtime cost.
*   **When not to use:** When complex atmospheric effects like rainbows or non-Earth-like atmospheres are needed.

## Background
The method is associated with the implementation of MagicaVoxel, specifically in the context of adding a physically based sky with Rayleigh, Mie, and ozone scattering [S1, unofficial]. The A-cube model was introduced by Inakage as a general illumination model for atmospheric effects [S2].

## Sources
[S1] Milestone 13: MagicaVoxel look (.vox, DDA voxel grid, sun/sky, bloom, style voxel), https://github.com/adriendjn/mypbr/pull/10, fetched 2026-10-25 (unofficial)
[S2] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
