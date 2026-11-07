# Efficient multiple scattering in hair using spherical harmonics

Also called: SH Hair Scattering, SH-based multiple scattering for hair, Spherical Harmonic Multiple Scattering in Hair, Volumetric Hair Multiple Scattering

## Overview
This method computes physically based multiple scattering in hair volumes by representing the incident scattered radiance field as spherical harmonic (SH) coefficients, allowing the complex scattering integral to be evaluated via a precomputed sparse matrix multiplication. It solves the high computational cost of directional multiple scattering by exploiting the smoothness of the indirect radiance distribution, reducing the per-ray shading cost from sampling-based integration to a fixed-size linear algebra operation. It differs from its parent [Spherical Harmonic Volume Shading](5946-spherical-harmonic-volume-shading.md) by specifically addressing the anisotropic, fiber-based scattering properties of hair and using a voxelized light-tracing approach to populate the SH coefficients, rather than relying on Fourier transforms or local basis approximations alone.

## How it works
The method consists of four main phases: Voxelization, Light Tracing, Filtering, and Shading (which includes optional Scattering Matrix Precomputation).

### Inputs and Outputs
*   **Inputs:** Hair geometry (segments), light source definitions, fiber properties (diameter $2r$, eccentricity), grid resolution, SH degree $D$.
*   **Outputs:** Rendered image with multiple scattering; optionally, a sparse scattering matrix $F$ for future frames.

### Data Structures

**Voxel Grid**
A 3D rectilinear grid. Each active voxel stores:
*   `mean_direction`: $\bar{\omega}$, a 3D vector (normalized).
*   `std_dev`: $\nu$, a scalar (standard deviation of dot products with $\bar{\omega}$).
*   `attenuation`: $\sigma_t^\perp$, a scalar (perpendicular attenuation coefficient).
*   `coefficients`: A vector of size $(D+1)^2$ containing SH coefficients $c_k$ in the world frame.

**Scattering Matrix**
*   `F`: A sparse matrix of size $(D+1)^2 \times (D+1)^2$.
*   Non-zero entries: Approximately 2% of total entries.

### Step 1: Voxelization
Estimate hair density and direction distribution by rasterizing hair segments into the voxel grid.

1.  Initialize all voxels as empty.
2.  For each hair segment, identify all voxels within a search distance $d$.
3.  For each sample point near the fiber within distance $d$, add contributions to the voxel's direction list. The weight drops linearly to zero at distance $d$.
4.  After processing all segments, for each voxel:
    *   Sum the nearby segment directions weighted by their linear distance drop-off.
    *   Normalize the sum to obtain the mean direction $\bar{\omega}$.
    *   Calculate $\nu$ as the standard deviation of the dot products of the normalized segment directions with $\bar{\omega}$.
    *   Calculate the perpendicular attenuation coefficient $\sigma_t^\perp$ using formula (6):
        $$ \sigma_t^\perp = \frac{2r N}{\pi d^2} $$
        where $2r$ is the fiber diameter, $N$ is the number of nearby fibers, and $d$ is the voxelization search distance.
5.  Mark voxels with no contribution as **empty**.
6.  Mark voxels far from any non-empty cell (unable to participate in lookup) as **inactive**. Do not allocate coefficients for inactive cells.

### Step 2: Light Tracing
Estimate scattered radiance in all directions by tracing paths from light sources.

1.  Trace paths from light sources proportional to emitted power.
2.  After the first geometric scattering event, walk through the voxel grid along each path segment.
3.  For each intersected grid cell $j$ with volume $\Delta V$:
    *   Let $l_p$ be the length of the intersection between path segment $p$ and cell $j$.
    *   Let $\Delta \Phi_p$ be the power associated with the light particle.
    *   Add contribution to the SH coefficients $c_k$ of cell $j$ according to the density definition:
        $$ \text{Contribution} \propto \frac{l_p \cdot \Delta \Phi_p}{\Delta V} $$
        The contribution is distributed into the SH basis functions $Y_k$ based on the direction of the path segment.
