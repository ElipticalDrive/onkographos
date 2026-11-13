# Volumetric Light Scattering via Froxel Clustering

Also called: scatter local lights in the volumetric fog volume

## Overview
Volumetric Light Scattering via Froxel Clustering is a real-time rendering technique that computes light scattering for local lights (point and spot) within a participating media volume by evaluating contributions at the midpoints of a 3D froxel grid [S1, unofficial]. It differs from [Froxel-based Volumetric Rendering](1919-froxel-based-volumetric-rendering.md) by explicitly clustering lights per froxel to optimize the evaluation of the scattering integral for multiple local sources, while skipping the sun and non-volumetric lights [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A 3D froxel grid representing the view frustum, a list of scene lights, the camera's view/projection matrices, and fog parameters (albedo, anisotropy).
*   **Outputs:** The accumulated scattered radiance for each froxel.

**Data Structures:**
*   **Light Struct:** Contains a boolean flag `LIGHT_VOLUMETRIC`. Constructors for directional, point, and spot lights set this to `true`. A zero-initialized `Light` struct has this flag `false` [S1, unofficial].
*   **Froxel Grid:** A regular 3D grid partitioning the view frustum. Each cell stores a compact offset into a global flat light list [S1, unofficial].
*   **Light List:** A global buffer containing indices of lights affecting specific froxels.

**Algorithm Steps:**
1.  **Grid Construction:** Partition the view frustum into froxels [S1, unofficial].
2.  **Light Assignment:** Assign lights to froxels based on spatial overlap [S1, unofficial].
3.  **Shader Execution (Per Froxel):**
    a.  Determine the froxel's midpoint in world space.
    b.  Perform a cluster lookup to retrieve the list of lights affecting this froxel. If the froxel is outside the cluster grid or the cell is overflowing, walk the complete flat light list [S1, unofficial].
    c.  Initialize accumulated radiance to zero.
    d.  Iterate through the retrieved light list:
        i.  Check if the light is the sun (identified by index + 1 == sun_light index). If so, skip it, as the sun is handled separately [S1, unofficial].
        ii. Check if the light has the `LIGHT_VOLUMETRIC` flag set. If not, skip it [S1, unofficial].
        iii. Check if the light's `sample_light` radiance is zero. If so, skip it [S1, unofficial].
        iv. For remaining lights, compute the scattering contribution using the formula below.
4.  **Accumulation:** Sum the contributions from all valid lights.

**Formula:**
The contribution $C$ of a single light is calculated as:
$$ C = \text{albedo} \times \text{HG}(\text{dot}(\text{ray}, \text{light\_dir}), \text{fog\_anisotropy}) \times \text{radiance} \times \text{visibility} $$

Where:
*   $\text{albedo}$: The fog's albedo (scattering color).
*   $\text{HG}(\mu, g)$: The Henyey-Greenstein phase function, where $\mu = \text{dot}(\text{ray}, \text{light\_dir})$ and $g$ is $\text{fog\_anisotropy}$ [S1, unofficial].
*   $\text{radiance}$: The light's radiance value.
*   $\text{visibility}$: $1.0$ if the light does not cast shadows. If the light casts shadows, visibility is determined by a single shadow sample taken with a zero normal vector ($\vec{n} = \vec{0}$) [S1, unofficial].

**Parameters:**
*   `fog_anisotropy`: The anisotropy parameter for the Henyey-Greenstein phase function. Typical values depend on the desired fog density and scattering directionality [S1, unofficial].
*   `LIGHT_VOLUMETRIC`: Boolean flag controlling whether a light contributes to volumetric scattering [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_volumetric_scattering(froxel_midpoint, froxel_cell, cluster_grid, flat_light_list, sun_light_index, fog_albedo, fog_anisotropy):
    # Step 1: Retrieve light list
    if froxel_cell is within cluster_grid bounds and froxel_cell is not overflowing:
        light_indices = cluster_grid[froxel_cell].get_light_list()
    else:
        light_indices = flat_light_list # Walk complete list

    accumulated_radiance = vec3(0.0)

    # Step 2: Iterate lights
    for light_idx in light_indices:
        light = get_light(light_idx)

        # Skip Sun
        if light_idx + 1 == sun_light_index:
            continue

        # Skip non-volumetric lights
        if not light.flags.LIGHT_VOLUMETRIC:
            continue

        # Skip zero radiance lights
        if light.sample_light().radiance == 0.0:
            continue

        # Compute Contribution
        ray_dir = normalize(froxel_midpoint - camera_position) # Assuming ray from camera to froxel
        light_dir = normalize(light.position - froxel_midpoint)
        
        dot_product = dot(ray_dir, light_dir)
        hg_phase = heney_greenstein(dot_product, fog_anisotropy)
        
        radiance = light.sample_light().radiance
        
        # Visibility: 1 if no shadow, else 1 shadow sample with zero normal
        if light.casts_shadow:
            visibility = sample_shadow(light, froxel_midpoint, vec3(0.0))
        else:
            visibility = 1.0

        contribution = fog_albedo * hg_phase * radiance * visibility
        accumulated_radiance += contribution

    return accumulated_radiance
```

**Edge Cases and Pitfalls:**
*   **Opting Out:** Setting `LIGHT_VOLUMETRIC` to false saves radiance and shadow sample computation but does not save the cost of walking the per-light list [S1, unofficial].
*   **Fallback Performance:** Froxels outside the cluster grid fall back to a flat light list. The percentage of such froxels increases with `far_distance`. For a 60° vertical FOV and near plane 0.1, at `far_distance` 200, 22.6% of froxels fall outside the grid; at 500, 51.0%; at 1000, 65.4% [S1, unofficial].
*   **Sun Handling:** The sun is explicitly skipped in the loop to avoid double-counting, as it is added separately [S1, unofficial].

## Subtopics

### [Froxel Light Assignment](13184-froxel-light-assignment.md)
The core data-structure preparation step that maps scene lights to a 3D grid of froxels by determining spatial associations and computing compact offsets for variable-length light lists. Choose this when optimizing the build phase of clustered lighting for participating media.

### [Froxel Grid Construction](13185-froxel-grid-construction.md)
The process of partitioning the 3D view frustum into a regular 3D grid of voxels (froxels) to spatially organize lights. Choose this when defining the spatial discretization for clustered volumetric rendering, specifically targeting participating media by evaluating at froxel midpoints.

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** Complexity is proportional to the number of froxels multiplied by the average number of lights per froxel. At 1440p on an RTX 4090, 64 lamps at a default 14m range cost approximately 0.0983ms, comparable to 4 lamps everywhere (0.0829ms) [S1, unofficial]. On lavapipe (1280x720), 64 lamps in Layout A cost 19.56ms, dropping to 9.89ms if opted out [S1, unofficial].
*   **Memory:** Requires storage for the froxel grid, cluster offsets, and the flat light list. The `LightGpu` structure size remains unchanged as `LIGHT_VOLUMETRIC` is packed into existing flags [S1, unofficial].

**Scaling:**
*   Performance scales with the number of lights and the density of the froxel grid. Increasing `far_distance` significantly increases the number of froxels falling back to the flat list, degrading performance if not managed by increasing `max_distance` [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Efficiently handles multiple local lights in volumetric fog using existing clustered lighting infrastructure. Low overhead for scenes with few volumetric lights.
*   **Limitations:** Fallback to flat lists for distant froxels can become a bottleneck. Does not handle the sun (handled separately). Opting out of scattering does not reduce list traversal cost [S1, unofficial].

**When to Use:**
*   Use for real-time volumetric fog with multiple local lights (point/spot) where performance is critical and the sun is handled via a separate pass.
*   Avoid when the scene requires complex multiple scattering solutions or when the majority of froxels fall outside the cluster grid due to large `far_distance` settings [S1, unofficial].

## Background
This method was introduced as part of a feature update to scatter local lights in the volumetric fog volume, specifically addressing issue #357 [S1, unofficial]. It was implemented by fesoliveira014 and merged into the main branch of the c3d.c3l repository on October 12, 2026 [S1, unofficial]. The implementation builds on existing clustered lighting principles and integrates with the engine's light serialization (version 2) and GPU data structures [S1, unofficial].

## Sources
[S1] Render: scatter local lights in the volumetric fog volume, https://github.com/fesoliveira014/c3d.c3l/pull/458, fetched 2026-10-26 (unofficial)
