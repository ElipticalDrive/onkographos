# Majorant Supergrid Traversal

Also called: Majorant Supergrid for Medium Traversal, Supergrid DDA

## Overview
Majorant Supergrid Traversal is a medium traversal technique that uses a supergrid to allow empty and thin regions to be crossed in far fewer null collisions than paying for the densest voxel in the volume [S1, unofficial]. It builds on [Ray-grid traversal](17-ray-grid-traversal.md) by replacing a single global majorant extinction bound with a spatially varying one, reducing the overhead of null-collision sampling in heterogeneous media [S1, unofficial]. Unlike [Woodcock Tracking with Macrocell Majorant](986-woodcock-tracking-with-macrocell-majorant.md), which may use fixed macrocells, this method dynamically adapts the bounding extinction to local density variations on a coarse grid [S1, unofficial].

## How it works
The method combines a Digital Differential Analyzer (DDA) traversal with a null-collision sampling loop, fused into a single iteration when a supergrid is active [S1, unofficial]. It requires a majorant extinction field defined on a supergrid, which is a downsampled version of the high-resolution medium grid.

**Inputs and Outputs:**
- Input: Ray origin $\mathbf{o}$, direction $\mathbf{d}$, high-resolution extinction/scattering data, supergrid resolution factor $R$, majorant safety factor $M$.
- Output: Sequence of interaction points (or null collisions) along the ray.

**Data Structures:**
- **Supergrid:** A 3D grid with dimensions $N_x/R \times N_y/R \times N_z/R$ (assuming uniform downsampling). Each cell stores the maximum extinction coefficient $\mu_t$ found in the corresponding region of the high-res grid.
- **Majorant:** The value $\mu_{max}$ used for sampling is $\max(\mu_{supergrid\_cell}) \times M$.

**Parameters:**
- `majorant_resolution_factor` ($R$): Controls supergrid resolution. Default value is 8 [S1, unofficial].
- `majorant_factor` ($M$): Safety factor for the majorant. Default value is 1.2 [S1, unofficial].

**Algorithm Steps:**
1. **Initialize Supergrid:** For each cell in the supergrid, compute $\mu_{supergrid}[i,j,k] = \max_{\text{voxels in cell}} (\mu_t(\mathbf{x}))$. If the medium is homogeneous, this step is skipped or rejected [S1, unofficial].
2. **Ray Initialization:** Compute initial voxel indices and boundary distances using standard DDA logic .
3. **Fused Loop:** Iterate while the ray is within bounds:
    a. Determine the current supergrid cell containing the ray's current position.
    b. Retrieve the local majorant $\mu_{local} = \mu_{supergrid}[cell] \times M$.
    c. Sample a null collision distance $t_{null}$ from an exponential distribution with rate $\mu_{local}$: $t_{null} = -\ln(u) / \mu_{local}$, where $u \in (0,1)$ is a uniform random number.
    d. Compare $t_{null}$ with the distance to the next supergrid boundary $t_{grid\_boundary}$.
    e. If $t_{null} < t_{grid\_boundary}$:
        i. Advance the ray by $t_{null}$.
        ii. Sample a real interaction (scattering) with probability $\mu_t(\mathbf{x}) / \mu_{local}$. If sampled, record the interaction and terminate the current path segment. If not, record a null collision and continue.
    f. If $t_{null} \ge t_{grid\_boundary}$:
        i. Advance the ray to $t_{grid\_boundary}$ using the DDA step.
        ii. Update the current supergrid cell.
        iii. Repeat loop.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def majorant_supergrid_traversal(ray_origin, ray_direction, supergrid, majorant_factor, voxel_size):
    # Initialize DDA state
    current_voxel = get_voxel_index(ray_origin, voxel_size)
    step = get_step_direction(ray_direction)
    t_max = get_next_boundary_distances(ray_origin, ray_direction, current_voxel, voxel_size)
    t_delta = get_boundary_distances_delta(ray_direction, voxel_size)
    t_entry = 0.0
    
    # Get initial supergrid cell and majorant
    supergrid_cell = get_supergrid_cell(current_voxel, supergrid.resolution_factor)
    mu_local = supergrid.max_extinction[supergrid_cell] * majorant_factor
    
    while is_inside_grid(current_voxel, grid_bounds):
        # Distance to next supergrid boundary
        # Note: Supergrid boundaries are spaced by R * voxel_size
        supergrid_step = R * voxel_size
        t_max_super = compute_supergrid_boundary_distance(ray_origin, ray_direction, current_voxel, supergrid_step, step)
        
        # Sample null collision
        u = random_uniform()
        t_null = -log(u) / mu_local
        
        if t_null < t_max_super:
            # Interaction or null collision within current supergrid cell
            t_exit = t_entry + t_null
            yield null_collision_or_interaction(t_exit, mu_local, supergrid)
            
            # Update ray position for next step
            ray_origin = ray_origin + ray_direction * t_null
            t_entry = t_exit
            
            # Re-evaluate supergrid cell if position changed significantly, 
            # or simply keep mu_local if still in same cell. 
            # For simplicity, assume we re-check cell after small steps or use DDA to advance.
            # In fused loop, we typically advance via DDA to next cell boundary if t_null is large.
            # Here, if t_null is small, we might stay in same cell. 
            # To keep loop simple, we advance ray and re-calculate supergrid cell.
            supergrid_cell = get_supergrid_cell(get_voxel_index(ray_origin, voxel_size), supergrid.resolution_factor)
            mu_local = supergrid.max_extinction[supergrid_cell] * majorant_factor
        else:
            # Advance to next supergrid boundary
            # Find which axis hits the supergrid boundary first
            min_idx = argmin(t_max_super) # Logic adapted from DDA
            t_exit = t_max_super[min_idx]
            
            yield null_collision(t_entry + t_exit) # Or just advance
            
            # Update DDA state to next supergrid boundary
            # This requires aligning DDA steps to supergrid resolution
            current_voxel[min_idx] += step[min_idx]
            t_max[min_idx] += t_delta[min_idx]
            t_entry = t_entry + t_exit
            
            supergrid_cell = get_supergrid_cell(current_voxel, supergrid.resolution_factor)
            mu_local = supergrid.max_extinction[supergrid_cell] * majorant_factor
