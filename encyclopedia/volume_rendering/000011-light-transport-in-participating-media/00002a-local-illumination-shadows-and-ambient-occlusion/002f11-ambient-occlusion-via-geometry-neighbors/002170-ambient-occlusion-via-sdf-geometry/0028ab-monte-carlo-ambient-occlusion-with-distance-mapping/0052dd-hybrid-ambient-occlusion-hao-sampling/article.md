# Hybrid Ambient Occlusion (HAO) Sampling

Also called: Jittered ray sampling with 3D noise, Monte-Carlo integration over hemisphere, Monte-Carlo integration over hemisphere with noise jitter

## Overview
Hybrid Ambient Occlusion (HAO) is a screen-space technique that computes ambient occlusion by performing Monte-Carlo integration over the hemisphere of each pixel to evaluate the occlusion factor [S1]. It distinguishes itself from standard hemisphere sampling by dynamically jittering ray directions using a space-dependent 3D noise texture, which prevents flickering during camera movement, and by using a fixed number of uniform steps for grid occupancy testing to optimize GPU performance [S1].

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   Pixel position $p$ and surface normal $n_i$ (from a G-buffer or depth/normal reconstruction).
    *   A 3D white noise texture $t$ tiling the entire space.
    *   A voxelized 3D proxy of the scene geometry.
    *   Parameters: number of rays $n$, spread angle $\theta_{spread}$, maximum distance $d_{max}$, and number of steps per ray $k$.
*   **Outputs:**
    *   Ambient Occlusion value $A(p, n_i)$ for each pixel.

### Data Structures
*   **Voxel Grid:** A 3D grid representing the scene proxy. Occupancy is binary (filled/empty).
*   **Noise Texture:** A 3D texture $t(x, y, z)$ containing white noise values, typically stored as floating-point or normalized integer data.

### Algorithm Steps