4.  Scattering within the volume occurs according to $\sigma_t(\omega)$ derived from the voxel's $\bar{\omega}$ and $\nu$.
5.  Upon scattering, choose a fiber direction $u$ from the distribution described by $\bar{\omega}$ and $\nu$.
6.  Sample the scattered direction $\omega_i$ using a table-driven procedure:
    *   Precompute several thousand random directions for each possible inclination based on the refracted direction for a rough elliptical fiber (simulating roughness by randomizing surface normals).
    *   During simulation, randomly choose one of these stored directions.
    *   Incident rays have a random azimuthal angle and random position across the fiber width.

### Step 3: Filtering
Reduce noise and Gibbs phenomenon ringing.

1.  **Spectral Filtering:** Apply a gentle lowpass filter to the SH coefficients. Attenuate coefficients by a factor that drops smoothly to zero just beyond the maximum degree $D$. This prevents ringing caused by sharp peaks in the accumulated radiance.
2.  **Spatial Filtering:** Convolve the grid of coefficients with a box filter with spherical support. The radius is typically between 1.5 and 3 grid cells. Use more aggressive filtering for animations to prevent temporal flicker.

### Step 4: Shading (Per Eye Ray)
Compute exitant radiance for a point $\mathbf{x}$ on a hair fiber.

1.  **Retrieve Coefficients:** Trilinearly interpolate the SH coefficients $c_k(\mathbf{x})$ from the grid (stored in world frame).
2.  **Rotate to Local Frame:** Rotate the interpolated coefficients $c$ to the local fiber frame at $\mathbf{x}$ using the rotation matrix $R(\mathbf{x})$:
    $$ c_0 = R(\mathbf{x}) c $$
3.  **Compute Exitant Coefficients:**
    *   **Option A (Precomputed Matrix):** Multiply the precomputed scattering matrix $F$ by the rotated incident coefficients:
        $$ d_0 = F c_0 $$
        where $d_0$ are the coefficients of the exitant scattered radiance.
    *   **Option B (Stabbing - Fallback):** If $F$ is not precomputed or parameters change frequently, generate 20 to 50 directional "stabs" by sampling the phase function $f_s(\mathbf{x}, \omega_o, \omega_i) \sin(\omega_i, u)$. Use the filtered grid values to evaluate incident radiance for each stab direction, weight by attenuation, and average.
4.  **Reconstruct Radiance:** Convert exitant coefficients $d_0$ back to world frame and evaluate the SH basis functions to get the exitant radiance $L_o(\mathbf{x}, \omega_o)$.
5.  **Add Direct Illumination:** Calculate direct illumination separately using standard ray tracing and add to the result.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

# Constants
D = 15  # SH degree
NUM_COEFFS = (D + 1) ** 2  # 256

def voxelize_hair(segments, grid_size, search_distance_d, fiber_diameter_2r):
    """
    Step 1: Voxelize hair geometry to populate voxel parameters.
    """
    grid = create_empty_grid(grid_size)
    
    for segment in segments:
        voxels_in_range = get_voxels_near_segment(segment, search_distance_d)
        for voxel in voxels_in_range:
            # Weight drops linearly to zero at distance d
            weight = max(0, 1.0 - distance(segment, voxel) / search_distance_d)
            voxel.add_direction(segment.direction, weight)
            
    for voxel in grid.active_voxels():
        if voxel.has_directions():
            # Calculate mean direction
            weighted_sum = voxel.sum_weighted_directions()
            voxel.mean_direction = normalize(weighted_sum)
            
            # Calculate standard deviation nu
            dot_products = [dot(normalize(dir), voxel.mean_direction) for dir in voxel.directions]
            voxel.std_dev = std_dev(dot_products)
            
            # Calculate perpendicular attenuation
            N = voxel.num_nearby_fibers
            voxel.sigma_t_perp = (fiber_diameter_2r * N) / (math.pi * search_distance_d ** 2)
        else:
            voxel.mark_empty()
            
    return grid

