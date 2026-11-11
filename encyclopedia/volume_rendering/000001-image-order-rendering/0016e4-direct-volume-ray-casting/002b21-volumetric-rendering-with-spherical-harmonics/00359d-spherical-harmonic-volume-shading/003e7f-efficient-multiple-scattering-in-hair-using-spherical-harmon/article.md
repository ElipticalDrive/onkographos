# Efficient multiple scattering in hair using spherical harmonics

Also called: SH Hair Scattering, SH-based multiple scattering for hair, Spherical Harmonic Multiple Scattering in Hair, Volumetric Hair Multiple Scattering

## Overview
This method computes physically based multiple scattering in hair volumes by representing the incident scattered radiance field as spherical harmonic (SH) coefficients, allowing the complex scattering integral to be evaluated via a precomputed sparse matrix multiplication [S1]. It solves the high computational cost of directional multiple scattering by exploiting the smoothness of the indirect radiance distribution, reducing the per-ray shading cost from sampling-based integration to a fixed-size linear algebra operation [S1]. It differs from its parent [Spherical Harmonic Volume Shading](5946-spherical-harmonic-volume-shading.md) by specifically addressing the anisotropic, fiber-based scattering properties of hair and using a voxelized light-tracing approach to populate the SH coefficients, rather than relying on Fourier transforms or local basis approximations alone [S1].

## How it works
The method consists of four main phases: Voxelization, Light Tracing, Filtering, and Shading (which includes optional Scattering Matrix Precomputation) [S1].

### Inputs and Outputs
*   **Inputs:** Hair geometry (segments), light source definitions, fiber properties (diameter $2r$, eccentricity), grid resolution, SH degree $D$ [S1].
*   **Outputs:** Rendered image with multiple scattering; optionally, a sparse scattering matrix $F$ for future frames [S1].

### Data Structures

**Voxel Grid**
A 3D rectilinear grid. Each active voxel stores:
*   `mean_direction`: $\bar{\omega}$, a 3D vector (normalized) [S1].
*   `std_dev`: $\nu$, a scalar (standard deviation of dot products with $\bar{\omega}$) [S1].
*   `attenuation`: $\sigma_t^\perp$, a scalar (perpendicular attenuation coefficient) [S1].
*   `coefficients`: A vector of size $(D+1)^2$ containing SH coefficients $c_k$ in the world frame [S1].

**Scattering Matrix**
*   `F`: A sparse matrix of size $(D+1)^2 \times (D+1)^2$ [S1].
*   Non-zero entries: Approximately 2% of total entries [S1].

### Step 1: Voxelization
Estimate hair density and direction distribution by rasterizing hair segments into the voxel grid [S1].

1.  Initialize all voxels as empty.
2.  For each hair segment, identify all voxels within a search distance $d$ [S1].
3.  For each sample point near the fiber within distance $d$, add contributions to the voxel's direction list. The weight drops linearly to zero at distance $d$ [S1].
4.  After processing all segments, for each voxel:
    *   Sum the nearby segment directions weighted by their linear distance drop-off [S1].
    *   Normalize the sum to obtain the mean direction $\bar{\omega}$ [S1].
    *   Calculate $\nu$ as the standard deviation of the dot products of the normalized segment directions with $\bar{\omega}$ [S1].
    *   Calculate the perpendicular attenuation coefficient $\sigma_t^\perp$ using formula (6):
        $$ \sigma_t^\perp = \frac{2r N}{\pi d^2} $$
        where $2r$ is the fiber diameter, $N$ is the number of nearby fibers, and $d$ is the voxelization search distance [S1].
5.  Mark voxels with no contribution as **empty** [S1].
6.  Mark voxels far from any non-empty cell (unable to participate in lookup) as **inactive**. Do not allocate coefficients for inactive cells [S1].

### Step 2: Light Tracing
Estimate scattered radiance in all directions by tracing paths from light sources [S1].

1.  Trace paths from light sources proportional to emitted power [S1].
2.  After the first geometric scattering event, walk through the voxel grid along each path segment [S1].
3.  For each intersected grid cell $j$ with volume $\Delta V$:
    *   Let $l_p$ be the length of the intersection between path segment $p$ and cell $j$.
    *   Let $\Delta \Phi_p$ be the power associated with the light particle.
    *   Add contribution to the SH coefficients $c_k$ of cell $j$ according to the density definition:
        $$ \text{Contribution} \propto \frac{l_p \cdot \Delta \Phi_p}{\Delta V} $$
        The contribution is distributed into the SH basis functions $Y_k$ based on the direction of the path segment [S1].
