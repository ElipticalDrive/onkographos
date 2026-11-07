# Spherical Harmonic Volumetric Visibility

Also called: SH Volumetric Visibility

## Overview
Spherical Harmonic Volumetric Visibility is a real-time dynamic lighting technique for direct volume rendering that encodes local and global volumetric visibility using spherical harmonic basis functions [S1, unofficial]. It sets itself apart from its siblings by storing these coefficients in a multiresolution grid to handle directional lights, point lights, and environment maps efficiently [S1, unofficial].

## How it works
**Inputs:**
- Volumetric density field (voxel grid).
- Light sources: directional, point, or environment maps [S1, unofficial].
- Camera position and view parameters.

**Outputs:**
- Rendered volume image with dynamic lighting and self-shadowing [S1, unofficial].

**Data Structures:**
- **Multiresolution Grid:** Stores spherical harmonic (SH) coefficients for each voxel or grid cell [S1, unofficial].

**Steps:**
1. **Preprocessing/Encoding:** For each relevant location in the volume, compute the local and global volumetric visibility [S1, unofficial].
2. **SH Projection:** Project the computed visibility function onto spherical harmonic basis functions [S1, unofficial].
3. **Storage:** Store the resulting SH coefficients in a multiresolution grid [S1, unofficial].
4. **Rendering:** During real-time rendering, sample the SH coefficients from the multiresolution grid at the current ray-marching position and reconstruct the lighting contribution using the SH basis functions and the current view direction [S1, unofficial].

**Formulas:**
The exact mathematical formulation for the SH projection and reconstruction is not given in the sources [S1, unofficial].

**Parameters:**
- **SH Band Limit ($N$):** Not given in the sources.
- **Grid Resolution:** Not given in the sources.
- **Light Types:** Directional, point, environment maps [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Heterogeneous Density:** The method must account for varying density within the volume to compute accurate visibility [S1, unofficial].
- **Real-time Constraints:** The encoding step must be efficient enough to support dynamic scenes [S1, unofficial].
- **Precision:** SH approximation errors may occur if the band limit is too low for complex lighting scenarios [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_visibility(volume_grid, lights):
    sh_grid = {}
    for voxel in volume_grid:
        pos = voxel.position
        density = voxel.density
        # Compute visibility function for this position
        # Sources do not specify the exact integration method over the sphere [S1, unofficial]
        visibility_function = compute_visibility(pos, density, lights)
        # Project onto SH basis functions (standard SH projection) [G]
        sh_coefficients = project_to_sh(visibility_function)
        sh_grid[pos] = sh_coefficients
    return sh_grid

def render_volume(camera, volume_grid, sh_grid):
    image = []
    for pixel in camera.raster:
        ray = camera.generate_ray(pixel)
        radiance = 0.0
        while ray.has_intersection(volume_grid):
            pos = ray.position
            density = volume_grid.get_density(pos)
            sh_coefficients = sh_grid.get(pos)
            # Reconstruct lighting from SH coefficients [G]
            lighting = reconstruct_lighting(sh_coefficients, ray.direction)
            radiance += lighting * density * ray.step_size
            ray.advance()
        image.append(radiance)
    return image
```

## Subtopics

### [Real-Time Image-Based Volume Lighting](2202-real-time-image-based-volume-lighting.md)
Real-Time Image-Based Volume Lighting is a two-step, GPU-friendly technique for the real-time rendering of heterogeneous participating media under distant environment lighting. It estimates spherical scattered radiance at discrete points in the medium, projects this function into spherical harmonics (SH), and uses this cached information to compute single scattering via ray-marching. It sets itself apart from its parent [Spherical Harmonic Volumetric Visibility](1225-spherical-harmonic-volumetric-visibility.md) by explicitly splitting the scattered radiance integral into a volume-independent factor and a volume-dependent factor to reduce the computational cost of the triple product inherent in SH-based scattering to a simple dot product. Unlike [Spherical Harmonics Irradiance Encoding](4336-spherical-harmonics-irradiance-encoding.md), which typically encodes irradiance at surface points, this method encodes scattered radiance within the volume itself to handle self-shadowing and heterogeneous density.

### [Spherical Harmonics Irradiance Encoding](4336-spherical-harmonics-irradiance-encoding.md)
An approach to encode full spherical irradiance information using spherical harmonics, the most generic method to encode directional surface lighting.

### [Spherical Harmonical Exponentiation](4345-spherical-harmonical-exponentiation.md)
A method that encodes visibility in the form of spherical harmonic vectors to provide directional occlusion.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Real-time dynamic lighting support for directional, point, and environment map lights [S1, unofficial].
- **Limitations:** Approximation errors from SH basis; precomputation cost for dynamic scenes [S1, unofficial].
- **When to use:** Real-time volume rendering with dynamic lighting [S1, unofficial].
- **When not to use:** Offline rendering requiring high accuracy; static scenes where simpler methods suffice.

## Background
The method was developed by Kronander et al. in 2011 [S1, unofficial]. It builds on earlier work by Mavridis et al. [Mavridis 2010] who used voxelization for illumination, and Landis [Landis 2002] and Stewart [Stewart 2003] who developed ambient occlusion and vicinity shading techniques. The technique allows for real-time dynamic lighting in direct volume rendering, supporting directional lights, point lights, and environment maps [S1, unofficial].

## Sources
[S1] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