def light_trace(lights, grid, fiber_properties):
    """
    Step 2: Trace light paths and accumulate SH coefficients.
    """
    for light in lights:
        # Trace paths proportional to emitted power
        paths = trace_paths_from_light(light)
        
        for path in paths:
            # After first geometric scattering, walk through grid
            for segment in path.segments_after_first_scatter:
                intersected_voxels = get_voxels_intersected_by_segment(segment)
                
                for voxel in intersected_voxels:
                    # Calculate contribution proportional to path length and power
                    length = segment.length_in_voxel(voxel)
                    power = path.power
                    volume = voxel.volume
                    
                    # Accumulate into SH coefficients
                    # Direction of segment determines which SH basis functions are excited
                    voxel.accumulate_sh_coefficients(segment.direction, power * length / volume)
                    
                # Handle scattering event within voxel
                if random() < scattering_probability(voxel.sigma_t_perp):
                    # Choose fiber direction from distribution
                    u = sample_fiber_direction(voxel.mean_direction, voxel.std_dev)
                    
                    # Sample scattered direction using table-driven method
                    omega_i = sample_scattered_direction_table(u, fiber_properties)
                    
                    # Continue path with new direction and reduced power
                    path.continue(omega_i, voxel.attenuation(omega_i))

def filter_grid(grid):
    """
    Step 3: Apply spectral and spatial filtering.
    """
    # Spectral filtering: Lowpass filter SH coefficients
    for voxel in grid.active_voxels():
        for k in range(NUM_COEFFS):
            # Attenuate coefficients smoothly to zero beyond max degree
            factor = smooth_step(k, D) 
            voxel.coefficients[k] *= factor
            
    # Spatial filtering: Box filter with spherical support
    radius = 2.0 # Typical value between 1.5 and 3
    filtered_grid = create_empty_grid(grid.size)
    for voxel in grid.active_voxels():
        neighbors = get_spherical_neighbors(voxel, radius)
        voxel.coefficients = average_coefficients(voxel.coefficients, neighbors)
        
    return filtered_grid

def shade_hair_point(x, direction_o, grid, scattering_matrix_F=None):
    """
    Step 4: Compute exitant radiance for a point on a hair fiber.
    """
    # Retrieve and rotate coefficients
    c_world = trilinear_interpolate_coefficients(grid, x)
    R_x = get_rotation_matrix_to_local_frame(x, direction_o)
    c_local = rotate_sh_coefficients(c_world, R_x)
    
    if scattering_matrix_F is not None:
        # Matrix multiplication approach
        d_local = matrix_vector_multiply(scattering_matrix_F, c_local)
    else:
        # Stabbing fallback
        d_local = compute_stabbing_sh_coefficients(c_world, direction_o, grid)
        
    # Convert back to world frame and evaluate
    R_x_inv = transpose(R_x)
    d_world = rotate_sh_coefficients(d_local, R_x_inv)
    
    L_o = evaluate_sh_basis(d_world, direction_o)
    
    # Add direct illumination
    L_d = trace_direct_light(x, direction_o)
    
    return L_o + L_d

def precompute_scattering_matrix(fiber_properties):
    """
    Step 5 (Optional): Precompute sparse scattering matrix F.
    """
    F = zeros((NUM_COEFFS, NUM_COEFFS))
    
    # Monte Carlo integration of the scattering integral
    num_samples = 500000
    for i in range(NUM_COEFFS):
        for j in range(NUM_COEFFS):
            # Sample directions omega'_o and omega'_i
            omega_o = sample_sphere()
            omega_i = sample_sphere()
            
            # Evaluate phase function and basis functions
            fs = evaluate_phase_function(omega_o, omega_i, fiber_properties)
            Y_o = evaluate_sh_basis(i, omega_o)
            Y_i = evaluate_sh_basis(j, omega_i)
            
            # Accumulate integral
            F[i, j] += fs * Y_o * Y_i * sin(omega_i)
            
    # Normalize and threshold
    F /= num_samples
    threshold = 0.01 # Example threshold
    F = where(abs(F) > threshold, F, 0)
    
    return F
