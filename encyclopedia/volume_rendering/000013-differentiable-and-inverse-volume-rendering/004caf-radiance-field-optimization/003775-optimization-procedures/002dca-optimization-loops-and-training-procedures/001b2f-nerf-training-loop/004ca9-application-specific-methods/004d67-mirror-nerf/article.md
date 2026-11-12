# Mirror-NeRF

Also called: Mirror-NeRF: Learning Neural Radiance Fields for Mirrors with Whitted-Style Ray Tracing

## Overview
Mirror-NeRF is a neural rendering framework that learns accurate mirror geometry and reflection by unifying the radiance field and tracing rays according to Whitted-style light transport, enabling novel view synthesis and scene manipulation [S1]. Unlike NeRF and NeRFReN, which treat mirror reflections as separate virtual scenes or interpolate memorized reflections, Mirror-NeRF uses a unified radiance field with explicit reflection probability and Whitted-style ray tracing to synthesize physically consistent reflections from novel viewpoints [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of multi-view images of a scene containing mirrors, camera poses, and intrinsics.
*   **Outputs:** A rendered image for a target viewpoint, and implicit representations of geometry, density, radiance, surface normals, and reflection probability.

**Data Structures:**
The method uses a unified set of Multi-Layer Perceptrons (MLPs) to represent scene properties at any 3D position $\mathbf{x}$ and view direction $\mathbf{d}$. The fields are:
1.  **Geometry MLP ($F_{geo}$):** Input: Positional encoded position $\gamma_x(\mathbf{x})$. Output: Geometry feature vector $f_{geo}$.
2.  **Volume Density MLP ($F_\sigma$):** Input: $f_{geo}$. Output: Volume density $\sigma$.
3.  **Radiance MLP ($F_c$):** Input: $f_{geo}$ and positional encoded view direction $\gamma_d(\mathbf{d})$. Output: Radiance color $c$.
4.  **Normal MLP ($F_n$):** Input: $f_{geo}$. Output: Smoothed surface normal $\tilde{\mathbf{n}}$. This is used to avoid noise from analytical gradients of volume density [S1].
5.  **Reflection Probability MLP ($F_m$):** Input: $f_{geo}$. Output: Reflection probability $m \in [0, 1]$. This parameterizes the probability that a ray is reflected at a spatial point [S1].

**Numbered Steps:**
1.  **Ray Generation:** For each pixel in the target image, generate a camera ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the camera origin and $\mathbf{d}$ is the direction.
2.  **Sampling:** Sample $N$ points $t_i$ along the ray within the bounding volume of the scene.
3.  **Field Evaluation:** For each sample point $\mathbf{x}_i = \mathbf{r}(t_i)$:
    *   Compute geometry feature: $f_{geo} = F_{geo}(\gamma_x(\mathbf{x}_i))$.
    *   Compute volume density: $\sigma_i = F_\sigma(f_{geo})$.
    *   Compute radiance: $c_i = F_c(f_{geo}, \gamma_d(\mathbf{d}))$.
    *   Compute reflection probability: $m_i = F_m(f_{geo})$.
    *   Compute smoothed normal: $\tilde{\mathbf{n}}_i = F_n(f_{geo})$.
4.  **Ray Tracing and Reflection:**
    *   Determine if the ray hits a mirror surface. The sources indicate that physical reflection is triggered when the ray intersects a surface point with high reflection probability [S1].
    *   If a reflection occurs (based on $m_i$ and surface intersection), compute the reflected ray direction $\mathbf{d}_{refl}$ using the surface normal $\tilde{\mathbf{n}}$ and the incident direction, following Whitted-style ray tracing [S1].
    *   Trace the reflected ray to accumulate color $C_{refl}$ from the rest of the scene.
5.  **Volume Rendering Accumulation:**
    *   Compute transmittance $T_i$ and weight $\alpha_i$ for each sample:
        $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$
        $$ \alpha_i = 1 - \exp(-\sigma_i \delta_i) $$
        where $\delta_i$ is the distance between sample points [S1].
    *   The final color $\tilde{C}(\mathbf{r})$ is a blend of the direct camera ray color and the reflected ray color, weighted by the reflection probability. The source provides a formula structure:
        $$ \tilde{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i c_i $$
        *Note: The source snippet for the formula contains typos ($\mathbf{u}_{Ray_r}$, etc.) but context implies standard volume rendering accumulation over $N$ points [S1]. The blending with reflection probability $m$ is described conceptually as "blending the final image color from the camera ray color and the reflected ray color weighted by the reflection probability" [S1]. A precise implementation requires integrating $m$ into the accumulation or as a post-blend factor. Given the "unified radiance field" description, the reflection probability likely modulates the contribution of the reflected ray component against the direct component.*
6.  **Optimization:**
    *   Minimize the difference between rendered images and ground-truth images.
    *   Apply **Plane Consistency Constraint**: Since mirrors are typically planar, enforce that the learned geometry adheres to a plane equation.
    *   Apply **Forward-Facing Normal Constraint**: Ensure normals face the camera to reduce ambiguity.
    *   Use a **Progressive Training Strategy** to stabilize the optimization of mirror geometry [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mirror_nerf_forward(ray_o, ray_d, scene_params, num_samples=192):
    """
    Renders a single ray using Mirror-NeRF.
    
    Args:
        ray_o: Camera origin (3,)
        ray_d: Camera direction (3,)
        scene_params: Dictionary containing MLPs F_geo, F_sigma, F_c, F_n, F_m
        num_samples: Number of points sampled along the ray
        
    Returns:
        color: Rendered color (3,)
        depth: Rendered depth (1,)
    """
    # 1. Sample points along the ray
    # Assuming a fixed range [t_min, t_max] for the scene bounding box
    t_vals = torch.linspace(t_min, t_max, num_samples)
    pts = ray_o.unsqueeze(0) + t_vals.unsqueeze(1) * ray_d.unsqueeze(0) # (num_samples, 3)
    
    # 2. Evaluate Neural Fields
    # Positional Encoding
    pts_enc = positional_encoding(pts)
    
    # Geometry Feature
    f_geo = scene_params['F_geo'](pts_enc)
    
    # Density
    sigma = scene_params['F_sigma'](f_geo)
    
    # Radiance
    # View direction is constant for the ray
    d_enc = positional_encoding(ray_d)
    radiance = scene_params['F_c'](f_geo, d_enc)
    
    # Reflection Probability
    m = scene_params['F_m'](f_geo)
    
    # Smoothed Normal
    normals = scene_params['F_n'](f_geo)
    
    # 3. Volume Rendering Parameters
    # Compute distances between samples
    dists = t_vals[1:] - t_vals[:-1]
    dists = torch.cat([dists, torch.tensor([1e10])]) # Last point contributes negligible density
    
    # Alpha and Transmittance
    alpha = 1.0 - torch.exp(-sigma * dists)
    transmittance = torch.cumprod(torch.exp(-sigma * dists), dim=0)
    transmittance = torch.cat([torch.ones_like(transmittance[:1]), transmittance[:-1]])
    
    weights = alpha * transmittance
    
    # 4. Accumulate Direct Color
    direct_color = torch.sum(weights.unsqueeze(-1) * radiance, dim=0)
    
    # 5. Handle Reflections (Whitted-Style)
    # Identify points with high reflection probability and surface intersection
    # Note: Exact thresholding or stochastic sampling for reflection is not specified in sources.
    # We assume a simplified approach where reflection is blended based on m.
    
    reflected_color = torch.zeros_like(radiance)
    
    # For points identified as mirror surfaces (e.g., high m and specific geometric criteria)
    # Compute reflected direction
    # d_reflected = reflect(ray_d, normals)
    # Trace reflected ray to get color from the rest of the scene
    # This requires recursive ray tracing or a separate pass.
    # For a single forward pass approximation in standard NeRF pipelines:
    # The source states: "blending the final image color from the camera ray color and the reflected ray color weighted by the reflection probability"
    
    # Let's assume we have a function to trace the reflected ray
    # reflected_ray_o = intersection_point
    # reflected_ray_d = reflect(ray_d, normals)
    # reflected_color = mirror_nerf_forward(reflected_ray_o, reflected_ray_d, ...)
    
    # Since recursive tracing is complex for a single forward pass description,
    # and sources emphasize "unified radiance field", the reflection probability m
    # likely acts as a weight in the final composition.
    
    # Simplified blending based on source description:
    # C_final = (1 - m) * C_direct + m * C_reflected
    # However, m varies per sample. The standard approach in such unified fields
    # is often to treat the mirror as a semi-transparent surface or use the probability
    # to switch the ray direction entirely at the first high-probability hit.
    
    # Given the "Whitted-style" mention, the ray likely reflects at the first surface hit.
    # Let's implement the first-hit reflection logic.
    
    # Find the first point with significant density and high reflection probability
    # This is a simplification; exact implementation details for the "trigger" are [G]
    hit_idx = torch.argmax((sigma > threshold) & (m > threshold), dim=0)
    
    if hit_idx is not None:
        hit_pt = pts[hit_idx]
        hit_normal = normals[hit_idx]
        hit_m = m[hit_idx]
        
        # Reflect direction
        hit_d_reflected = reflect(ray_d, hit_normal)
        
        # Trace reflected ray (recursive call or pre-computed)
        # In a full training loop, this would be a separate ray trace
        reflected_color_val = mirror_nerf_forward(hit_pt, hit_d_reflected, scene_params, num_samples)
        
        # Blend based on reflection probability at the hit point
        final_color = (1.0 - hit_m) * direct_color + hit_m * reflected_color_val
    else:
        final_color = direct_color
        
    return final_color

def reflect(v, n):
    """Reflect vector v around normal n."""
    return v - 2.0 * torch.dot(v, n) * n
```

**Parameters:**
*   **Number of samples ($N$):** Not given in the sources.
*   **Positional Encoding:** Standard NeRF positional encoding is used for $\mathbf{x}$ and $\mathbf{d}$ [S1].
*   **MLP Architecture:** Not given in the sources.
*   **Progressive Training Strategy:** Used to stabilize mirror geometry optimization [S1].

**Edge Cases and Pitfalls:**
*   **Mirror Ambiguity:** The "appearance" of a mirror comes from other objects and is inconsistent from different viewpoints, making reconstruction ambiguous [S1].
*   **Noise in Normals:** Analytical gradients of volume density can be noisy; using a separate MLP for normals avoids this [S1].
*   **Unobserved Reflections:** Without accurate mirror geometry, reflections of unobserved parts of the scene (e.g., missing ceiling) cannot be inferred [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Supports photo-realistic novel view synthesis of scenes with mirrors; reconstructs accurate geometry and reflection; enables scene manipulation (adding objects/mirrors, controlling roughness) [S1].
*   **Limitations:** Learning both geometry- and reflection-accurate mirrors is non-trivial; requires regularization constraints (plane consistency, forward-facing normals) [S1].
*   **When to use:** When the scene contains mirrors and physically consistent reflections are required for novel viewpoints.
*   **When not to use:** When the scene does not contain mirrors, or when computational resources are too limited for the additional MLPs and ray tracing overhead.

## Background
Mirror-NeRF was introduced by Junyi Zeng, Chong Bao, et al. from Zhejiang University and the State Key Lab of CAD&CG [S1]. It was presented at MM '23, October 29-November 3, 2023, Ottawa, ON, Canada [S1]. The work addresses the limitations of NeRF and NeRFReN in handling mirrors by introducing a unified radiance field with explicit reflection probability and Whitted-style ray tracing [S1].

## Sources
[S1] Mirror-NeRF: Learning Neural Radiance Fields for Mirrors with Whitted-Style Ray Tracing, https://arxiv.org/abs/2308.03280v1, fetched 2026-10-26
