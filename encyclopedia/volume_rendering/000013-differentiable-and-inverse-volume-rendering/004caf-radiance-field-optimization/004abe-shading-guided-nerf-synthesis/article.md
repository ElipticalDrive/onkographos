# Shading-Guided NeRF Synthesis

Also called: Shading Map Guided Synthesis Pipeline, Shading vector synthesis

## Overview
Shading-Guided NeRF Synthesis is a method for generating large-scale radiance fields from a smaller exemplar by controlling lighting effects using shading vectors, avoiding the need to decouple geometry, material, and lighting [S1]. It solves the problem of efficiently synthesizing repetitive 3D structures while maintaining consistent lighting across complex scenes [S1]. It sets itself apart from methods that rely solely on feature grid matching, which leads to discontinuous shading changes under complex lighting, by explicitly incorporating a shading map and shading map guider to ensure continuous lighting [S1].

## How it works
**Inputs:**
1.  **Exemplar Radiance Field:** Represented by a density field $V^{(d)}$ and a color feature field $V^{(f)}$ [S1].
2.  **Exemplar Shading Map:** A 3D volume where each voxel column stores a shading vector derived from the exemplar [S1].
3.  **Target Scene Geometry:** A geometrically continuous large scene (synthesized via geometry-based synthesis) used to generate the shading map guider [S1].
4.  **Light Source Information:** Position $x_l$ and intensity $I$ of point light sources in the target scene [S1].
5.  **Parameters:**
    *   $M$: Number of viewpoints to approximate the shading vector (typically 50) [S1].
    *   $k_s$: Number of closest shading patches to find in the exemplar [S1].
    *   $k_g$: Number of candidate patches with geometric continuity [S1].
    *   $n_p$: Number of high-density points selected for ray tracing [S1].
    *   $n_s$: Number of sample points along the light path [S1].
    *   $n_c$: Number of shading vector channels retained (those with the lowest missing value rate) [S1].

**Outputs:**
*   A synthesized large-scale NeRF scene with density field and radiance field stitched from exemplar patches, guided by the target lighting [S1].

**Data Structures:**
*   **Shading Map:** A 3D volume. Each voxel at position $v$ contains a vector $\mathbf{s}_v \in \mathbb{R}^M$, where $M$ is the number of viewpoints [S1]. The $m$-th component is the shading value of the voxel column at viewpoint $m$ [S1].
*   **Shading Map Guider:** A 3D volume of the same dimensions as the Shading Map, containing shading vectors derived from the target scene's geometry and light sources [S1].
*   **Density Field $V^{(d)}$:** 3D grid storing density values $\sigma$ [S1].
*   **Color Feature Field $V^{(f)}$:** 3D grid storing color feature values [S1].

**Algorithm Steps:**

1.  **Extract Exemplar Shading Map:**
    a.  Perform intrinsic decomposition on multiple images of the exemplar using PIE-Net to derive shading and albedo [S1].
    b.  Apply Gaussian blurring and polynomial fitting to the shading image to smooth it and remove geometric interference [S1].
    c.  For each pixel in the smoothed shading image, project the shading value along the camera ray to a depth position $d$ in the 3D voxel grid [S1].
    d.  Calculate depth $d$ using the density grid:
        $$ d = \sum_{k} T_i (1 - \exp(-\sigma_i \delta_i)) t_i $$
        where $T_i$ is transmittance, $\sigma_i$ is density, $\delta_i$ is distance, and $t_i$ is a position/weight factor [S1].
    e.  Collect shading information from $M$ viewpoints to form the shading vector for each voxel column [S1].
    f.  **Post-processing:**
        i.  Fill holes in shading vector channels (caused by occlusions/limited pixels) using KNN interpolation with inverse distance weighting [S1].
        ii. Repair noisy points using median filtering [S1].
        iii. Retain only the $n_c$ channels with the lowest rate of missing values [S1].

2.  **Generate Shading Map Guider:**
    a.  Synthesize a geometrically continuous large scene using geometry-based synthesis (matching density only) to create a base target scene. Then, apply ray tracing to generate the shading map guider for the target illumination [S1].
    b.  Select $n_p$ points from this scene with high density [S1].
    c.  For each selected point $x$ and a point light source $x_l$:
        i.  Sample $n_s$ points $x_1, ..., x_{n_s}$ along the path between $x_l$ and $x$, where $x_{n_s} = x$ [S1].
        ii. Calculate the probability $p_{n_s}$ of the ray propagating to $x_{n_s}$:
            $$ p_{n_s} = T_{n_s} (1 - \exp(-\sigma_{n_s} \delta_{n_s})) $$
            [S1].
        iii. Calculate brightness $b$:
            $$ b = p_{n_s} \frac{1}{n_s} \frac{I}{\|x - x_l\|_2^2} $$
            where $I$ is light intensity [S1].
    d.  The brightness values from $M$ viewpoints (or equivalent lighting configurations) form the shading vector for each voxel column in the target scene, creating the Shading Map Guider [S1].