```

**Edge Cases:**
- **Homogeneous Medium:** The 'homogeneous' medium type explicitly rejects the supergrid parameter because it cannot honour the spatially varying majorant [S1, unofficial].
- **Zero Extinction:** If $\mu_t$ goes to zero, the derivative involving $\mu_s / \mu_t$ can go unstable [S1, unofficial].

## Subtopics

### [Progressive Null-Tracking with Super-Voxel Grid](23124-progressive-null-tracking-with-super-voxel-grid.md)
A variant that progressively estimates and updates spatially varying bounding extinctions during rendering, making null-tracking resilient to non-bounding extinctions. It builds on [Majorant Supergrid Traversal](1967-majorant-supergrid-traversal.md) by allowing the bounding extinction to adapt to local density variations rather than relying on a single global worst-case bound. Unlike progressive photon mapping or many-light methods, which introduce bias in every iteration (e.g., from blurring or loss of short-range transport), this approach's individual passes become unbiased once a bounding extinction is reached.

## Operating characteristics
- **Performance:** Supergrid traversal is faster than the global majorant approach because it avoids paying `optixTrace` for every null collision, which `prbvolpath` does in its symbolic-loop iteration [S1, unofficial]. In measurements, rendering the same frame at 64 spp with no AD graph, `prbvolpath` took 104.5 s while the supergrid plugin took 72.1 s [S1, unofficial].
- **Memory:** The supergrid adds memory overhead proportional to the downsampled grid size. The archived branch notes that a slot per segment costs about 4.6 GB more than `prbvolpath` at a specific wavefront, while one slot per lane costs 32 MiB more [S1, unofficial].
- **Limitations:** Not applicable to homogeneous media [S1, unofficial]. Stability issues may arise if $\mu_t$ approaches zero without proper handling of the $\mu_s / \mu_t$ ratio [S1, unofficial].

## Background
The method was introduced in the context of Mitsuba 3's sample matching for extinction gradients [S1, unofficial]. It was implemented to optimize null-collision sampling in heterogeneous media by fusing the DDA traversal with the null-collision walk in C++ [S1, unofficial].

## Sources
[S1] Sample matching for extinction gradients (review branch), https://github.com/AuroraRyan0301/mitsuba3/pull/1, fetched 2026-10-26 (unofficial)
