# Cone Casting with Inscribed Sphere Sampling

Also called: Cone casting, MipNeRF Cone Sampling, Tri-MipRF Cone Casting

## Overview
Cone casting with inscribed sphere sampling is a rendering technique that formulates a pixel as a disc on the image plane and emits a cone from the camera's projection center to sample the volume, avoiding the aliasing artifacts of distance-agnostic ray casting [S1]. Unlike sibling methods that treat pixels as points or use anisotropic Gaussian distributions for area sampling, this method uses isotropic spheres inscribed within the cone to match pre-filtered feature encodings [S1].

## How it works
**Inputs:**
*   Camera parameters: optical center $\mathbf{o}$, focal length $f$, pixel dimensions in world coordinates $\Delta x, \Delta y$.
*   Ray direction $\mathbf{d}$ (normalized or unnormalized, see formula).
*   Sampling distances $t_i$ along the ray.
*   Model parameters: Tri-Mip encoding maps $\mathcal{M}$, tiny MLP weights $\Theta$.
*   Scene bounds: near plane $t_n$, far plane $t_f$ (or AABB intersections).

**Outputs:**
*   Final pixel color $C$.

**Data Structures:**
*   **Sphere $S_i$**: Defined by center $\mathbf{x}_i$ and radius $r_i$.
*   **Tri-Mip Encoding**: Three 2D mipmap arrays representing orthogonal planes (XY, YZ, XZ).
*   **Tiny MLP**: A shallow neural network mapping feature vectors to density and color.

**Steps:**

1.  **Pixel Disc and Cone Definition**:
    Calculate the pixel disc radius $\tilde{r}$ in world coordinates:
    $$
    \tilde{r} = \frac{\Delta x \cdot \Delta y}{\pi}
    $$
    Emit a cone $C$ from the camera projection center $\mathbf{o}$ along the direction $\mathbf{d} = \mathbf{p}_o - \mathbf{o}$, where $\mathbf{p}_o$ is the pixel center. The apex is at $\mathbf{o}$ and the intersection with the image plane is the disc. The central axis of the cone is defined as:
    $$
    \mathbf{a}(t) = \mathbf{o} + t\mathbf{d}
    $$

2.  **Sampling Distance Selection**:
    Determine a set of sampling distances $t_i$. These are typically uniformly sampled between the camera's near $t_n$ and far $t_f$ planes, or the intersections of the cone's central axis with the Axis-Aligned Bounding Box (AABB) of the 3D space [S1]. A binary occupancy grid may be used to skip samples in empty areas and concentrate samples near surfaces [S1].

3.  **Sphere Parameterization**:
    For each distance $t_i$, calculate the sphere center $\mathbf{x}_i$ and inscribed radius $r_i$. The center lies on the central axis:
    $$
    \mathbf{x}_i = \mathbf{o} + t_i \mathbf{d}
    $$
    The radius $r_i$ is calculated to ensure the sphere is inscribed within the cone. The formula provided in the source is:
    $$
    r_i = \frac{\|\mathbf{x}_i - \mathbf{o}\|_2 \cdot f \cdot \beta}{\|\mathbf{d}\|_2 \cdot \sqrt{\left(\sqrt{\|\mathbf{d}\|_2^2 - f^2} - \beta\right)^2 + f^2}}
    $$
    where $f$ is the focal length and $\beta$ is a base radius parameter [S1]. Note: The source notation $\baseradius$ is mapped to $\beta$.

4.  **Featurization (Tri-Mip Encoding)**:
    Orthogonally project each sphere $S_i(\mathbf{x}_i, r_i)$ onto the three Tri-Mip planes (XY, YZ, XZ). Sample the corresponding mipmaps based on the sphere's occupied area to generate a feature vector $\mathbf{f}_i$ [S1]. This step leverages the isotropic pre-filtering of the Tri-Mip encoding.

5.  **Density and Color Prediction**:
    Feed the feature vector $\mathbf{f}_i$ and the view direction $\mathbf{d}$ into a tiny MLP to predict density $\tau_i$ and color $\mathbf{c}_i$:
    $$
    [\tau_i, \mathbf{c}_i] = \text{MLP}(\mathbf{f}_i, \mathbf{d}; \Theta)
    $$
    where $\Theta$ are the MLP weights [S1].

