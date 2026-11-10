# Two-Pass Shadow Casting (Kajiya and Von Herzen)

Also called: Kajiya and Von Herzen Method, Shadow Calculation via Illumination Pass, Shadow Casting via Two-Pass Method, Two-Pass Numerical Algorithm for Shadows, Two-Pass Numerical Shadow Algorithm

## Overview
The Two-Pass Shadow Casting method, introduced by Kajiya and Von Herzen, computes shadows in participating media by decoupling light transport into an illumination pass and a gathering pass [S1, unofficial][S2]. It solves the problem of realistic shadowing in volume rendering, where light is attenuated by the medium itself, by propagating flux from the light source through the volume layer-by-layer before scattering it to the viewpoint [S1, unofficial][S2]. Unlike recursive ray tracing, which sends individual "shadow feelers" to the light source for every point along the viewing ray, this method integrates the shadowing effect globally along the ray, allowing the illumination pass to be amortized over multiple frames if only the viewpoint moves [S2].

## How it works
The method evaluates the rendering equation for a single scattering event, accounting for the transparency of the volume between the light source and the point of interest, and between that point and the viewpoint [S1, unofficial][S2].

**Inputs and Outputs:**
*   **Inputs:**
    *   Volume density field $\tau(\mathbf{x})$ (extinction coefficient) [S1, unofficial][S2].
    *   Light source intensity $L$ (assumed infinite/directional for the base formulation) [S1, unofficial][S2].
    *   Light direction $-\omega'$ (unit vector pointing from the volume to the light) [S1, unofficial][S2].
    *   Viewpoint position $\mathbf{X}$ and viewing direction $-\omega$ (unit vector pointing from the volume to the camera) [S1, unofficial][S2].
    *   Shading rule $r(\mathbf{x}, \omega, \omega')$ (phase function or reflectance model) [S1, unofficial][S2].
    *   Background intensity $I_0$ [S1, unofficial][S2].
    *   Ray length $D$ (distance from viewpoint to the far plane or volume boundary) [S1, unofficial][S2].
*   **Outputs:**
    *   Final pixel intensity $I(\mathbf{X})$ [S1, unofficial][S2].

**Data Structures:**
*   The volume is typically represented as a 3D grid or voxel array where each cell stores the extinction coefficient $\tau$ [S1, unofficial][S2].
*   The algorithm processes the volume "one voxel layer at a time" relative to the light direction [S1, unofficial][S2]. This implies iterating through the volume along the axis defined by the light direction $\omega'$, or using a spatial partitioning structure that allows efficient traversal in that direction [G].

**Numbered Steps:**

1.  **Pass 1: Illumination Calculation**
    Compute the illumination $i(\mathbf{X}, \omega')$ reaching each point $\mathbf{X}$ in the volume from the light source [S2]. This involves propagating flux from the light source through the volume, accounting for the transparency of each layer before propagating to the next [S1, unofficial][S2].
    
    The illumination at point $\mathbf{X}$ from an infinite light source $L$ in direction $-\omega'$ is given by:
    $$ i(\mathbf{X}, \omega') = L \exp\left(-\int_0^\infty \tau(\mathbf{X} - t\omega') \, dt\right) $$
    
    In practice, the integral does not run to infinity but only to the edge of the data volume [S1, unofficial][S2]. Numerically, this is approximated by summing $\tau \cdot \Delta s$ along the ray from the light source entry point to $\mathbf{X}$ [G].

2.  **Pass 2: Gathering (Scattering to Viewpoint)**
    Reflect or scatter the computed illumination $i(\mathbf{X}, \omega')$ to the viewpoint using the shading rule $g(\mathbf{X}, \omega) = r(\mathbf{X}, \omega, \omega') i(\mathbf{X}, \omega')$ [S1, unofficial][S2]. Gather the reflected intensity $g(\mathbf{X}, \omega)$ along viewing rays [S2].
    
    To simplify the integration, the ray parameterization is reversed so that the parameter $s$ starts at the viewpoint $\mathbf{X}$ and goes out in direction $-\omega$, reaching $\mathbf{X} - s\omega$ at distance $s$ [S1, unofficial][S2]. The final intensity $I(\mathbf{X})$ is calculated as:
    $$ I(\mathbf{X}) = I_0 \exp\left(-\int_0^D \tau(\mathbf{X} - t\omega) \, dt\right) + \int_0^D r(\mathbf{X} - s\omega, \omega, \omega') L \exp\left(-\int_0^\infty \tau(\mathbf{X} - s\omega - t\omega') \, dt\right) \exp\left(-\int_0^s \tau(\mathbf{X} - t\omega) \, dt\right) \, ds $$
    
    Here:
    *   The first term is the background intensity attenuated by the volume along the view ray [S1, unofficial][S2].
    *   The integral term represents the light scattered from the volume into the view direction [S1, unofficial][S2].
    *   The term $\exp\left(-\int_0^\infty \tau(\mathbf{X} - s\omega - t\omega') \, dt\right)$ represents the illumination $i(\mathbf{X} - s\omega, \omega')$ at the scattering point $\mathbf{X} - s\omega$ [S1, unofficial][S2].
    *   The term $\exp\left(-\int_0^s \tau(\mathbf{X} - t\omega) \, dt\right)$ represents the transmittance from the scattering point to the viewpoint [S1, unofficial][S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_pass_shadow_casting(tau_grid, volume_bounds, light_dir, view_dir, viewpoint, background_intensity, phase_func, ray_length, step_size):
    """
    Computes volumetric shadows using the Kajiya and Von Herzen two-pass method.
    
    Args:
        tau_grid: 3D array of extinction coefficients tau(x).
        volume_bounds: Tuple (min_x, max_x, min_y, max_y, min_z, max_z) defining the volume.
        light_dir: Unit vector omega' pointing from volume to light.
        view_dir: Unit vector omega pointing from volume to camera.
        viewpoint: 3D position of the camera X.
        background_intensity: Scalar I0.
        phase_func: Function r(x, omega, omega') returning scattering coefficient.
        ray_length: Distance D from viewpoint to far plane.
        step_size: Sampling interval Delta s.
        
    Returns:
        Final intensity I(X).
    """
    def sample_tau(pos):
        x, y, z = pos
        if (volume_bounds[0] <= x < volume_bounds[1] and
            volume_bounds[2] <= y < volume_bounds[3] and
            volume_bounds[4] <= z < volume_bounds[5]):
            return tau_grid[x, y, z]
        return 0.0

    # Pass 1: Precompute Illumination Field
    # Propagate flux from light source through the volume, one voxel layer at a time.
    # We iterate over voxels sorted by projection onto light_dir to simulate layer traversal.
    illumination_field = [[0.0 for _ in range(volume_bounds[1]-volume_bounds[0])] 
                          for _ in range(volume_bounds[3]-volume_bounds[2])]
    
    # Collect voxel centers and sort by light direction for layer-by-layer propagation
    voxels = []
    for x in range(volume_bounds[0], volume_bounds[1]):
        for y in range(volume_bounds[2], volume_bounds[3]):
            for z in range(volume_bounds[4], volume_bounds[5]):
                voxels.append((x, y, z))
    voxels.sort(key=lambda v: v[0]*light_dir[0] + v[1]*light_dir[1] + v[2]*light_dir[2])
    
    for X in voxels:
        optical_depth = 0.0
        pos = list(X)
        # March from X in direction light_dir to the volume boundary
        while (volume_bounds[0] <= pos[0] < volume_bounds[1] and
               volume_bounds[2] <= pos[1] < volume_bounds[3] and
               volume_bounds[4] <= pos[2] < volume_bounds[5]):
            optical_depth += sample_tau(tuple(pos)) * step_size
            pos[0] += light_dir[0] * step_size
            pos[1] += light_dir[1] * step_size
            pos[2] += light_dir[2] * step_size
        illumination_field[X[0]-volume_bounds[0]][X[1]-volume_bounds[2]] = background_intensity * exp(-optical_depth)

    # Pass 2: Gather Scattered Light along view ray
    scattering_integral = 0.0
    transmittance_view = 1.0
    current_pos = list(viewpoint)
    
    # Iterate s from 0 to ray_length
    num_steps = int(ray_length / step_size)
    for s in range(num_steps):
        scattering_point = [viewpoint[0] - view_dir[0] * s * step_size,
                            viewpoint[1] - view_dir[1] * s * step_size,
                            viewpoint[2] - view_dir[2] * s * step_size]
        
        if (volume_bounds[0] <= scattering_point[0] < volume_bounds[1] and
            volume_bounds[2] <= scattering_point[1] < volume_bounds[3] and
            volume_bounds[4] <= scattering_point[2] < volume_bounds[5]):
            
            # Retrieve precomputed illumination
            i_scatter = illumination_field[int(scattering_point[0]-volume_bounds[0])][int(scattering_point[1]-volume_bounds[2])]
            
            # Phase function evaluation
            r_val = phase_func(scattering_point, view_dir, light_dir)
            
            # Accumulate scattering integral: r * i * T_view * ds
            scattering_integral += r_val * i_scatter * transmittance_view * step_size
            
            # Update transmittance from camera to current scattering point
            tau_val = sample_tau(scattering_point)
            transmittance_view *= exp(-tau_val * step_size)
            
        current_pos[0] += view_dir[0] * step_size
        current_pos[1] += view_dir[1] * step_size
        current_pos[2] += view_dir[2] * step_size
        
    background_term = background_intensity * transmittance_view
    return background_term + scattering_integral
```

**Parameters:**
*   **Light Intensity ($L$):** Typically normalized to 1.0 or based on physical units [G].
*   **Ray Length ($D$):** Determined by the volume bounds or camera clipping planes [G].
*   **Step Size ($\Delta s$):** Determined by the voxel size or a fixed sampling rate [G]. Smaller steps increase accuracy but cost more [G].

**Edge Cases and Pitfalls:**
*   **Multiple Scattering:** The method is a single-scattering model. It is only valid if the albedo or density is low, so that multiple scattering is unlikely [S1, unofficial][S2]. It is not suitable for dense media like atmospheric clouds where multiple scattering is significant [S1, unofficial][S2].
*   **Performance:** The method takes only twice as long as the gathering pass without shadows [S1, unofficial][S2].
*   **Amortization:** The illumination pass can be amortized over several animated frames if only the viewpoint moves [S2].
*   **Boundary Conditions:** The integral for illumination runs to the edge of the data volume, not infinity [S1, unofficial][S2].

## Subtopics

### [Two-pass Spectral Rendering Integral](5266-two-pass-spectral-rendering-integral.md)
The Two-pass Spectral Rendering Integral is a method for computing volumetric shadows in participating media by adapting the Kajiya and von Herzen two-pass algorithm to spectral representation. It separates shadow computation (illumination phase) from radiance accumulation (radiation phase), allowing for the simulation of complex spectral effects such as fluorescence and phosphorescence. It is distinguished from its parent [Two-Pass Shadow Casting (Kajiya and Von Herzen)](2196-two-pass-shadow-casting-kajiya-and-von-herzen.md) by its explicit handling of wavelength-dependent opacity and emission spectra.

## Operating characteristics
*   **Time Complexity:** The method takes only twice as long as the gathering pass without shadows [S1, unofficial][S2]. This is significantly faster than recursive ray tracing for dense volumes, as it avoids sending shadow feelers for every sample point.
*   **Memory Complexity:** Requires storing the illumination field for the entire volume, which is $O(N)$ where $N$ is the number of voxels.
*   **Strengths:**
    *   Provides realistic shadows in participating media.
    *   Efficient for static lights and moving viewpoints, as the illumination pass can be amortized [S2].
    *   Simpler to implement than full path tracing for single-scattering scenarios.
*   **Limitations:**
    *   Single-scattering only; inaccurate for high-density or high-albedo media [S1, unofficial][S2].
    *   Requires precomputing the illumination field, which can be memory-intensive for large volumes.
    *   Less flexible for dynamic lights compared to ray casting methods.
*   **When to Use:** Suitable for medical imaging (e.g., Heidelberg ray tracing model) and scenes with low-density participating media where single scattering is a good approximation [S1, unofficial][S2].
*   **When Not to Use:** Unsuitable for atmospheric clouds or dense fog where multiple scattering is dominant [S1, unofficial][S2].

## Background
The two-pass numerical algorithm for shadows was suggested by Kajiya and Von Herzen [S1, unofficial][S2]. It was introduced as a more general approach to account for shadows in volume rendering, moving beyond simpler shading models that ignored volume transparency between the light and the point of interest. The method has been systematically applied to medical images in the "Heidelberg ray tracing model" by Meinzer et al. [S1, unofficial][S2]. While its utility in general volume rendering applications has been questioned due to the prevalence of multiple scattering in natural media like clouds, it remains a foundational technique for single-scattering shadow computation [S1, unofficial][S2].

## Sources
[S1] [PDF] Optical Models for Direct Volume Rendering - Duke Computer Science, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S2] Optical models for direct volume rendering (PDF) - Academia.edu, https://www.academia.edu/49467168/Optical_models_for_direct_volume_rendering, fetched 2026-10-27