4.  Scattering within the volume occurs according to $\sigma_t(\omega)$ derived from the voxel's $\bar{\omega}$ and $\nu$ [S1].
5.  Upon scattering, choose a fiber direction $u$ from the distribution described by $\bar{\omega}$ and $\nu$ [S1].
6.  Sample the scattered direction $\omega_i$ using a table-driven procedure:
    *   Precompute several thousand random directions for each possible inclination based on the refracted direction for a rough elliptical fiber (simulating roughness by randomizing surface normals) [S1].
    *   During simulation, randomly choose one of these stored directions [S1].
    *   Incident rays have a random azimuthal angle and random position across the fiber width [S1].

### Step 3: Filtering
Reduce noise and Gibbs phenomenon ringing.

1.  **Spectral Filtering:** Apply a gentle lowpass filter to the SH coefficients. Attenuate coefficients by a factor that drops smoothly to zero just beyond the maximum degree $D$. This prevents ringing caused by sharp peaks in the accumulated radiance [S1].
2.  **Spatial Filtering:** Convolve the grid of coefficients with a box filter with spherical support. The radius is typically between 1.5 and 3 grid cells. Use more aggressive filtering for animations to prevent temporal flicker [S1].

### Step 4: Shading (Per Eye Ray)
Compute exitant radiance for a point $\mathbf{x}$ on a hair fiber.

1.  **Retrieve Coefficients:** Trilinearly interpolate the SH coefficients $c_k(\mathbf{x})$ from the grid (stored in world frame) [S1].
2.  **Rotate to Local Frame:** Rotate the interpolated coefficients $c$ to the local fiber frame at $\mathbf{x}$ using the rotation matrix $R(\mathbf{x})$:
    $$ c_0 = R(\mathbf{x}) c $$
3.  **Compute Exitant Coefficients:**
    *   **Option A (Precomputed Matrix):** Multiply the precomputed scattering matrix $F$ by the rotated incident coefficients:
        $$ d_0 = F c_0 $$
        where $d_0$ are the coefficients of the exitant scattered radiance [S1].
    *   **Option B (Stabbing - Fallback):** If $F$ is not precomputed or parameters change frequently, generate 20 to 50 directional "stabs" by sampling the phase function $f_s(\mathbf{x}, \omega_o, \omega_i) \sin(\omega_i, u)$. Use the filtered grid values to evaluate incident radiance for each stab direction, weight by attenuation, and average [S1].