1.  **Ray Distribution:** Distribute $n$ ray directions $\{\omega_1, ..., \omega_n\}$ over the hemisphere defined by the surface normal $n_i$. The distribution is uniform within a cone of open angle $\theta_{spread}$ (the "spread" parameter) [S1].
2.  **Jittering:** For each base ray direction $\omega(p, r)$, sample a jitter offset $\delta$ from the 3D noise texture $t$ using the pixel's world position and normal as coordinates. Shift the ray direction to $\omega(p, r')$ using this offset [S1].
    $$ \omega(p, r') = \text{shift}(\omega(p, r), \delta) $$
    *Note: The exact implementation of `shift` depends on the coordinate system but effectively perturbs the vector based on the noise value [S1].*
3.  **Uniform Step Testing:** For each jittered ray, test grid occupancy at $k$ uniformly spaced points along the ray, rather than marching cell-by-cell. The points are distributed over the distance $d_{max}$ [S1].
4.  **Weighting:** If a filled voxel cell is found at any of the $k$ steps, the result is weighted by a distance function $w(V, d_{max})$ and the dot product between the ray direction and the surface normal. This creates a progressive penumbra [S1].
5.  **Integration:** Sum the weighted results and divide by $n$ to obtain the final AO value [S1].

### Formula
The ambient occlusion value $A(p, n_i)$ is computed as:

$$ A(p, n_i) = \frac{1}{n} \sum_{\omega_i \in \{\omega_1, ..., \omega_n\}} w(V(p, \omega_i), d_{max}) \langle \omega_i, n_i \rangle $$

Where:
*   $n$ is the number of rays.
*   $\omega_i$ is the $i$-th jittered ray direction.
*   $n_i$ is the surface normal at pixel $p$.
*   $V(p, \omega_i)$ is the visibility result (binary occupancy) along ray $\omega_i$ starting at $p$.
*   $w(V, d_{max})$ is a weighting function based on distance. If $V$ is occluded, the weight depends on the distance to the occluder up to $d_{max}$ [S1].
*   $\langle \omega_i, n_i \rangle$ denotes the dot product (cosine of the angle) between the ray and the normal [S1].

### Parameters
*   **$n$ (Number of rays):** Controls sampling density. Typical values range from 16 to 128 rays per pixel [S1]. Higher values reduce noise but increase cost.
*   **$d_{max}$ (Maximum distance):** The cutoff distance for the weighting function $w$. Values beyond this distance are considered unoccluded or fully occluded depending on the specific weighting function implementation [S1].
*   **$\theta_{spread}$ (Spread):** The open angle within which rays are distributed. Influences contrast and smoothness. Typical values are 0.25, 0.5, or 0.75 (radians or normalized) [S1].
*   **$k$ (Steps per ray):** A fixed, typically small number of steps (e.g., 8) used to test grid occupancy [S1].

### Edge Cases and Pitfalls
*   **Noise from Fixed Steps:** Using a fixed number of steps $k$ instead of marching cell-by-cell can cause rays passing near the corners of non-empty cells to miss them, producing more noise in the AO map [S1]. However, this simplification speeds up sampling and produces visually smooth results after filtering [S1].
*   **Flickering:** Without the 3D noise jitter, hemisphere sampling can cause flickering when the camera moves. The space-dependent noise texture avoids this [S1].
*   **AO Bleeding:** The filtering stage (see below) must be designed carefully to avoid AO bleeding between distant objects that are projected closely in image space. Using object-space entities for the range space of the bilateral filter prevents this [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_hao(p_world, n_normal, voxel_grid, noise_texture, n_rays, d_max, spread_angle, steps_per_ray):
    """
    Computes Hybrid Ambient Occlusion for a single pixel.
    
    Args:
        p_world: World position of the pixel.
        n_normal: Normalized surface normal at the pixel.
        voxel_grid: 3D grid for occupancy tests.
        noise_texture: 3D texture for jittering.
        n_rays: Number of rays to cast.
        d_max: Maximum distance for occlusion weighting.
        spread_angle: Open angle for ray distribution.
        steps_per_ray: Fixed number of steps along each ray.
        
    Returns:
        AO value between 0 (fully occluded) and 1 (fully open).
    """
    total_weight = 0.0
    
    # 1. Generate base ray directions uniformly in the hemisphere cone
    base_directions = generate_uniform_hemisphere_samples(n_rays, n_normal, spread_angle)
    
    for i in range(n_rays):
        omega_base = base_directions[i]
        
        # 2. Jitter ray direction using 3D noise
        # Sample noise based on position and normal
        noise_val = noise_texture.sample(p_world, n_normal)
        omega_jittered = jitter_direction(omega_base, noise_val)
        
        # 3. Test occupancy with fixed steps
        occluded = False
        min_dist = d_max
        
        for step in range(steps_per_ray):
            # Uniformly spaced points along the ray up to d_max
            t = (step + 1) / steps_per_ray * d_max
            point = p_world + omega_jittered * t
            
            # Check if point is inside a filled voxel
            if voxel_grid.is_filled(point):
                occluded = True
                # Record the distance to the first occluder found
                if t < min_dist:
                    min_dist = t
                # Note: The source implies we just need to know if it's filled
                # and weight by distance. We don't necessarily break early
                # if we want to capture multiple occluders, but typically
                # the first hit is sufficient for AO. The source says "If a 
                # filled voxel cell is found... weighted by distance".
                # We assume the first hit determines the weight.
                break
        
        # 4. Weight the result
        if occluded:
            # Weight by distance and angle
            # w(V, d_max) is a function of distance. 
            # A common simple weighting is linear falloff or similar.
            # The source does not specify the exact form of w, only that it 
            # produces progressive penumbra.
            # We assume a standard distance weighting function.
            dist_weight = distance_weight_function(min_dist, d_max)
            
            # Cosine term
            cos_angle = dot(omega_jittered, n_normal)
            
            # Ensure non-negative cosine (rays pointing into surface)
            if cos_angle > 0:
                total_weight += dist_weight * cos_angle
            else:
                # Ray points away from surface, contributes nothing to AO
                # or contributes 0 occlusion.
                pass
        else:
            # No occlusion found, contributes full light (weight 0 to occlusion sum)
            pass
            
    # 5. Average
    ao_value = 1.0 - (total_weight / n_rays)
    return ao_value

def generate_uniform_hemisphere_samples(count, normal, spread_angle):
    """
    Generates 'count' directions uniformly distributed in a cone 
    around 'normal' with opening angle 'spread_angle'.
    """
    samples = []
    for i in range(count):
        # Generate random direction in cone
        # Using standard spherical coordinate transformation relative to normal
        u = random_float()
        v = random_float()
        
        # Map to cone
        theta = sqrt(u) * sin(spread_angle)
        phi = 2 * pi * v
        
        # Construct vector in local space where normal is (0,0,1)
        local_x = cos(phi) * theta
        local_y = sin(phi) * theta
        local_z = sqrt(1 - theta**2)
        
        # Rotate to align with world normal
        direction = rotate_to_normal(local_x, local_y, local_z, normal)
        samples.append(direction)
    return samples

def jitter_direction(base_direction, noise_val):
    """
    Perturbs the base direction based on noise value.
    Implementation detail: [G]
    A simple implementation adds the noise value to the spherical 
    coordinates or adds a small random vector scaled by noise.
    For this implementation, we assume a simple perturbation.
    """
    # [G] Simple perturbation: shift the direction vector slightly
    # based on the noise value. The exact "shift" function is not 
    # specified in the source, but it must preserve the hemisphere 
    # constraint.
    perturbation = noise_val * 0.1 # Scale factor for jitter magnitude
    jittered = base_direction + Vector3(perturbation, perturbation, perturbation)
    return normalize(jittered)

def distance_weight_function(dist, d_max):
    """
    Weighting function for distance.
    [G] Common implementation is linear or quadratic falloff.
    """
    if dist > d_max:
        return 0.0
    return 1.0 - (dist / d_max)
```

## Subtopics

### Bilateral AO Map Filtering
A post-processing step that upsamples the low-resolution AO map to screen resolution while filtering noise. It uses a joint bilateral filter that considers image-space distance, object-space distance, and normal differences to prevent AO bleeding [S1].

### Voxelization Proxy
The 3D grid representation of the scene used for fast occupancy tests. It is typically generated from the scene geometry and must be updated dynamically or semi-statically [S1].

## Operating characteristics

*   **Time Complexity:** $O(N \cdot K)$ per pixel, where $N$ is the number of rays and $K$ is the number of steps per ray. The total cost is $O(R \cdot N \cdot K)$ for $R$ pixels.
*   **Memory Complexity:** $O(V)$ for the voxel grid and $O(T)$ for the noise texture, where $V$ is the voxel grid size and $T$ is the texture size.
*   **Performance:** The method is optimized for GPU execution by using a fixed number of steps, avoiding branching, and ensuring a constant number of intersection tests per ray [S1]. Generating a low-resolution AO map and upsampling it further reduces the computational load [S1].
*   **Strengths:** Fast execution on GPUs, visually smooth results after filtering, no flickering due to 3D noise jitter, and effective penumbra generation [S1].
*   **Limitations:** The fixed-step approach can introduce noise if rays miss small occluders. The quality depends heavily on the voxelization resolution and the number of rays [S1].
*   **When to use:** Real-time applications requiring approximate ambient occlusion with good visual quality and performance.
*   **When not to use:** Applications requiring precise shadowing or where the voxelization proxy cannot accurately represent the geometry.

## Background

Hybrid Ambient Occlusion was introduced by C.K. Reinbothe, T. Boubekeur, and M. Alexa in 2009 [S1]. The method combines Monte-Carlo integration with spatial noise jittering and voxel-based occupancy testing to provide a fast and visually appealing approximation of ambient occlusion for real-time rendering [S1]. It builds on the concept of hemisphere sampling but adds optimizations for GPU execution and visual stability [S1].

## Sources

[S1] Hybrid Ambient Occlusion, https://doi.org/10.2312/ega.20091008.051-057, fetched 2026-11-03
