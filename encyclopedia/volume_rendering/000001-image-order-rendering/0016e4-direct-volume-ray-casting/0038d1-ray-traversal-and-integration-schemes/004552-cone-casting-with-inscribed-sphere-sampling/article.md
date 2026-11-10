# Cone Casting with Inscribed Sphere Sampling

Also called: Cone casting, MipNeRF Cone Sampling, Tri-MipRF Cone Casting

## Overview
Cone casting with inscribed sphere sampling is a volume rendering technique that formulates a pixel as a disc on the image plane and emits a cone from the camera's projection center to sample the volume, thereby avoiding the aliasing artifacts inherent in distance-agnostic ray casting [S1]. Unlike MipNeRF, which samples the cone with anisotropic multivariate Gaussians, this method uses spheres inscribed within the cone to provide isotropic pre-filtering, making it particularly effective when paired with Tri-Mip encoding [S1].

## How it works
**Inputs:** Camera intrinsic parameters (focal length $f$, pixel dimensions $\Delta x, \Delta y$), camera extrinsic parameters (optical center $\mathbf{o}$), scene bounding box (AABB), and a density/color field (typically parameterized by a Tri-Mip encoding and a tiny MLP).
**Outputs:** Final pixel color $C_{out}$.

**Data Structures:**
*   **Cone $C$:** Defined by apex $\mathbf{o}$ and direction $\mathbf{d}$.
*   **Sampling Spheres $S$:** A set of spheres $\{S_i\}$, each defined by center $\mathbf{x}_i$ and radius $r_i$.
*   **Occupancy Grid:** A binary grid marking empty vs. non-empty space to skip samples [S1].

**Steps:**

1.  **Pixel Disc and Cone Definition:**
    Calculate the radius $\tilde{r}$ of the pixel disc on the image plane using the pixel width $\Delta x$ and height $\Delta y$ in world coordinates:
    $$ \tilde{r} = \sqrt{\frac{\Delta x \cdot \Delta y}{\pi}} $$
    [S1]
    Emit a cone $C$ from the camera projection center $\mathbf{o}$ along the direction $\mathbf{d} = \mathbf{p}_o - \mathbf{o}$, where $\mathbf{p}_o$ is the pixel center. The central axis of the cone is defined as:
    $$ \mathbf{a}(t) = \mathbf{o} + t\mathbf{d} $$
    [S1]

2.  **Sampling Distance Determination:**
    Determine sampling distances $t_i$ by uniformly sampling between the camera's near plane $t_n$ and far plane $t_f$, or the intersections of the cone's central axis with the AABB of the 3D space. An occupancy grid is used to skip samples in empty areas and concentrate samples near surfaces [S1].

3.  **Sphere Parameterization:**
    For each distance $t_i$, calculate the sphere center $\mathbf{x}_i$ and inscribed radius $r_i$. The center lies on the central axis:
    $$ \mathbf{x}_i = \mathbf{o} + t_i \mathbf{d} $$
    [S1]
    The radius $r_i$ is calculated to ensure the sphere is inscribed within the cone:
    $$ r_i = \frac{\|\mathbf{x}_i - \mathbf{o}\|_2 \cdot f \cdot \beta}{\|\mathbf{d}\|_2 \cdot \sqrt{\left(\sqrt{\|\mathbf{d}\|_2^2 - f^2} - \beta\right)^2 + f^2}} $$
    where $f$ is the focal length and $\beta$ is a base radius parameter [S1].

4.  **Featurization:**
    Featurize each sphere by orthogonally projecting it onto the three Tri-Mip planes (XY, YZ, XZ) and sampling the corresponding mipmaps based on the sphere's occupied area. This yields a feature vector $\mathbf{f}_i$ [S1].

5.  **Density and Color Prediction:**
    Feed the feature vector $\mathbf{f}_i$ and view direction $\mathbf{d}$ into a tiny MLP to predict density $\tau_i$ and color $\mathbf{c}_i$:
    $$ [\tau_i, \mathbf{c}_i] = \text{MLP}(\mathbf{f}_i, \mathbf{d}; \Theta) $$
    where $\Theta$ are the MLP weights [S1].