```

### Parameters
*   **SH Degree ($D$):** Typically up to 15, resulting in 256 coefficients per grid point.
*   **Fiber Diameter ($2r$):** Used in $\sigma_t^\perp$ calculation.
*   **Voxelization Search Distance ($d$):** Determines the spatial extent of fiber influence per voxel.
*   **Smoothing Factor:** Values range from 2.0 to 2.2 cells across tested scenes.
*   **Grid Cell Parameter ($r$):** Values range from 8 to 10 cells across tested scenes (likely referring to smoothing radius or similar).
*   **Paths Traced:** Ranges from 100M to 160M depending on the scene.
*   **Image Resolution:** 600 x 600 pixels.
*   **Rays per Pixel:** 64.

### Edge Cases and Pitfalls
*   **Gibbs Phenomenon:** Directly using accumulated coefficients with sharp peaks leads to ringing. Increasing SH degree does not reduce this until the peak is fully resolved. Solution: Apply lowpass filter.
*   **Temporal Incoherence:** Spatial noise in static images is hard to notice, but becomes perceptible flicker in animations. Solution: Use more aggressive spatial filtering for animations.
*   **Sampling Difficulty:** The Marschner et al. phase function model is difficult to sample directly. Solution: Use table-driven scattering with precomputed random directions based on refracted directions for rough elliptical fibers.
*   **Smoothness Assumption:** The method limits accuracy for highly directional radiance distributions, as they cannot be accurately represented by the SH basis.
*   **Spatial Resolution:** Constrained by rectilinear grid spacing, causing spatial and angular blurring comparable to photon mapping.

## Operating characteristics
*   **Time Complexity:** The shading step is $O(1)$ per ray (matrix multiplication or stabbing). The dominant cost is light tracing, which scales with the number of paths traced (100M-160M paths).
*   **Memory Complexity:** Proportional to the number of active voxels times the number of SH coefficients. For a 33x109x49 grid with 46,074 active cells and 256 coefficients, memory usage is significantly lower than photon mapping (270 MB vs 1.5 GB).
*   **Measured Performance:** On a 3.0 GHz Intel Core 2 Duo workstation, single-threaded, 1 GB RAM:
    *   Voxelization: ~10 seconds.
    *   Attenuation/Scattering function tabulation: ~3 seconds each.
    *   Scattering matrix precomputation: 700 seconds (500,000 evaluations).
    *   Multiple scattering rendering: 13 minutes per frame (animations), 5 minutes for stills (vs 1.5 hours for photon mapping).
    *   Direct illumination: Additional 10 minutes.
*   **Strengths:** Significantly faster than photon mapping for hair multiple scattering; preserves directionally varying effects; low memory footprint.
*   **Limitations:** Limited by SH basis smoothness; spatial blurring from grid resolution; requires precomputation for optimal speed.
*   **When to use:** Production hair rendering where multiple scattering is required and performance is critical.
*   **When not to use:** Scenes with highly directional, non-smooth indirect lighting that cannot be captured by SH; real-time applications requiring dynamic hair geometry changes without re-voxelization.

## Background
This method was introduced by [S1] to address the high cost of physically based multiple scattering in hair. It builds on the fiber radiance cache concept from Moon and Marschner [2006] but replaces the directional stabbing approach with spherical harmonics to exploit the smoothness of multiple scattering. The implementation was done in Java using a hierarchical grid for ray-hair intersection acceleration.

## Sources
[S1] Efficient multiple scattering in hair using spherical harmonics, https://doi.org/10.1145/1360612.1360630, fetched 2026-10-29