6.  **Volume Rendering Integration**:
    Integrate the density and color of the sampled spheres using standard front-to-back volume rendering to produce the final pixel color [S1]:
    $$
    C = \sum_{i} T_i \left(1 - \exp(-\tau_i (t_{i+1} - t_i))\right) \mathbf{c}_i
    $$
    with
    $$
    T_i = \exp\left(-\sum_{k < i} \tau_k (t_{k+1} - t_k)\right)
    $$
    where $t$ represents the distance between sampled spheres [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cone_casting_with_inscribed_spheres(o, d, pixel_width_world, pixel_height_world, 
                                       focal_length, base_radius, t_near, t_far, 
                                       num_samples, tri_mip_maps, mlp_model):
    """
    Renders a single pixel using cone casting with inscribed sphere sampling.
    
    Args:
        o: Camera optical center (3D vector).
        d: Ray direction vector (o + t*d points into scene).
        pixel_width_world: Width of pixel in world units.
        pixel_height_world: Height of pixel in world units.
        focal_length: Camera focal length f.
        base_radius: Base radius parameter beta.
        t_near: Near plane distance.
        t_far: Far plane distance.
        num_samples: Number of spheres to sample along the cone.
        tri_mip_maps: Dictionary of 2D mipmap arrays for XY, YZ, XZ planes.
        mlp_model: Function implementing the tiny MLP.
        
    Returns:
        final_color: Rendered color of the pixel.
    """
    
    # 1. Calculate pixel disc radius
    # r_tilde = (dx * dy) / pi
    r_tilde = (pixel_width_world * pixel_height_world) / math.pi
    
    # 2. Define sampling distances
    # Uniformly sample between near and far
    t_values = linspace(t_near, t_far, num_samples)
    
    final_color = zeros(3)
    transmittance = 1.0
    
    for i in range(num_samples):
        t_i = t_values[i]
        t_next = t_values[i+1] if i < num_samples - 1 else t_far
        
        # 3. Sphere Center and Radius
        # x_i = o + t_i * d
        x_i = o + t_i * d
        
        # Calculate radius r_i
        # Note: Source formula uses norms. Assuming d is not necessarily normalized.
        dist_from_origin = norm(x_i - o)
        norm_d = norm(d)
        
        # Denominator term inside sqrt
        # sqrt(||d||^2 - f^2) - beta
        # Guard against negative value inside sqrt if ||d|| < f
        inner_sqrt_arg = norm_d**2 - focal_length**2
        if inner_sqrt_arg < 0:
            inner_sqrt_arg = 0
            
        term = sqrt(inner_sqrt_arg) - base_radius
        denominator = norm_d * sqrt(term**2 + focal_length**2)
        
        if denominator == 0:
            r_i = 0
        else:
            r_i = (dist_from_origin * focal_length * base_radius) / denominator
            
        # 4. Featurization via Tri-Mip
        # Project sphere onto XY, YZ, XZ planes and sample mipmaps
        # This requires a helper function to sample 2D mipmap at a location with a given radius
        f_xy = sample_mipmap(tri_mip_maps['XY'], x_i.x, x_i.y, r_i)
        f_yz = sample_mipmap(tri_mip_maps['YZ'], x_i.y, x_i.z, r_i)
        f_xz = sample_mipmap(tri_mip_maps['XZ'], x_i.x, x_i.z, r_i)
        
        # Concatenate features
        f_i = concatenate([f_xy, f_yz, f_xz])
        
        # 5. Predict Density and Color
        tau_i, c_i = mlp_model(f_i, d)
        
        # 6. Volume Rendering Step
        # Optical depth for this segment
        delta_t = t_next - t_i
        alpha = 1.0 - exp(-tau_i * delta_t)
        
        # Contribution
        contribution = transmittance * alpha * c_i
        final_color += contribution
        
        # Update transmittance
        transmittance *= (1.0 - alpha)
        
        # Early termination if transmittance is negligible
        if transmittance < 1e-4:
            break
            
    return final_color

def sample_mipmap(mipmap_array, u, v, radius):
    """
    Samples a 2D mipmap array at (u, v) with radius r.
    This is a placeholder for the specific Tri-Mip sampling logic.
    In practice, this involves selecting the appropriate mipmap level 
    based on the projected radius and performing bilinear/trilinear interpolation.
    """
    # Implementation detail: Map world coordinates to texture coordinates
    # Select level based on radius
    # Return feature vector
    pass
```

**Parameters:**
*   `base_radius` ($\beta$): A base radius parameter controlling the scale of inscribed spheres. Not given in the sources.
*   `num_samples`: Number of spheres along the cone. Not given in the sources.
*   `tiny MLP`: A shallow neural network. Specific architecture (layers, activation functions) is not given in the sources.

**Edge Cases and Pitfalls:**
*   **Focal Length vs Ray Direction Norm**: The formula for $r_i$ involves $\sqrt{\|\mathbf{d}\|_2^2 - f^2}$. If $\|\mathbf{d}\| < f$, the argument to the square root is negative. The implementation must handle this, potentially by clamping to 0 or ensuring $\mathbf{d}$ is normalized such that $\|\mathbf{d}\|=1$ and $f$ is scaled appropriately [S1].
*   **Isotropy**: The method relies on isotropic pre-filtering. Using anisotropic kernels (like Gaussians in MipNeRF) is incompatible with the Tri-Mip encoding's design [S1].
*   **Occupancy Grid**: If a binary occupancy grid is used, degenerate cases where the ray grazes the boundary of the AABB or empty regions must be handled to avoid skipping necessary samples [S1].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of sampled spheres per pixel. The use of a binary occupancy grid allows skipping empty space, reducing effective complexity in sparse scenes [S1].
*   **Memory Complexity**: Depends on the size of the Tri-Mip maps (three 2D mipmap arrays) and the tiny MLP. The Tri-Mip encoding is described as compact, requiring only three 2D maps for base levels [S1].
*   **Strengths**: Produces anti-aliased high-fidelity renderings in both close-up and distant views. More efficient than super-sampling techniques [S1].
*   **Limitations**: Requires the specific Tri-Mip encoding structure. Not directly compatible with hash encoding without modification due to hash collisions and anisotropy issues [S1].

## Background
Cone casting with inscribed sphere sampling was introduced in the Tri-MipRF paper (2023) as a rendering strategy for efficient anti-aliased neural radiance fields [S1]. It addresses the flaw in previous methods (like NeRF) that formulate pixels as single points, causing blurring in close-up views and aliasing in distant views. It differs from MipNeRF, which uses multivariate Gaussians, by using inscribed spheres to match the isotropic pre-filtering of the Tri-Mip encoding [S1].

## Sources
[S1] Tri-MipRF: Tri-Mip Representation for Efficient Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01811, fetched 2026-10-26