6.  **Volume Rendering Integration:**
    Integrate the density and color of the sampled spheres using front-to-back volume rendering to produce the final pixel color. The contribution of each sphere is approximated by numerical quadrature:
    $$ C_{out} = \sum_{i} T_i \left(1 - \exp(-\tau_i (t_{i+1} - t_i))\right) \mathbf{c}_i $$
    with transmittance:
    $$ T_i = \exp\left(-\sum_{k < i} \tau_k (t_{k+1} - t_k)\right) $$
    [S1]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cone_casting_rendering(camera, scene, pixel_center):
    # 1. Cone Definition
    o = camera.optical_center
    d = normalize(pixel_center - o)
    # Calculate pixel disc radius in world coordinates
    # Note: delta_x, delta_y must be derived from camera intrinsics/extrinsics
    delta_x, delta_y = get_pixel_world_dimensions(camera, pixel_center)
    r_tilde = sqrt((delta_x * delta_y) / pi)
    
    # 2. Sampling Distances
    # Intersect central axis with AABB or use near/far planes
    t_start, t_end = intersect_axis_with_aabb(o, d, scene.aabb)
    if t_start is None:
        return black_color
        
    # Uniform sampling or occupancy-grid guided sampling
    t_values = uniform_sample(t_start, t_end, num_samples)
    
    # 3. Sphere Parameterization and Rendering Loop
    accumulated_color = 0.0
    accumulated_transmittance = 1.0
    
    for i in range(len(t_values) - 1):
        t_curr = t_values[i]
        t_next = t_values[i + 1]
        
        # Sphere center on axis
        x_curr = o + t_curr * d
        
        # Sphere radius (inscribed)
        # beta is a base radius parameter, f is focal length
        norm_d = norm(d)
        norm_x_o = norm(x_curr - o)
        numerator = norm_x_o * camera.focal_length * beta
        denominator_term1 = sqrt(norm_d**2 - camera.focal_length**2) - beta
        denominator = norm_d * sqrt(denominator_term1**2 + camera.focal_length**2)
        r_curr = numerator / denominator
        
        # Check occupancy grid to skip empty space
        if not scene.occupancy_grid.contains(x_curr, r_curr):
            continue
            
        # 4. Featurization (Tri-Mip)
        # Project sphere onto XY, YZ, XZ planes and sample mipmaps
        f_curr = tri_mip_encode(x_curr, r_curr, scene.tri_mip_maps)
        
        # 5. MLP Prediction
        tau_curr, c_curr = tiny_mlp(f_curr, d, scene.mlp_weights)
        
        # 6. Integration
        # Step size for this segment
        dt = t_next - t_curr
        
        # Opacity for this segment
        alpha_curr = 1.0 - exp(-tau_curr * dt)
        
        # Front-to-back compositing
        weight = accumulated_transmittance * alpha_curr
        accumulated_color += weight * c_curr
        accumulated_transmittance *= (1.0 - alpha_curr)
        
        # Early termination if fully opaque
        if accumulated_transmittance < epsilon:
            break
            
    return accumulated_color
```

**Parameters:**
*   $\beta$ (base radius): Not given in the sources.
*   $f$ (focal length): Provided by camera intrinsics [S1].
*   $\Delta x, \Delta y$ (pixel dimensions): Derived from camera parameters [S1].
*   $t_n, t_f$ (near/far planes): Provided by camera settings or AABB intersections [S1].

**Edge Cases and Pitfalls:**
*   **Anisotropy:** MipNeRF uses multivariate Gaussians which are anisotropic; this method uses spheres to ensure isotropic pre-filtering, which is required for Tri-Mip encoding [S1].
*   **Empty Space:** Without an occupancy grid, computation is wasted in empty areas [S1].
*   **Precision:** The radius formula involves square roots and differences; care must be taken with numerical stability when the camera is very close to the scene or when $f$ is large relative to $\|\mathbf{d}\|$ [S1].

## Subtopics

### Tri-Mip Encoding
*   **What it is:** A compact 2D representation for 3D content using three mipmaps (XY, YZ, XZ planes) to model the pre-filtered 3D feature space.
*   **What sets it apart:** It allows efficient area sampling by taking advantage of different mipmap levels, avoiding hash collisions associated with hash encoding.
*   **When to choose it:** When rendering with cone casting to achieve anti-aliased high-fidelity images efficiently.
*   **Link:** [Tri-Mip Encoding](8342-tri-mip-encoding.md)

## Operating characteristics
*   **Time Complexity:** Depends on the number of samples $N$ and the cost of Tri-Mip sampling and MLP inference. The occupancy grid helps skip empty space, reducing effective $N$ [S1].
*   **Memory Complexity:** Compact, as the Tri-Mip encoding only requires three 2D maps for base levels and a shallow MLP [S1].
*   **Strengths:** Produces anti-aliased high-fidelity renderings, efficient convergence, and compact model size [S1].
*   **Limitations:** Requires calibrated multi-view images for reconstruction; relies on the specific Tri-Mip encoding structure [S1].
*   **When to use it:** For static scene reconstruction and rendering where anti-aliasing is critical and computational efficiency is desired [S1].
*   **When not to use it:** For dynamic scenes without extension, or when isotropic pre-filtering is not required [S1].

## Background
*   **Origin:** Proposed in "Tri-MipRF: Tri-Mip Representation for Efficient Anti-Aliasing Neural Radiance Fields" (2023) [S1].
*   **History:** It addresses the flaw in previous methods (like NeRF and MipNeRF) of formulating pixels as points or using anisotropic sampling, which causes blurring in close-up views and aliasing in distant views [S1].
*   **Usage:** Used in neural radiance field reconstruction and rendering [S1].

## Sources
[S1] Tri-MipRF: Tri-Mip Representation for Efficient Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01811, fetched 2026-10-26
