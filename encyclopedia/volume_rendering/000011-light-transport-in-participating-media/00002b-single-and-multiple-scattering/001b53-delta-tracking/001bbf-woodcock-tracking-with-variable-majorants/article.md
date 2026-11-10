# Woodcock Tracking with Variable Majorants

Also called: Variable Majorant Delta Tracking, Coarse Grid Majorant Tracking

## Overview
Woodcock Tracking with Variable Majorants is a Monte Carlo method for simulating light transport in participating media that reduces the variance and computational cost of standard Delta Tracking by using spatially varying upper bounds on the extinction coefficient [S1, unofficial]. While standard Delta Tracking uses a single global majorant, this method subdivides the volume into regions (such as coarse grid macrocells) to compute tighter local majorants, thereby significantly reducing the number of null collisions in sparse or heterogeneous areas [S1, S2, unofficial]. It builds on [Delta tracking](985-delta-tracking.md) by optimizing the homogenization step, addressing the inefficiency of uniform majorants in volumes with large empty spaces [S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A volume dataset (scalar field), a ray (origin, direction, current distance $t$), a transfer function mapping scalar values to color/density, and a majorant structure (global or regional).
- **Output:** The accumulated color contribution from the first real interaction or the background color if the ray exits the volume.

**Data Structures:**
- **Volume:** A data structure supporting point sampling `volume.sample(position)` returning a scalar value.
- **Transfer Function:** A function `transferFunction.mapColor(scalar)` returning a struct containing `density` (extinction coefficient $\sigma_t$) and other optical properties.
- **Majorant Grid:** A 3D grid of macrocells. Each cell stores a precomputed maximum density value ($\sigma_{t,max}$) for that region. The grid size is typically $32 \times 32 \times 32$ [S2, unofficial].

**Steps:**
1. **Initialization:** Determine the majorant $\sigma_{maj}$ for the current region containing the ray's origin. If the ray crosses a region boundary, update $\sigma_{maj}$ to the precomputed maximum density of the new region [S2, unofficial].
2. **Free Path Sampling:** Sample the distance to the next collision using the exponential distribution with the current majorant:
   $$ t_{step} = -\frac{\ln(1 - r)}{\sigma_{maj}} $$
   where $r$ is a uniform random number in $[0, 1)$ [S1, unofficial].
3. **Position Update:** Advance the ray:
   $$ \mathbf{p}_{new} = \mathbf{p}_{origin} + \mathbf{d}_{ray} \times (t + t_{step}) $$
4. **Sampling:** Query the volume at $\mathbf{p}_{new}$:
   $$ s = \text{volume.sample}(\mathbf{p}_{new}) $$
5. **Mapping:** Map the scalar to optical properties:
   $$ \mathbf{c} = \text{transferFunction.mapColor}(s) $$
   Let $\sigma_t = \mathbf{c}.density$ [S1, unofficial].
6. **Null-Collision Test:** Generate a new random number $r' \in [0, 1)$. If:
   $$ \sigma_t \ge r' \times \sigma_{maj} $$
   then a real collision has occurred. Return the sample (or continue with scattering logic) [S1, unofficial].
7. **Null Collision:** If the test fails, the collision is null. Continue the loop from Step 2 with the same ray position updated by $t_{step}$ [S1, unofficial].
8. **Termination:** If the ray exits the volume bounds, return the background color [S1, unofficial].

**Parameters:**
- **Majorant ($\sigma_{maj}$):** The extinction coefficient upper bound. In variable majorant tracking, this changes per region [S1, unofficial].
- **Grid Resolution:** The subdivision grid size. A $32 \times 32 \times 32$ grid is cited as a compromise between overhead and null-collision reduction [S2, unofficial].

**Pitfalls:**
- **Global Majorant:** Using a single majorant for the entire volume results in excessive null collisions in sparse regions [S1, unofficial].
- **Boundary Handling:** Failing to update the majorant when crossing region boundaries leads to incorrect sampling probabilities [S2, unofficial].
- **Memory Overhead:** Storing the majorant grid increases VRAM usage [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def woodcock_tracking_variable_majorant(volume, ray, majorant_grid, transfer_function, background_color):
    """
    Simulates light transport using Woodcock tracking with variable majorants.
    
    Args:
        volume: Object with method sample(position) -> scalar.
        ray: Object with attributes origin, dir, t (current distance).
        majorant_grid: 3D grid where get_max_density(region_id) returns sigma_t,max.
        transfer_function: Object with method mapColor(scalar) -> {density, ...}.
        background_color: Color to return if ray exits volume.
        
    Returns:
        Color contribution from the first real interaction.
    """
    # Helper to find region ID based on ray position
    def get_region_id(position):
        # Implementation depends on grid layout; returns index into majorant_grid
        pass

    # Initialize majorant based on starting region
    current_region_id = get_region_id(ray.origin)
    sigma_maj = majorant_grid.get_max_density(current_region_id)
    
    while volume.contains(ray.origin + ray.dir * ray.t):
        # 1. Sample free path
        r = random.uniform(0, 1)
        t_step = -log(1 - r) / sigma_maj
        
        # 2. Update ray position
        ray.t += t_step
        pos = ray.origin + ray.dir * ray.t
        
        # 3. Check if ray crossed a region boundary
        new_region_id = get_region_id(pos)
        if new_region_id != current_region_id:
            current_region_id = new_region_id
            sigma_maj = majorant_grid.get_max_density(current_region_id)
            
        # 4. Sample volume and map to density
        scalar = volume.sample(pos)
        color_data = transfer_function.mapColor(scalar)
        sigma_t = color_data.density
        
        # 5. Null-collision test
        r_test = random.uniform(0, 1)
        if sigma_t >= r_test * sigma_maj:
            # Real collision
            return color_data # Or further scattering logic
            
    # Ray exited volume
    return background_color
```

## Subtopics

### Region Subdivision
A technique to reduce null collisions in sparse volumes by subdividing the volume into smaller regions, each with its own majorant extinction coefficient [S2, unofficial]. It sets itself apart by explicitly managing the majorant update logic at region boundaries, whereas other variants might use continuous functions or hierarchical structures. It is chosen when the volume has large empty spaces or distinct density clusters, such as clouds or the Stanford Bunny model [S2, unofficial]. The grid size of $32 \times 32 \times 32$ is a typical compromise [S2, unofficial].

### Weighted Tracking
A non-analog estimator variant that allows the combined coefficient to occasionally underestimate the real extinction, requiring reweighting of samples to maintain unbiasedness [S3]. It differs from standard Woodcock tracking by permitting negative null-collision coefficients, which reduces the majorant but introduces statistical weights [S3]. It is used when tighter majorants are difficult to compute or when specific weighting schemes are needed to increase collision rates in measurement regions [S3].

## Operating characteristics
- **Time Complexity:** Proportional to the number of null collisions plus one real collision. The number of null collisions is minimized by tighter local majorants [S1, unofficial].
- **Memory Complexity:** $O(N)$ where $N$ is the number of macrocells in the majorant grid. A $32^3$ grid adds negligible overhead compared to the volume data [S2, unofficial].
- **Measured Performance:** In the Stanford Bunny example, region subdivision reduced render time from 216s to 33s [S2, unofficial]. In cloud rendering, it reduced time from 248s to 14s (at 10 samples per pixel) [S2, unofficial].
- **Strengths:** Significantly reduces null collisions in sparse volumes; remains unbiased; simple to implement with a coarse grid [S2, unofficial].
- **Limitations:** Requires precomputation of regional majorants; increases VRAM usage for the grid; boundary handling adds complexity [S2, unofficial].
- **When to Use:** Heterogeneous volumes with large empty spaces or sharp density variations [S2, unofficial].
- **When Not to Use:** Uniform volumes where a single global majorant is tight enough, or when VRAM is extremely constrained [S2, unofficial].

## Background
Woodcock tracking, also known as delta tracking, was introduced by Woodcock et al. in 1975 for particle transport in heterogeneous media [S3]. The variable majorant approach extends this by using coarse structured grid macrocells to obtain tighter majorants for parts of a volume, a technique cited in recent works on direct volume rendering [S1, unofficial]. The specific implementation details regarding region subdivision and performance benchmarks are documented in unofficial sources on volumetric rendering and path tracing [S1, S2, unofficial].

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
[S2] Volumetric Rendering using Path Tracing, https://zydak.github.io/Clouds/index.html, fetched 2026-10-27 (unofficial)
[S3] PDF Spectral and Decomposition Tracking for Rendering Heterogeneous Volumes, https://media.disneyanimation.com/uploads/production/publication_asset/158/asset/SpectralAndDecompositionTracking.pdf, fetched 2026-10-30