4.  **Reconstruct Radiance:** Convert exitant coefficients $d_0$ back to world frame and evaluate the SH basis functions to get the exitant radiance $L_o(\mathbf{x}, \omega_o)$ [S1].
5.  **Add Direct Illumination:** Calculate direct illumination separately using standard ray tracing and add to the result [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def voxelize(segments, grid_size, d, fiber_diameter_2r):
    grid = {}
    for seg in segments:
        for v_key in voxels_near_segment(seg, d):
            if v_key not in grid:
                grid[v_key] = {'dirs': [], 'count': 0}
            dist = segment_voxel_distance(seg, v_key)
            w = max(0.0, 1.0 - dist / d)
            grid[v_key]['dirs'].append((seg.dir, w))
            grid[v_key]['count'] += 1
            
    for v_key in list(grid.keys()):
        data = grid[v_key]
        if not data['dirs']:
            del grid[v_key]
            continue
        w_sum = [0.0, 0.0, 0.0]
        for d_vec, w in data['dirs']:
            w_sum[0] += d_vec[0] * w
            w_sum[1] += d_vec[1] * w
            w_sum[2] += d_vec[2] * w
        norm = math.sqrt(sum(v**2 for v in w_sum))
        mean_dir = [v / norm for v in w_sum] if norm > 0 else [0.0, 0.0, 0.0]
        
        dots = [sum(a*b for a, b in zip(d_vec, mean_dir)) for d_vec, _ in data['dirs']]
        mean_dot = sum(dots) / len(dots)
        std_dev = math.sqrt(sum((x - mean_dot)**2 for x in dots) / len(dots))
        
        N = data['count']
        sigma_t_perp = (fiber_diameter_2r * N) / (math.pi * d ** 2)
        
        grid[v_key] = {
            'mean_direction': mean_dir,
            'std_dev': std_dev,
            'sigma_t_perp': sigma_t_perp,
            'coeffs': [0.0] * ((15 + 1) ** 2)
        }
    return grid

def light_trace(lights, grid):
    for light in lights:
        for path in trace_paths_from_light(light):
            for seg in path.segments_after_first_scatter:
                for v_key in voxels_intersected_by_segment(seg):
                    if v_key in grid:
                        vol = voxel_volume(v_key)
                        contrib = seg.length_in_voxel(v_key) * path.power / vol
                        accumulate_sh_coefficients(grid[v_key]['coeffs'], seg.dir, contrib)
                if random.random() < scattering_probability(grid[v_key]['sigma_t_perp']):
                    u = sample_fiber_direction(grid[v_key]['mean_direction'], grid[v_key]['std_dev'])
                    omega_i = sample_scattered_direction_table(u)
                    path.continue(omega_i)

def filter_grid(grid):
    for v_key in grid:
        coeffs = grid[v_key]['coeffs']
        for k in range(len(coeffs)):
            factor = smooth_step(k, 15)
            coeffs[k] *= factor
    radius = 2.0
    for v_key in grid:
        neighbors = get_spherical_neighbors(v_key, radius)
        neighbor_coeffs = [grid[n]['coeffs'] for n in neighbors if n in grid]
        if neighbor_coeffs:
            grid[v_key]['coeffs'] = average_coefficients(neighbor_coeffs)
    return grid

def shade(x, dir_o, grid, F=None):
    c_world = trilinear_interp(grid, x)
    R_x = rotation_to_local(x, dir_o)
    c_local = rotate_sh(c_world, R_x)
    if F is not None:
        d_local = mat_vec_mul(F, c_local)
    else:
        d_local = stabbing(c_world, dir_o, grid)
    d_world = rotate_sh(d_local, transpose(R_x))
    L_o = eval_sh(d_world, dir_o)
    L_d = trace_direct(x, dir_o)
    return L_o + L_d
```

### Parameters
*   **SH Degree ($D$):** Typically up to 15, resulting in 256 coefficients per grid point [S1].
*   **Fiber Diameter ($2r$):** Used in $\sigma_t^\perp$ calculation [S1].
*   **Voxelization Search Distance ($d$):** Determines the spatial extent of fiber influence per voxel [S1].
*   **Smoothing Factor:** Values range from 2.0 to 2.2 cells across tested scenes [S1].
*   **Grid Cell Parameter ($r$):** Values range from 8 to 10 cells across tested scenes [S1].
*   **Paths Traced:** Ranges from 100M to 160M depending on the scene [S1].
*   **Image Resolution:** 600 x 600 pixels [S1].
*   **Rays per Pixel:** 64 [S1].

### Edge Cases and Pitfalls
*   **Gibbs Phenomenon:** Directly using accumulated coefficients with sharp peaks leads to ringing. Increasing SH degree does not reduce this until the peak is fully resolved. Solution: Apply lowpass filter [S1].
*   **Temporal Incoherence:** Spatial noise in static images is hard to notice, but becomes perceptible flicker in animations. Solution: Use more aggressive spatial filtering for animations [S1].
*   **Sampling Difficulty:** The Marschner et al. phase function model is difficult to sample directly. Solution: Use table-driven scattering with precomputed random directions based on refracted directions for rough elliptical fibers [S1].
*   **Smoothness Assumption:** The method limits accuracy for highly directional radiance distributions, as they cannot be accurately represented by the SH basis [S1].
*   **Spatial Resolution:** Constrained by rectilinear grid spacing, causing spatial and angular blurring comparable to photon mapping [S1].

## Operating characteristics
*   **Time Complexity:** The shading step is $O(1)$ per ray (matrix multiplication or stabbing). The dominant cost is light tracing, which scales with the number of paths traced (100M-160M paths) [S1].
*   **Memory Complexity:** Proportional to the number of active voxels times the number of SH coefficients. For a 33x109x49 grid with 46,074 active cells and 256 coefficients, memory usage is significantly lower than photon mapping (270 MB vs 1.5 GB) [S1].
*   **Measured Performance:** On a 3.0 GHz Intel Core 2 Duo workstation, single-threaded, 1 GB RAM:
    *   Voxelization: ~10 seconds [S1].
    *   Attenuation/Scattering function tabulation: ~3 seconds each [S1].
    *   Scattering matrix precomputation: 700 seconds (500,000 evaluations) [S1].
    *   Multiple scattering rendering: 13 minutes per frame (animations), 5 minutes for stills (vs 1.5 hours for photon mapping) [S1].
    *   Direct illumination: Additional 10 minutes [S1].
*   **Strengths:** Significantly faster than photon mapping for hair multiple scattering; preserves directionally varying effects; low memory footprint [S1].
*   **Limitations:** Limited by SH basis smoothness; spatial blurring from grid resolution; requires precomputation for optimal speed [S1].
*   **When to use:** Production hair rendering where multiple scattering is required and performance is critical [S1].
*   **When not to use:** Scenes with highly directional, non-smooth indirect lighting that cannot be captured by SH; real-time applications requiring dynamic hair geometry changes without re-voxelization [S1].

## Background
This method was introduced by [S1] to address the high cost of physically based multiple scattering in hair. It builds on the fiber radiance cache concept from Moon and Marschner [2006] but replaces the directional stabbing approach with spherical harmonics to exploit the smoothness of multiple scattering. The implementation was done in Java using a hierarchical grid for ray-hair intersection acceleration [S1].

## Sources
[S1] Efficient multiple scattering in hair using spherical harmonics, https://doi.org/10.1145/1360612.1360630, fetched 2026-10-29
