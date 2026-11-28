# Volumetric Fur Rendering with LDPRT

Also called: Lengyel et al. fur with PRT, Microgeometry PRT, PRT Shell Textures, Volumetric Fur with PRT

## Overview
This method renders realistic microgeometry (specifically fur) with global lighting effects, such as self-shadowing, at real-time frame rates by combining a volumetric fur model with Local, Deformable Precomputed Radiance Transfer (LDPRT) [S1, unofficial]. It solves the problem of capturing microgeometric self-shadowing in a volumetric fur model by precomputing light attenuation to each voxel in the fur volume and rendering the volume using shell textures [S1, unofficial]. Unlike general participating media methods, it specifically models the microgeometry of fur hairs using a multi-layered shell approach and defers the conversion of illuminance to luminous exitance to the reconstruction phase, allowing for heterogeneous lighting models [S1, unofficial].

## How it works
### Inputs and Outputs
*   **Inputs:** Target surface geometry, light direction vector $\mathbf{d}$, fur parameters (base width ranges, falloff rates, curl trajectories), and voxel grid resolution [S1, unofficial].
*   **Outputs:** A rendered image of the fur volume with self-shadowing effects [S1, unofficial].

### Data Structures
*   **Volumetric Fur Model:** A 3D grid (voxel volume) representing the fur, constructed by sampling the target geometry using a 3D box filter [S1, unofficial].
*   **Shell Textures:** The volume is stored and rendered using concentric 3D rings or shells, which are transparent everywhere except at the intersection of the shell and the microgeometry [S1, unofficial].
*   **Illumination Data:** Precomputed attenuation values stored per voxel [S1, unofficial].

