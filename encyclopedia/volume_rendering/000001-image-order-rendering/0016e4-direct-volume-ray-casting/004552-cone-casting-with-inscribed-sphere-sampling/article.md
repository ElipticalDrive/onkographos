# Cone Casting with Inscribed Sphere Sampling

Also called: MipNeRF Cone Sampling, Tri-MipRF Cone Casting

## Overview
This method renders a pixel as a disc on the image plane and emits a cone from the camera's projection center to sample the volume, avoiding the aliasing artifacts inherent in distance-agnostic ray casting [S1]. It sets itself apart from standard ray casting by treating the pixel area explicitly and from MipNeRF by using a set of inscribed spheres rather than anisotropic multivariate Gaussians for sampling, which aligns with isotropic pre-filtering requirements [S1].

## How it works
**Inputs:** Camera projection center $\mathbf{o}$, pixel center $\mathbf{p}_o$, pixel width $\Delta x$ and height $\Delta y$ in world coordinates, focal length $f$, base radius parameter $\beta$, near plane $t_n$, far plane $t_f$, and the 3D space's Axis-Aligned Bounding Box (AABB).
**Outputs:** Final pixel color $C_{out}$.

**Data Structures:**
*   **Binary Occupancy Grid:** A coarse grid marking empty vs. non-empty space to skip samples [S1].
*   **Distance Vector $t_i$:** A sorted vector of distances along the cone's central axis where sampling spheres are placed [S1].

**Steps:**

1.  **Pixel Disc Formulation:** Calculate the radius $\tilde{r}$ of the pixel disc on the image plane:
    $$ \tilde{r} = \sqrt{\frac{\Delta x \cdot \Delta y}{\pi}} $$
    where $\Delta x$ and $\Delta y$ are the pixel dimensions in world coordinates [S1].

2.  **Cone Emission:** Emit a cone $C$ from the camera projection center $\mathbf{o}$ along the direction $\mathbf{d} = \mathbf{p}_o - \mathbf{o}$. The apex is at $\mathbf{o}$ and the intersection with the image plane is the disc [S1].

3.  **Central Axis Definition:** Define the central axis of the cone as a ray:
    $$ \mathbf{a}(t) = \mathbf{o} + t \mathbf{d} $$
    where $t$ is the distance from the origin [S1].

4.  **Sampling Distance Determination:** Determine sampling distances $t_i$ by uniformly sampling between the camera's near $t_n$ and far $t_f$ planes, or the intersections of the cone's central axis with the AABB of the 3D space [S1].

5.  **Sphere Sampling:** Sample the cone using a set of spheres $S(\mathbf{x}, r)$ parameterized by centers $\mathbf{x}$ and radii $r$. The centers $\mathbf{x}$ lie on the central axis, and the radii $r$ are set such that the spheres are inscribed with the cone. For a distance $t_i$, the center $\mathbf{x}_i$ and radius $r_i$ are calculated as:
    $$ \mathbf{x}_i = \mathbf{o} + t_i \mathbf{d} $$
    $$ r_i = \frac{\|\mathbf{x}_i - \mathbf{o}\|_2 \cdot f \cdot \beta}{\|\mathbf{d}\|_2 \cdot \sqrt{\left(\sqrt{\|\mathbf{d}\|_2^2 - f^2} - \beta\right)^2 + f^2}} $$
    where $\|\cdot\|_2$ is the Euclidean norm, $f$ is the focal length, and $\beta$ is the base radius parameter [S1].

6.  **Volume Query and Compositing:** For each sphere $S(\mathbf{x}_i, r_i)$, query the volume density $\rho_i$ and color $\mathbf{c}_i$. Approximate the volume rendering integral using numerical quadrature. The transmittance $T_i$ and accumulated color are computed as:
    $$ T_i = \exp\left(-\sum_{k < i} \rho_k (t_{k+1} - t_k)\right) $$
    $$ C_{out} = \sum_i T_i \left(1 - \exp(-\rho_i (t_{i+1} - t_i))\right) \mathbf{c}_i $$
    where $t_{k+1} - t_k$ represents the distance between sampled spheres [S1].

