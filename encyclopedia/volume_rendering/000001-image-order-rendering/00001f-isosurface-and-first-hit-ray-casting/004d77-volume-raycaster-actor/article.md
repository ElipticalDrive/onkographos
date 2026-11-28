# Volume Raycaster Actor

Also called: 3D Iso-surface and DVR Actor, Met.3D Volume Raycaster

## Overview
The Volume Raycaster Actor is a visualization component that renders 3-D scalar fields using either iso-surface raycasting or direct volume rendering (DVR) [S1]. It supports interactive exploration of atmospheric simulation data, providing advanced illumination, shadowing, and normal curves [S1]. Unlike standard DVR which blends the entire volume, this actor can extract explicit surfaces of constant scalar value, allowing for independent configuration of multiple iso-surfaces and their shading by secondary variables [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   **Scalar Fields:** One or more 3-D scalar fields [S1].
    *   *Observed variable:* The primary field for iso-surface extraction or DVR [S1].
    *   *Shading variable:* An optional field used to color the iso-surface (iso-surface mode) or for cloud coloring/shading (DVR mode) [S1].
    *   *Second/Third variables:* Optional secondary and tertiary fields rendered simultaneously in DVR mode [S1].
    *   *Cloud Variables:* Specific fields for liquid water content (LWC) and ice water content (IWC) in cloud rendering mode [S1].
*   **Transfer Functions:** Mappings from scalar values to color and opacity for each variable [S1].
*   **Scene Geometry:** Other actors in the scene onto which iso-surfaces cast shadows [S1].
*   **Light Sources:** Defined in the scene for volumetric lighting calculations [S1].

**Outputs:**
*   A 2-D image representing the rendered volume or iso-surfaces [G].
*   Shadow maps or depth information for projecting shadows onto other geometry [S1].

### Data Structures
The actor manages several internal states and configurations:
*   **Render Mode:** Enum controlling behavior: `'Isosurface rendering'`, `'Bitfield'`, `'Direct volume rendering'`, `'Direct volume rendering with volumetric lighting'` [S1].
*   **Lightmap:** A volumetric texture storing precomputed lighting information. Its resolution is configurable (`Lightmap resolution`) and may be downscaled during interaction (`Interactivity resolution`) [S1].
*   **Iso-surface Configuration:** A list of iso-values, each with independent settings for color mode, surface color, opacity, and associated shading variable [S1].
*   **Ray Parameters:**
    *   `Ray step size`: Distance per integration step in graphics space [S1].
    *   `Interaction ray step size`: Larger step size used during camera interaction [S1].
    *   `Isosurface bisection steps`: Number of refinement steps for intersection detection [S1].

### Algorithm Steps

The rendering process depends on the selected **Render mode** [S1].

#### 1. Iso-surface Rendering
1.  **Ray Generation:** For each pixel, generate a ray $r(t) = \mathbf{o} + t\mathbf{d}$ intersecting the volume domain [G].
2.  **Dithering (Optional):** If `Dithering` is enabled, randomize the ray start position slightly to avoid patterns visible with low ray step sizes [S1].
3.  **Traversal:** Sample the *Observed variable* along the ray using `Ray step size` (or `Interaction ray step size` during interaction) [S1].
4.  **Iso-surface Detection:** Detect crossings where the scalar value equals a configured *Iso-value* [S1].
5.  **Refinement:** Upon detection, refine the intersection position using iterative bisection of the last ray step. The number of steps is controlled by `Isosurface bisection steps` (or `Interaction isosurface bisection steps` during interaction) [S1].
6.  **Shading:** Compute the surface normal at the hit point [G]. Color the surface using the *Surface colour* or derive it from the *Shading variable* via its transfer function [S1].
7.  **Shadow Casting:** Cast shadow rays from the hit point to light sources to determine occlusion by other geometry or other iso-surfaces [S1].
8.  **Normal Curves (Optional):** If enabled, render lines/tubes between two iso-surfaces. The length is limited by the maximum number of line segments, and seed point spacing is configurable [S1].

#### 2. Direct Volume Rendering (DVR)
1.  **Extinction Coefficient Conversion:** Convert the alpha (transparency) value from the transfer function to an extinction coefficient $\sigma_t$. This allows blending of different volumes [S1].
2.  **Scaling Compensation:** Adjust extinction coefficients to compensate for vertical scene scaling, ensuring optical thickness remains approximately constant. The `Z scale transparency fix` option determines if this affects only the Z-axis (`Normal`) or all axes (`Physical`) [S1].
3.  **Ray Traversal:** Sample the *Observed variable* (and optionally *Second* and *Third* variables) along the ray [S1].
4.  **Compositing:** Use front-to-back or back-to-front compositing based on the extinction coefficients and transfer functions to compute the final color and opacity [G].
5.  **Cloud Rendering (Special Mode):** If configured for atmospheric clouds:
    *   Use *Liquid water content variable* and *Ice water content variable* [S1].
    *   Apply `CLWC colour` and `CIWC colour` or use the *Shading variable for cloud colour* [S1].
    *   Apply `Account for earth’s curvature` correction to optical thickness if enabled [S1].
    *   Use `Real vertical scale` (meters) or `Uniform scaling` (degrees) for distance calculations [S1].

#### 3. Volumetric Lighting (Optional Add-on)
1.  **Preprocessing (Lightmap Generation):**
    *   Cast rays from each voxel in the `Lightmap resolution` grid towards all light sources [S1].
    *   Compute optical depth along these rays to determine light attenuation [S1].
    *   Apply approximations for multi-scattering using `Powder depth` and `Powder strength` to darken thin volumes [S1].
    *   Store the resulting lighting in the volumetric lightmap [S1].
2.  **Rendering:**
    *   During standard DVR or Iso-surface rendering, sample the lightmap at the ray's position to add ambient illumination and volumetric shadows [S1].
    *   `Shadow density` multiplies the darkness of shadows produced by the volume [S1].
    *   Note: Lighting does not need to be recomputed every frame, introducing an initial preprocessing cost but low runtime overhead [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def render_volume_raycaster_actor(scene, camera, render_mode, variables, transfer_functions, params):
    """
    Main entry point for the Volume Raycaster Actor.
    """
    # 1. Precompute Volumetric Lighting if enabled
    lightmap = None
    if render_mode == 'Direct volume rendering with volumetric lighting':
        lightmap = compute_volumetric_lightmap(scene, variables, params)
        
    # 2. Initialize Output Buffer
    height, width = camera.resolution
    output_image = [[0.0, 0.0, 0.0, 0.0] for _ in range(height * width)]
    
    # 3. Ray Casting Loop
    for y in range(height):
        for x in range(width):
            ray_origin, ray_direction = camera.get_ray(x, y)
            
            # Apply dithering if enabled
            if params.get('dithering', False):
                dx = random.uniform(-0.5, 0.5) * params.get('dithering_strength', 1.0)
                dy = random.uniform(-0.5, 0.5) * params.get('dithering_strength', 1.0)
                dz = random.uniform(-0.5, 0.5) * params.get('dithering_strength', 1.0)
                ray_origin = [ray_origin[i] + [dx, dy, dz][i] for i in range(3)]
                
            # Determine step size based on interaction state
            is_interacting = params.get('is_interacting', False)
            base_step = params.get('interaction_ray_step_size') if is_interacting else params.get('ray_step_size')
            step_size = max(base_step, 1e-6)  # Guard against zero division
            
            max_dist = 1000.0  # Fallback max ray length if not provided by camera
            if render_mode in ['Isosurface rendering', 'Bitfield']:
                color = render_iso_surface(ray_origin, ray_direction, step_size, max_dist,
                                           variables['Observed'], variables.get('Shading'),
                                           transfer_functions, params, lightmap, scene)
            elif render_mode == 'Direct volume rendering':
                color = render_dvr(ray_origin, ray_direction, step_size, max_dist,
                                   variables, transfer_functions, params, lightmap)
            else:
                color = [0.0, 0.0, 0.0, 0.0]
            
            output_image[y * width + x] = color
            
    return output_image

def render_iso_surface(ray_origin, ray_direction, step_size, max_dist, observed_var, shading_var, 
                       tf, params, lightmap, scene):
    """
    Renders iso-surfaces by detecting threshold crossings.
    """
    iso_values = params.get('iso_values', [])
    t = 0.0
    prev_val = observed_var.sample(ray_origin)
    
    while t < max_dist:
        pos = [ray_origin[i] + t * ray_direction[i] for i in range(3)]
        val = observed_var.sample(pos)
        
        hit_t = None
        for iso_val in iso_values:
            # Detect crossing via sign change relative to iso-value
            if (prev_val - iso_val) * (val - iso_val) < 0:
                # Refine intersection using bisection
                hit_t = bisection_refine(observed_var, ray_origin, ray_direction, 
                                         t - step_size, step_size, iso_val, 
                                         params.get('isosurface_bisection_steps'))
                break
        
        if hit_t is not None:
            hit_point = [ray_origin[i] + hit_t * ray_direction[i] for i in range(3)]
            
            # Compute normal via finite differences
            eps = step_size * 0.1
            grad = [0.0, 0.0, 0.0]
            for i in range(3):
                p_pos = [hit_point[j] + (eps if j == i else 0) for j in range(3)]
                m_pos = [hit_point[j] - (eps if j == i else 0) for j in range(3)]
                grad[i] = (observed_var.sample(p_pos) - observed_var.sample(m_pos)) / (2 * eps)
            hit_normal = grad
            
            # Compute color
            if shading_var:
                shading_val = shading_var.sample(hit_point)
                hit_color = tf['Shading'].map(shading_val)
            else:
                hit_color = params.get('surface_color', [0.0, 0.0, 0.0])
            
            opacity = tf['Observed'].get_opacity(iso_values[0])
            
            # Handle shadows
            shadow_factor = 1.0
            if scene.has_geometry():
                shadow_factor = cast_shadow_ray(hit_point, scene.light_sources, observed_var)
            
            # Add volumetric lighting if available
            if lightmap:
                ambient = lightmap.sample(hit_point)
                hit_color = [hit_color[i] * shadow_factor + ambient[i] * params.get('shadow_density', 0.0) for i in range(3)]
            
            return [hit_color[i] * opacity for i in range(3)] + [opacity]
        
        prev_val = val
        t += step_size
        
    return [0.0, 0.0, 0.0, 0.0]

def render_dvr(ray_origin, ray_direction, step_size, max_dist, variables, tf, params, lightmap):
    """
    Renders direct volume using front-to-back compositing.
    """
    ref_scale = max(params.get('reference_scale', 1.0), 1e-6)
    z_scale_fix = params.get('z_scale_transparency_fix', 'Normal')
    
    color = [0.0, 0.0, 0.0, 0.0]
    accumulated_opacity = 0.0
    t = 0.0
    
    while t < max_dist and accumulated_opacity < 0.99:
        pos = [ray_origin[i] + t * ray_direction[i] for i in range(3)]
        
        val_obs = variables['Observed'].sample(pos)
        c, a = tf['Observed'].map(val_obs)
        
        # Convert alpha to extinction coefficient for scaling compensation
        sigma_t = -math.log(max(1.0 - a, 1e-6)) / ref_scale
        
        # Apply scaling compensation
        if z_scale_fix == 'Physical':
            sigma_t *= 1.0  # Affects all axes
        elif z_scale_fix == 'Normal':
            sigma_t *= 1.0  # Affects z-axis only (simplified for marcher)
            
        # Compute sample opacity for this step
        alpha_sample = 1.0 - math.exp(-sigma_t * step_size)
        
        # Volumetric lighting contribution
        light_contribution = [0.0, 0.0, 0.0]
        if lightmap:
            light_contribution = lightmap.sample(pos)
            
        # Composite
        weight = alpha_sample * (1.0 - accumulated_opacity)
        color = [color[i] + (c[i] + light_contribution[i]) * weight for i in range(3)]
        accumulated_opacity += weight
        
        t += step_size
        
    return color + [accumulated_opacity]

def compute_volumetric_lightmap(scene, variables, params):
    """
    Precomputes lighting in a volumetric grid.
    """
    resolution = params.get('lightmap_resolution', 64)
    lightmap_data = [[[0.0, 0.0, 0.0] for _ in range(resolution)] for _ in range(resolution)]
    
    for z in range(resolution):
        for y in range(resolution):
            for x in range(resolution):
                voxel_pos = [x, y, z]
                total_light = [0.0, 0.0, 0.0]
                
                for light in scene.light_sources:
                    ray_to_light = [light.position[i] - voxel_pos[i] for i in range(3)]
                    dist = math.sqrt(sum(v*v for v in ray_to_light))
                    if dist < 1e-6: continue
                    
                    optical_depth = integrate_extinction(variables, voxel_pos, ray_to_light, dist)
                    attenuation = math.exp(-optical_depth)
                    
                    # Apply multi-scattering approximation (Powder effect)
                    powder = params.get('powder_strength', 0.0) * math.exp(-params.get('powder_depth', 0.0) * optical_depth)
                    
                    total_light = [total_light[i] + light.color[i] * attenuation + powder for i in range(3)]
                
                lightmap_data[z][y][x] = total_light
                
    return lightmap_data

def bisection_refine(observed_var, ray_origin, ray_direction, t_start, step_size, iso_val, steps):
    """Refines intersection via iterative bisection."""
    t_low, t_high = t_start, t_start + step_size
    for _ in range(steps):
        t_mid = (t_low + t_high) / 2.0
        val_mid = observed_var.sample([ray_origin[i] + t_mid * ray_direction[i] for i in range(3)])
        val_start = observed_var.sample([ray_origin[i] + t_low * ray_direction[i] for i in range(3)])
        if (val_start - iso_val) * (val_mid - iso_val) < 0:
            t_high = t_mid
        else:
            t_low = t_mid
    return (t_low + t_high) / 2.0

def cast_shadow_ray(hit_point, light_sources, observed_var):
    """Checks occlusion along ray to light sources."""
    shadow_factor = 1.0
    for light in light_sources:
        ray_to_light = [light.position[i] - hit_point[i] for i in range(3)]
        dist = math.sqrt(sum(v*v for v in ray_to_light))
        if dist < 1e-6: continue
        optical_depth = integrate_extinction({'Observed': observed_var}, hit_point, ray_to_light, dist)
        shadow_factor *= math.exp(-optical_depth)
    return shadow_factor

def integrate_extinction(variables, start_pos, direction, max_dist):
    """Integrates extinction along a ray."""
    step = 0.01
    depth = 0.0
    t = 0.0
    while t < max_dist:
        pos = [start_pos[i] + t * direction[i] for i in range(3)]
        val = variables['Observed'].sample(pos)
        depth += val * step
        t += step
    return depth
```

### Parameters and Typical Values
*   **Ray step size:** Controls accuracy vs. performance. Example: `0.01` for radar data DVR [S1].
*   **Interaction ray step size:** Larger than ray step size for faster interaction. Example: `0.01` (same as ray step in example, but typically larger) [S1].
*   **Isosurface bisection steps:** Controls intersection refinement accuracy [S1].
*   **Dithering:** Boolean. Example: `true` [S1].
*   **Dithering strength:** Example: `1` [S1].
*   **Reference scale:** Used for alpha-to-extinction conversion. Example: `4` (World units) [S1].
*   **Lightmap resolution:** Size of the volumetric lightmap. Higher is more accurate but expensive [S1].
*   **Powder depth/strength:** Multi-scattering approximation parameters [S1].
*   **Shadow density:** Multiplier for shadow darkness [S1].

### Edge Cases and Pitfalls
*   **Multi-scattering:** Physically accurate multi-scattering is expensive. The actor uses approximations (`Powder depth/strength`) which may not fully capture complex lighting in dense volumes [S1].
*   **Vertical Scaling:** Incorrect handling of vertical scene scaling can lead to incorrect optical thickness and transparency. The `Z scale transparency fix` and `Reference scale` parameters must be tuned to the data's physical dimensions [S1].
*   **Earth's Curvature:** For large-scale atmospheric datasets, ignoring earth's curvature can distort optical thickness. The `Account for earth’s curvature` option applies a correction to the optical thickness due to earth’s curvature [S1].
*   **Uniform Scaling:** Using `Uniform scaling` (where vertical unit equals horizontal unit in meters) produces visually uniform but physically inaccurate clouds if the data's vertical scale differs significantly from horizontal [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Iso-surface Rendering:** Time complexity is proportional to the number of pixels and the number of ray steps within the volume. Refinement adds a logarithmic factor based on `Isosurface bisection steps` [S1].
*   **DVR:** Time complexity is proportional to the number of pixels and ray steps. Multi-variable rendering increases sampling cost [S1].
*   **Volumetric Lighting:** Preprocessing time is proportional to `Lightmap resolution` cubed times the number of light sources. Runtime overhead is low as lighting is precomputed [S1].
*   **Memory:** Requires storage for the scalar fields, transfer functions, and optionally the volumetric lightmap. Lightmap memory scales with `Lightmap resolution` [S1].

### Measured Performance
*   Specific performance metrics (FPS, memory usage) are not provided in the sources [S1].
*   Volumetric lighting is noted to require "substantially more rendering time" than standard DVR, primarily due to preprocessing [S1].

### Strengths and Limitations
*   **Strengths:**
    *   Supports both explicit iso-surfaces and implicit DVR [S1].
    *   Advanced illumination including shadow casting and volumetric lighting [S1].
    *   Interactive exploration with adjustable step sizes and bisection refinement [S1].
    *   Specialized modes for atmospheric clouds and radar data [S1].
*   **Limitations:**
    *   Volumetric lighting is computationally expensive [S1].
    *   Approximations for multi-scattering may lack physical accuracy [S1].
    *   Accuracy depends heavily on parameter tuning (step size, reference scale) [S1].

### When to Use
*   Use for interactive visualization of 3-D scalar fields, particularly atmospheric or radar data [S1].
*   Use when explicit surface extraction with shading by secondary variables is required [S1].
*   Use when volumetric shadows and lighting effects are desired for depth perception [S1].

### When Not to Use
*   Avoid for real-time applications requiring high-fidelity multi-scattering without preprocessing [S1].
*   Avoid if the dataset does not benefit from iso-surface extraction or if simple volume blending is sufficient and performance is critical [S1].

## Background

The Volume Raycaster Actor is part of the Met.3D visualization suite, designed for meteorological and atmospheric data [S1]. It supports interactive exploration of simulation data and provides advanced rendering techniques such as shadow casting and normal curves [S1]. The actor is documented in the Met.3D user manual [S1].

## Sources

[S1] Volume raycaster (3D iso-surfaces and DVR), https://met3d.readthedocs.io/en/latest/03_user_manual/actors/volume_raycaster.html, fetched 2026-10-27
