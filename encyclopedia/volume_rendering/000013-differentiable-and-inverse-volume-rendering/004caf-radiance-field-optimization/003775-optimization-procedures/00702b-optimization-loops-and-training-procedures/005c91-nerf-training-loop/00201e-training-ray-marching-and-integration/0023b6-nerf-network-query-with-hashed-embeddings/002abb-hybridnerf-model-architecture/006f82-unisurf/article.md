# UNISURF

Also called: UniSurf, Unifying Implicit Surfaces and Radiance Fields, Unifying NeRFs with implicit surfaces

## Overview
UNISURF is a hybrid neural representation that unifies volume rendering (radiance fields) and surface rendering (implicit surfaces) to achieve high-quality geometry and sharp textures without requiring mask supervision [S3, unofficial]. It improves upon standard NeRF by replacing the density function with an occupancy network, offering a more robust representation of surface geometry [S2]. Unlike pure surface rendering which requires mask supervision and suffers from blurry texture mapping, UNISURF provides high-quality novel views and sharp colors by leveraging volume rendering in early optimization stages and surface rendering in later stages [S3, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of posed RGB images $\{I_i\}$, camera poses, and intrinsic parameters.
- **Outputs:** A trained occupancy field network $f_{occ}$ and a color network $f_{color}$, and a reconstructed 3D mesh extracted via marching cubes.

**Data Structures:**
- **Occupancy Network ($f_{occ}$):** A multi-layer perceptron (MLP) that takes a 3D coordinate $\mathbf{x}$ as input and outputs a scalar occupancy value $o \in [0, 1]$. The occupancy at the $i^{th}$ sample corresponds to an indicator of surface presence [S3, unofficial].
- **Color Network ($f_{color}$):** An MLP that takes the 3D coordinate $\mathbf{x}$ and the viewing direction $\mathbf{d}$ as input and outputs the view-dependent color $\mathbf{c}$.

**Method Steps:**
1. **Ray Sampling with Root Finding:** For a given ray, UNISURF treats the volume as an occupancy field. It uses root finding to locate a "surface" intersection point where the occupancy transitions from 0 to 1 [S1, unofficial].
2. **Interval Sampling:** Volume rendering samples are distributed only within a specific interval around the found surface intersection point [S1, unofficial].
3. **Progressive Interval Narrowing:** The width of this sampling interval monotonically decreases over the course of optimization [S1, unofficial].
   - **Early Stage:** The wide interval allows volume rendering to supervise the whole training volume, enabling optimization without mask supervision [S3, unofficial].
   - **Late Stage:** The narrow interval allows the method to more efficiently refine the surface with tightly spaced samples, transitioning towards surface rendering to extract level-set surfaces [S3, unofficial].
4. **Rendering:** The final pixel color is computed by integrating the color and occupancy along the sampled points within the interval.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray(ray_origin, ray_direction, t_near, t_far, network_occ, network_color, t_surface, interval_width):
    """
    Renders a single ray using the UNISURF hybrid approach.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, normalized direction of the ray.
        t_near: float, near plane distance.
        t_far: float, far plane distance.
        network_occ: Occupancy MLP, returns occupancy value for a 3D point.
        network_color: Color MLP, returns color for a 3D point and viewing direction.
        t_surface: float, the parameter along the ray where the surface intersection was found.
        interval_width: float, the width of the sampling interval around t_surface.
        
    Returns:
        color: 3D vector, the rendered color.
        depth: float, the expected depth.
    """
    # Define the sampling interval around the surface intersection
    t_start = max(t_near, t_surface - interval_width / 2)
    t_end = min(t_far, t_surface + interval_width / 2)
    
    # Number of samples within the interval
    num_samples = 32  # Typical value, not explicitly given in sources [G]
    
    # Sample points uniformly within the interval
    t_samples = linspace(t_start, t_end, num_samples)
    points = ray_origin + t_samples[:, None] * ray_direction
    
    # Query occupancy and color
    # Note: In practice, occupancy might be queried with a sigmoid or similar activation
    # to map raw network output to [0,1]. The sources describe it as an "occupancy field"
    # where values indicate surface presence. We assume the network outputs raw logits
    # or probabilities. Standard NeRF uses sigmoid for density. UNISURF uses occupancy.
    # We assume network_occ outputs values in [0,1] or uses a sigmoid activation.
    # If raw logits, apply sigmoid. Sources do not specify activation, but "occupancy"
    # implies [0,1]. We assume the network is trained to output occupancy directly or
    # via sigmoid. Let's assume sigmoid for safety as is common in occupancy networks.
    occupancy_raw = network_occ(points)
    occupancy = sigmoid(occupancy_raw)
    
    color = network_color(points, ray_direction)
    
    # Compute transmittance and alpha
    # Standard volume rendering formula:
    # alpha_i = 1 - exp(-sigma_i * delta_t)
    # In UNISURF, occupancy acts as a proxy for density/surface presence.
    # The sources state it unifies surface and volume.
    # A common interpretation for occupancy-based rendering is:
    # alpha_i = occupancy_i (if occupancy is treated as probability of surface presence in the voxel)
    # However, standard volume rendering requires density.
    # UNISURF paper [S1, unofficial] suggests using occupancy to define the interval and then rendering.
    # The exact density-to-occupancy mapping is not detailed in the provided notes.
    # We assume a direct mapping or that occupancy is used as alpha for simplicity in this interval.
    # Given the note "occupancy ... corresponds to an indicator of surface presence",
    # we treat occupancy as alpha for the samples within the interval.
    alpha = occupancy
    
    # Volume rendering integration
    T = 1.0
    rendered_color = zeros(3)
    rendered_depth = 0.0
    total_alpha = 0.0
    
    for i in range(num_samples):
        delta_t = t_samples[i] - (t_samples[i-1] if i > 0 else t_start)
        # If using occupancy as alpha directly:
        # alpha_i = occupancy[i]
        # T_i = T_{i-1} * (1 - alpha_i)
        # color_i = T_i * alpha_i * color[i]
        
        # Standard NeRF density integration requires sigma.
        # If occupancy is used as a proxy for density, one might use:
        # sigma = -log(1 - occupancy) / delta_t
        # But sources don't specify. We stick to the high-level description:
        # "distributes volume rendering samples only within an interval"
        # and "occupancy ... indicator of surface presence".
        # We will use occupancy as alpha for the samples in the interval.
        
        T = T * (1 - alpha[i])
        rendered_color += T * alpha[i] * color[i]
        rendered_depth += T * alpha[i] * t_samples[i]
        total_alpha += T * alpha[i]
        
    # Normalize depth by total alpha to get expected depth
    if total_alpha > 0:
        rendered_depth /= total_alpha
    else:
        rendered_depth = t_surface # Fallback to surface intersection
        
    return rendered_color, rendered_depth

def train_step(images, poses, network_occ, network_color, t_surface_estimator, interval_width_scheduler):
    """
    Training step for UNISURF.
    """
    # 1. Estimate surface intersection for each ray using root finding on occupancy field
    # This is a simplified representation. Root finding requires querying the network
    # along the ray to find the zero-crossing of occupancy (or transition from 0 to 1).
    t_surfaces = []
    for ray in rays:
        t_surf = find_root(lambda t: network_occ(ray.origin + t * ray.direction), t_near, t_far)
        t_surfaces.append(t_surf)
        
    # 2. Get current interval width
    current_width = interval_width_scheduler.get_width()
    
    # 3. Render images using the hybrid sampling strategy
    rendered_images = []
    for i, ray in enumerate(rays):
        color, depth = render_ray(
            ray.origin, ray.direction, t_near, t_far,
            network_occ, network_color,
            t_surfaces[i], current_width
        )
        rendered_images.append(color)
        
    # 4. Compute loss against ground truth images
    loss = compute_loss(rendered_images, images)
    
    # 5. Backpropagate and update networks
    loss.backward()
    optimizer.step()
    
    # 6. Decrease interval width for next iteration
    interval_width_scheduler.decrease()
```

**Parameters:**
- **Interval Width:** Starts wide and monotonically decreases [S1, unofficial]. Typical values not given in sources.
- **Number of Samples:** Not given in sources [G].

**Edge Cases and Pitfalls:**
- **No Eikonal Regularization:** UNISURF does not leverage Eikonal regularization because its underlying model is an occupancy network, not a Signed Distance Field (SDF) [S2].
- **Surface Extraction:** Surfaces extracted from neural densities (using marching cubes) can be noisy; UNISURF addresses this by using the occupancy network for a more robust representation [S2].

## Operating characteristics
- **Time Complexity:** Not given in sources.
- **Memory Complexity:** Not given in sources.
- **Strengths:** High-quality geometry and sharp textures without mask supervision [S3, unofficial]. More robust surface representation than standard NeRF densities [S2].
- **Limitations:** Does not use Eikonal regularization, which may limit geometric smoothness compared to SDF-based methods like NeuS [S2].
- **When to use:** When high-quality geometry and sharp textures are needed without access to mask labels.
- **When not to use:** When Eikonal regularization is desired for smoother surfaces, or when SDF-based methods are preferred.

## Background
UNISURF was proposed by Oechsle et al. in ICCV 2021 [S3, unofficial]. It addresses the issue that surfaces extracted from standard NeRFs are not accurate and depend heavily on the threshold used for extraction [S3, unofficial]. By unifying implicit surfaces and radiance fields, it aims to provide the best of both worlds: the high-quality novel views of volume rendering and the clear surface definition of implicit surfaces [S3, unofficial].

## Sources
[S1] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S2] Geometric implicit neural representations for signed distance functions, https://doi.org/10.1016/j.cag.2024.104085, fetched 2026-10-26
[S3] Lecture 7 – Neural Radiance Fields Prof. Dr., https://virtualhumans.mpi-inf.mpg.de/3DVision25/slides25/pdf/Lecture_07_Neural_Radiance_Fields.pdf, fetched 2026-10-27 (unofficial)