7.  **Optimization:** Use the binary occupancy grid to skip samples in empty areas and concentrate samples near surfaces [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cone_casting(o, p_o, dx, dy, f, beta, t_n, t_f, aabb, occupancy_grid, volume_query):
    """
    Render a single pixel using cone casting with inscribed sphere sampling.
    
    Args:
        o: Camera projection center (3D vector)
        p_o: Pixel center in world coordinates (3D vector)
        dx, dy: Pixel width and height in world coordinates (floats)
        f: Focal length (float)
        beta: Base radius parameter (float)
        t_n, t_f: Near and far plane distances (floats)
        aabb: Axis-Aligned Bounding Box of the 3D space
        occupancy_grid: Binary grid for empty space skipping
        volume_query: Function returning (density, color) for a sphere center and radius
        
    Returns:
        C_out: Final pixel color (3D vector)
    """
    # Step 1: Calculate pixel disc radius
    # Note: The source formula uses sqrt(area/pi). Area = dx * dy.
    r_tilde = math.sqrt((dx * dy) / math.pi)
    
    # Step 2 & 3: Define cone direction and central axis
    d = p_o - o
    # Normalize direction for consistent t interpretation if needed, 
    # but source uses d directly in norm calculations.
    
    # Step 4: Determine sampling distances
    # Intersect central axis with AABB to get valid t range
    t_min, t_max = intersect_ray_aabb(o, d, aabb)
    # Clamp to near/far planes
    t_start = max(t_min, t_n)
    t_end = min(t_max, t_f)
    
    # Uniformly sample distances between t_start and t_end
    # Number of samples N is implementation detail, assumed fixed or adaptive
    N = 32 # Example value
    t_values = linspace(t_start, t_end, N)
    
    C_out = [0.0, 0.0, 0.0]
    accumulated_transmittance = 1.0
    
    for i in range(N):
        t_i = t_values[i]
        t_next = t_values[i+1] if i < N-1 else t_end
        
        # Step 5: Calculate sphere center and radius
        x_i = o + t_i * d
        norm_x_o = norm(x_i - o)
        norm_d = norm(d)
        
        # Inscribed sphere radius formula
        inner_sqrt = math.sqrt(norm_d**2 - f**2)
        denominator_term = math.sqrt((inner_sqrt - beta)**2 + f**2)
        r_i = (norm_x_o * f * beta) / (norm_d * denominator_term)
        
        # Optimization: Check occupancy grid
        if not is_non_empty(occupancy_grid, x_i, r_i):
            continue
            
        # Query volume for density and color
        density_i, color_i = volume_query(x_i, r_i)
        
        # Step 6: Compositing
        # Distance between samples
        dt = t_next - t_i
        
        # Opacity for this segment
        alpha_i = 1.0 - math.exp(-density_i * dt)
        
        # Contribution
        contribution = accumulated_transmittance * alpha_i * color_i
        C_out += contribution
        
        # Update transmittance
        accumulated_transmittance *= (1.0 - alpha_i)
        
    return C_out
```

**Edge Cases and Pitfalls:**
*   **Focal Length Constraint:** The formula for $r_i$ involves $\sqrt{\|\mathbf{d}\|_2^2 - f^2}$. If $\|\mathbf{d}\|_2 < f$, the term becomes imaginary. This implies the direction vector magnitude must be greater than the focal length, or $f$ must be scaled appropriately relative to the coordinate system [S1].
*   **Empty Space:** Without the binary occupancy grid, sampling uniformly along the axis wastes computation in empty regions [S1].
*   **Isotropy:** Using multivariate Gaussians (as in MipNeRF) is incompatible with isotropic pre-filtering encodings like Tri-Mip, necessitating the sphere approximation [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples $N$ per pixel, plus the cost of occupancy grid lookups [S1].
*   **Memory Complexity:** Depends on the size of the occupancy grid and the volume data structure queried [S1].
*   **Strengths:** Avoids aliasing artifacts by accounting for pixel area; efficient skipping of empty space via occupancy grid [S1].
*   **Limitations:** Approximates the cone with spheres, which may introduce minor geometric inaccuracies compared to exact integration [S1]. Requires careful tuning of the base radius parameter $\beta$ [S1].

## Background
The method was introduced in the context of Tri-MipRF (Tri-Mip Representation for Efficient Anti-Aliasing Neural Radiance Fields) [S1]. It addresses the limitations of standard NeRF (ray casting) and MipNeRF (anisotropic Gaussian cone sampling) by providing an isotropic area-sampling approach suitable for hash-based encodings [S1].

## Sources
[S1] Tri-MipRF: Tri-Mip Representation for Efficient Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01811, fetched 2026-10-26