3.  **Synthesize Scene (Patch Selection and Stitching):**
    a.  For a patch $P_n$ to be synthesized in the target area:
        i.  Compute the L2 distance between the shading vector of $P_n$ and patches in the exemplar's Shading Map [S1].
        ii. Accelerate search using a kd-tree [S1].
        iii. Identify the $k_s$ closest shading patches from the exemplar [S1].
        iv. Normalize the values of the exemplar's Shading Map and the target's Shading Map Guider to the same scale [S1].
        v.  Rotate the exemplar by 90, 180, and 270 degrees to obtain additional available patches, addressing selection limitations [S1].
    b.  **Two-Phase Patch Selection:**
        i.  **Phase 1 (Geometry):** From the $k_s$ candidates, find $k_g$ candidate patches with geometric continuity based on the density field $V^{(d)}$ [S1].
        ii. **Phase 2 (Color):** From the $k_g$ candidates, select one patch for color continuity in the overlap area based on the color feature field $V^{(f)}$ [S1].
    c.  Stitch the selected patch’s density field and radiance field onto the target area [S1].
    d.  Repeat until the target scene is fully synthesized [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def shading_guided_nerf_synthesis(exemplar_shading_images, exemplar_density, target_density, light_sources, M, k_s, k_g, n_p, n_s, n_c):
    # 1. Extract Exemplar Shading Map
    # [G] Intrinsic decomposition to derive shading and albedo from images
    smoothed_shading = [smooth_shading(img) for img in exemplar_shading_images]
    
    # Initialize Shading Map volume (X, Y, Z, M) with NaN
    shading_map = np.full((X, Y, Z, M), np.nan)
    
    for m in range(M):
        camera = get_camera_params(m)
        shading_img = smoothed_shading[m]
        for y in range(shading_img.shape[0]):
            for x in range(shading_img.shape[1]):
                ray_origin, ray_direction = camera.unproject(x, y)
                # [G] Ray march through exemplar density to get samples
                samples = ray_march(ray_origin, ray_direction, exemplar_density)
                # Depth formula: d = sum_k T_i (1 - exp(-sigma_i delta_i)) t_i
                d = sum(s.transmittance * (1 - np.exp(-s.sigma * s.delta)) * s.t for s in samples)
                voxel_coords = ray_origin + d * ray_direction
                if is_inside_grid(voxel_coords):
                    shading_map[voxel_coords] = shading_img[y, x]
                    
    # Post-processing
    for m in range(M):
        channel = shading_map[:, :, :, m]
        # KNN interpolation with inverse distance weighting for holes
        channel = knn_interpolate(channel, weights='inverse_distance')
        # Median filtering for noisy points
        channel = median_filter(channel, size=3)
        shading_map[:, :, :, m] = channel
        
    # Retain n_c channels with lowest missing rate
    missing_rates = np.mean(np.isnan(shading_map), axis=(0,1,2))
    best_channels = np.argsort(missing_rates)[:n_c]
    exemplar_shading_map = shading_map[:, :, :, best_channels]
    
    # 2. Generate Shading Map Guider
    # [G] Geometry-based synthesis to create base target scene
    base_scene_density = geometry_based_synthesis(target_density)
    # Select n_p points with high density
    high_density_points = select_high_density_points(base_scene_density, n_p)
    
    shading_map_guider = np.zeros((X, Y, Z, M))
    for point in high_density_points:
        x = point
        for m in range(M):
            light_source = get_light_source(m)
            I = light_source.intensity
            x_l = light_source.position
            # Sample n_s points along path
            path_points = sample_path(x_l, x, n_s)
            x_ns = path_points[-1]
            sigma_ns = get_density(base_scene_density, x_ns)
            delta_ns = distance(x_l, x_ns) / n_s
            T_ns = calculate_transmittance(x_l, x_ns, base_scene_density)
            p_ns = T_ns * (1 - np.exp(-sigma_ns * delta_ns))
            dist_sq = np.linalg.norm(x - x_l)**2
            brightness = p_ns * (1/n_s) * (I / dist_sq)
            voxel_coords = get_voxel_coords(x)
            shading_map_guider[voxel_coords, m] = brightness
            
    # 3. Synthesize Scene
    synthesized_density = np.zeros(target_density.shape)
    synthesized_color = np.zeros(target_density.shape)
    
    for patch in target_geometry.patches:
        # Normalize scales
        exemplar_patches = normalize(exemplar_patches)
        target_shading_vec = normalize(target_shading_vec)
        
        # Find k_s closest patches using L2 distance (accelerated by kd-tree)
        distances, indices = kd_tree.query(target_shading_vec, k=k_s)
        candidate_patches = exemplar_patches[indices]
        
        # Two-phase selection
        # Phase 1: Geometric continuity based on density field
        geometric_candidates = filter_by_density_continuity(candidate_patches, target_area_patch, k_g)
        # Phase 2: Color continuity based on color feature field
        best_patch_idx = select_by_color_continuity(geometric_candidates, target_area_patch)
        
        # Stitch patch
        patch_density = extract_patch_density(exemplar_density, best_patch_idx)
        patch_color = extract_patch_color(exemplar_color, best_patch_idx)
        stitch(synthesized_density, synthesized_color, patch, patch_density, patch_color)
        
    return synthesized_density, synthesized_color
```

**Edge Cases and Pitfalls:**
*   **Holes in Shading Map:** Occlusions and limited pixels cause missing values in shading vector channels. Addressed by KNN interpolation with inverse distance weighting [S1].
*   **Noisy Points:** Inaccurate depth values introduce noise. Addressed by median filtering [S1].
*   **Channel Selection:** Channels with high missing value rates have high interpolation error. Only retain $n_c$ channels with the lowest missing rate [S1].
*   **Selection Limitations:** Strict shading/geometry/texture requirements may limit patch selection. Addressed by rotating the exemplar by 90, 180, and 270 degrees to increase available patches [S1].
*   **Scale Mismatch:** Exemplar and target shading maps may have different scales. Must be normalized to the same scale before distance calculation [S1].

## Subtopics

### [NeRF Synthesis via DVGO](12848-nerf-synthesis-via-dvgo.md)
NeRF Synthesis via DVGO is a method for generating large-scale radiance fields by applying 2D texture synthesis algorithms to the flattened density and feature grids of a Differentiable Voxel Grid Object (DVGO) representation. It solves the problem of efficiently synthesizing repetitive 3D structures (e.g., vegetation) by exploiting the observation that only two spatial dimensions typically exhibit clear repeatability, thereby avoiding the inefficiency of full 3D synthesis. It sets itself apart from direct 3D synthesis by flattening the 3D grid along the Z-axis into 2D grids, allowing the use of established 2D texture synthesis techniques while treating each unit as storing density and feature values rather than RGB pixels.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources. Requires storage for the exemplar density/color fields, the exemplar shading map, and the target shading map guider [S1].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:**
    *   Controls lighting effects using shading guidance without decoupling geometry, material, and lighting [S1].
    *   Ensures continuous lighting in synthesized scenes under complex lighting [S1].
    *   Can synthesize new scenes on curved surfaces with arbitrary lighting effects [S1].
    *   Uses a two-phase method to ensure both geometric and appearance continuity [S1].
*   **Limitations:**
    *   Requires intrinsic decomposition (PIE-Net) and ray tracing for preprocessing [S1].
    *   Sensitive to occlusions and limited pixels, requiring post-processing [S1].
    *   Relies on the availability of multiple images of the exemplar for intrinsic decomposition [S1].
*   **When to use:** When synthesizing large-scale scenes with repetitive structures and complex lighting, where maintaining consistent lighting is critical [S1].

## Background
*   **Origin:** The method was proposed in "NeRF Synthesis with Shading Guidance" [S1].
*   **History:** The paper addresses the limitations of physically-based rendering (PBR) in virtual reality, augmented reality, and video games, and the challenges of attaching realistic surface structures onto mesh surfaces [S1]. It builds on the concept of 3D texture synthesis and NeRF representations [S1].
*   **Usage:** Demonstrated for generating high-quality results with consistent geometry and appearance for scenes with complex lighting, including synthesis on curved surfaces [S1].

## Sources
[S1] NeRF Synthesis with Shading Guidance, https://link.springer.com/chapter/10.1007/978-981-99-9666-7_16, fetched 2026-10-26