### Steps
1.  **Create a realistic volumetric fur model:** Sample the target geometry with a 3D box filter to define the base of the fur volume [S1, unofficial]. Model hairs with random base width, linear falloff, and parabolic curl trajectory [S1, unofficial]. Implement two layers of fur: undercoat (short, dense, small base width) and overcoat (longer, thicker, larger base width) [S1, unofficial]. Antialias by approximating a box filter for each voxel the hair passes through: slice the voxel with horizontal planes, calculate the intersecting hair area for each plane, and average the results [S1, unofficial]. Restrict hair width to at most 1 pixel to limit intersections to 1, 2, 3, or 4 adjacent pixels per level, computing areas using circle, wedge, and chord formulas [S1, unofficial].
2.  **Precompute illumination of the fur volume:** Calculate light attenuation to each voxel in the fur volume, corresponding to Step 1 of Kajiya's method [S1, unofficial]. Ignore scattering and microgeometry lighting models (Steps 2 and 3 of Kajiya's method are deferred or omitted) [S1, unofficial]. Assume the light is at infinity in the direction defined by vector $\mathbf{d}$ [S1, unofficial]. For each voxel, march a ray from the center in direction $\mathbf{d}$ until it leaves the volume, adding the density of each voxel hit scaled by the length of the ray that passed through it [S1, unofficial]. The transparency $T$ is calculated using the texel version of the attenuation formula: $T = e^{-r \sum_{t_{start}}^{t_{end}} \rho(\mathbf{e} + t\mathbf{d})}$, where $T$ is transparency, $r$ is the density factor, $\rho$ is the 3D density function, $\mathbf{e}$ is the ray origin (center of the voxel), $\mathbf{d}$ is the light direction vector, $t$ is distance, and $t_{start}$ and $t_{end}$ are the ray bounds within the volume [S1, unofficial]. The sum replaces the integral to account for the discrete nature of the texel model, ensuring non-zero contributions from surface intersections [S1, unofficial].
3.  **Render the volume:** Render the volume using alpha blending on graphics hardware, corresponding to Step 3 of Kajiya's method (attenuation from surface to eye) [S1, unofficial]. Use the shell textures defined in Step 1 [S1, unofficial].
4.  **Reconstruct the lighting:** Reconstruct the final lighting by applying the precomputed attenuation to the surface lighting [S1, unofficial]. The conversion of illuminance to luminous exitance (Step 2 of Kajiya's method) is pushed back to this phase, allowing for heterogeneous lighting models that vary over position and/or time, although the basic implementation simply displays the light reaching each voxel [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def create_fur_volume(geometry, params):
    """Step 1: Create a realistic volumetric fur model."""
    # Initialize 3D density grid
    volume = [[[0.0 for _ in range(geometry.z)] for _ in range(geometry.y)] for _ in range(geometry.x)]
    # [G] Sample target geometry surface points using a 3D box filter
    base_points = [geometry.sample_point(i) for i in range(geometry.num_points)]
    for point in base_points:
        # [G] Classify point into undercoat (short, dense, small width) or overcoat (long, thick, large width)
        layer = classify_layer(point)
        base_width = random.uniform(layer.width_min, layer.width_max)
        # Parabolic trajectory with random direction
        trajectory = generate_parabolic_curl(point, layer.falloff)
        for t in range(0, layer.max_length, params.step):
            pos = trajectory(t)
            # Linear falloff
            width_at_t = base_width * (1.0 - t / layer.max_length)
            # Restrict width to at most 1 pixel equivalent
            width_at_t = min(width_at_t, params.max_pixel_width)
            # [G] Map 3D position to voxel indices
            vx, vy, vz = int(pos[0]), int(pos[1]), int(pos[2])
            if 0 <= vx < geometry.x and 0 <= vy < geometry.y and 0 <= vz < geometry.z:
                # Antialiasing: slice voxel with horizontal planes
                planes = slice_voxel(0, 0, 0, params.antialias_slices)
                total_area = 0.0
                for plane in planes:
                    # [G] Compute intersection area using circle, wedge, and chord formulas
                    area = calculate_intersection_area(width_at_t, plane)
                    total_area += area
                volume[vx][vy][vz] = total_area / len(planes)
    return volume

def precompute_illumination(volume, light_direction, density_factor, params):
    """Step 2: Precompute illumination of the fur volume."""
    illumination = [[[0.0 for _ in range(len(volume[0][0]))] for _ in range(len(volume[0]))] for _ in range(len(volume))]
    for x in range(len(volume)):
        for y in range(len(volume[0])):
            for z in range(len(volume[0][0])):
                ray_origin = [x, y, z]
                density_sum = 0.0
                t = 0.0
                # March ray from voxel center in light direction until leaving volume
                while t < params.max_ray_length:
                    current_pos = [ray_origin[i] + light_direction[i] * t for i in range(3)]
                    # [G] Check bounds and compute ray length inside current voxel
                    if 0 <= current_pos[0] < len(volume) and 0 <= current_pos[1] < len(volume[0]) and 0 <= current_pos[2] < len(volume[0][0]):
                        ray_length = compute_ray_box_intersection_length(ray_origin, light_direction, current_pos)
                        density_sum += volume[int(current_pos[0])][int(current_pos[1])][int(current_pos[2])] * ray_length
                    t += params.step_size
                # Texel attenuation formula: T = e^{-r * sum(rho * dt)}
                transparency = math.exp(-density_factor * density_sum)
                illumination[x][y][z] = transparency
    return illumination

def render_and_reconstruct(volume, illumination, camera, surface_lighting):
    """Steps 3 & 4: Render the volume and reconstruct lighting."""
    image = empty_image(camera.resolution)
    # Render shells back-to-front for correct alpha blending
    for shell in sorted(volume.shells, key=lambda s: s.depth, reverse=True):
        shell_image = render_shell_geometry(shell, camera)
        # Step 3: Attenuation from surface to eye via alpha blending
        # Step 4: Reconstruct lighting (deferred illuminance to luminous exitance)
        # Basic implementation: apply precomputed transparency to shell color
        final_shell = apply_illumination(shell_image, illumination, shell)
        image = alpha_blend(image, final_shell)
    return image
```

### Parameters and Edge Cases
*   **Hair Width:** Restricted to at most 1 pixel to simplify intersection calculations to 1, 2, 3, or 4 adjacent pixels [S1, unofficial].
*   **Density Factor ($r$):** A scaling factor for the density function [S1, unofficial].
*   **Scattering:** The method currently ignores scattering. Adding a scattering term could increase realism [S1, unofficial].
*   **Lighting Model:** The current implementation does not take advantage of heterogeneous lighting models, simply displaying light reaching each voxel [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Performance:** Capable of real-time frame rates for microgeometry rendering [S1, unofficial].
*   **Strengths:** Captures microgeometric self-shadowing; real-time performance; supports heterogeneous lighting models in reconstruction [S1, unofficial].
*   **Limitations:** Ignores scattering; implementation may not fully exploit heterogeneous lighting; restricted hair width for antialiasing simplification [S1, unofficial].

## Background
The method is based on the work of Lengyel et al. [2001] for real-time fur rendering and combines it with Local, Deformable Precomputed Radiance Transfer (LDPRT) [S1, unofficial]. It builds on the lighting model for hair developed by Kajiya et al. [1989], specifically adapting their three-step process (attenuation to volume, microgeometry lighting, attenuation to eye) by precomputing only the first step and deferring the others [S1, unofficial]. The use of shell textures for microgeometry rendering is also derived from Lengyel et al. [S1, unofficial].

## Sources
[S1] A Volumetric Approach to Rendering Microgeometry Using PRT, https://www.cs.umbc.edu/~ugrad/honors/theses/JohnKloetzli.pdf, fetched 2026-10-28 (unofficial)
