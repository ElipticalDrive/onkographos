# Anisotropic Screen-Space Fluid Rendering

Also called: Anisotropic Point Sprite Fluid Rendering, Anisotropic Screen Space Transformation, Anisotropic screen space rendering for particle-based fluid simulation, Elliptical Point Sprite Rendering, Homogeneous Anisotropic Transformation, WPCA-based Anisotropic Fluid Rendering

## Overview
Anisotropic Screen-Space Fluid Rendering is a real-time technique that converts discrete particle data (e.g., from SPH simulations) into a continuous fluid volume by applying anisotropic transformations to point sprites before screen-space filtering [S1]. It solves the problem of jagged, uneven surfaces and over-smoothing inherent in standard isotropic point sprite rendering by stretching particles along the principal axes of local particle distribution [S1]. Unlike standard methods that use spherical sprites and simple Gaussian filters, this method uses elliptical sprites derived from Weighted Principal Components Analysis (WPCA) to preserve sharp high-frequency details while smoothing discontinuities [S1].

## How it works
The method extends the Screen-Space Fluid Rendering Pipeline by modifying the depth buffer generation step. Instead of rendering isotropic spheres, it renders anisotropic ellipsoids in parameter space, which are then projected to screen space.

### Inputs and Outputs
*   **Inputs:** Particle positions $\mathbf{x}_j$, particle densities $\rho_j$, particle volumes $m_j$, camera projection matrix $P$, model matrix $M$, camera position, and viewport dimensions.
*   **Outputs:** Depth buffer, thickness buffer, reconstructed normal buffer, and final shaded fluid image.

### Data Structures
1.  **Particle Data:** Array of particles, each containing position $\mathbf{x}_j \in \mathbb{R}^3$, density $\rho_j$, and volume $m_j$.
2.  **Anisotropy Matrix $B$:** A $3 \times 3$ matrix derived from WPCA for each particle or local neighborhood.
3.  **Homogeneous Anisotropic Transformation Matrix $C_{r,h}$:** A $4 \times 4$ matrix used to transform the sphere to an ellipsoid in parameter space.
    *   Layout:
        $$
        C_{r,h} = \begin{bmatrix} C_r & \mathbf{0} \\ \mathbf{0}^T & 1 \end{bmatrix}
        $$
    *   Where $C_r$ is the $3 \times 3$ regularized covariance matrix from WPCA.
4.  **Scaling Matrix $T$:** A $4 \times 4$ diagonal scaling matrix.
    *   Layout:
        $$
        T = \begin{bmatrix} r_c & 0 & 0 & x_o \\ 0 & r_c & 0 & y_o \\ 0 & 0 & r_c & z_o \\ 0 & 0 & 0 & 1 \end{bmatrix}
        $$
    *   $r_c$: Sphere radius in clip space.
    *   $(x_o, y_o, z_o)$: Particle position in object space.

### Parameters
*   $\sigma_i$: Standard deviation for Gaussian filter, determined by depth $z_i$ [S1].
*   $h$: Support radius of the kernel function. Beyond $h$, the kernel value is 0 [S1].
*   $\beta$: Scaling factor in the anisotropic kernel function [S1].
*   $r_c$: Radius of the sphere in clip space, measured from the bias between the sphere center and a point on the surface [S1].
*   $k$: Attenuation coefficient vector of the fluid [S1].
*   $I_0$: Light intensity value [S1].

### Steps

1.  **Calculate Particle Anisotropy (WPCA):**
    For each particle $i$, determine the local anisotropy matrix $B$ using Weighted Principal Components Analysis on neighboring particles $j$.
    *   Compute weights $\omega_{ij}$ based on particle proximity (standard SPH weighting).
    *   Construct the weighted covariance matrix from the relative positions of neighbors.
    *   Perform Singular Value Decomposition (SVD) on the covariance matrix to obtain eigenvalues and eigenvectors.
    *   Construct $B$ such that it stretches the kernel along the tangential directions of the fluid surface [S1].

2.  **Construct Transformation Matrices:**
    For each particle $j$ at object position $\mathbf{x}_j = (x_o, y_o, z_o)$:
    *   Define the scaling matrix $T$ using the clip-space radius $r_c$ and object position [S1]:
        $$
        T = \begin{bmatrix} r_c & 0 & 0 & x_o \\ 0 & r_c & 0 & y_o \\ 0 & 0 & r_c & z_o \\ 0 & 0 & 0 & 1 \end{bmatrix}
        $$
    *   Define the homogeneous anisotropic transformation matrix $C_{r,h}$ using the regularized covariance matrix $C_r$ derived from WPCA [S1]:
        $$
        C_{r,h} = \begin{bmatrix} C_r & 0 \\ 0 & 1 \end{bmatrix}
        $$

3.  **Render Anisotropic Point Sprites (Depth & Thickness):**
    *   Transform the particle to screen space.
    *   Instead of rendering a sphere, render an elliptical point sprite. The shape is defined by applying $C_{r,h}$ in parameter space before projection.
    *   To determine the projected ellipse size in clip space, trace the boundary back to parameter space. For a point $\mathbf{x}_c$ inside the sprite in clip space, the condition $\mathbf{n}_c^T \mathbf{x}_c \le 0$ must hold, where $\mathbf{n}_c = (1, 0, 0, b_x)^T$ [S1].
    *   Trace $b_x$ back to parameter space using the inverse transformation:
        $$
        \mathbf{n}_p = (P \cdot M \cdot T \cdot C_{r,h})^T \mathbf{n}_c
        $$
        where $\mathbf{n}_p$ is the normal vector in parameter space [S1].
    *   Render these ellipses to extract depth values $z_i$ and thickness information into the depth buffer. Note that points on the resulting surface may not lie on the projected plane ($x_{c,z} \neq x_{ani,c,z}$) [S1].

4.  **Calculate Thickness Buffer:**
    Compute the thickness $T(\mathbf{p}_i)$ at each screen pixel $\mathbf{p}_i$ by summing the contributions of all fluid particles $j$ using a Gaussian kernel:
    $$
    T(\mathbf{p}_i) = \sum_{j=0}^{n} G(\mathbf{p}_i - \hat{\mathbf{x}}_j)
    $$
    where $\hat{\mathbf{x}}_j$ is the projection position of particle $j$ in screen space, and $G$ is the Gaussian function [S1].

5.  **Smooth Depth Information:**
    Apply screen-space filters to the depth buffer to resolve jagged edges.
    *   **Gaussian Filter:**
        $$
        G(\mathbf{p}_i, \mathbf{p}_j, \sigma_i) = \frac{1}{2\pi\sigma_i^2} e^{-\frac{\|\mathbf{p}_i - \mathbf{p}_j\|^2}{\sigma_i}}
        $$
        where $\mathbf{p}_j$ is a neighbor pixel, and $\sigma_i$ is determined by the depth $z_i$ [S1].
    *   **Anisotropic Smoothing Kernel (Optional/Alternative):**
        Use the anisotropic kernel $W(\mathbf{r}, B)$ for smoothing:
        $$
        W(\mathbf{r}, B) = \beta \det(B) P(|\mathbf{r}B|)
        $$
        where $\mathbf{r}$ is the relative position, $B$ is the anisotropy matrix, $\beta$ is a scaling factor, and $P$ is a symmetric decaying spline with finite support [S1].
    *   Combine with curvature flow or narrow-range filters to preserve sharp details [S1].

6.  **Reconstruct Normals:**
    Calculate partial derivatives of the smoothed depth image using finite differences to reconstruct the surface normal at each pixel [S1].

7.  **Surface Shading:**
    Render the fluid surface using the reconstructed normals and thickness information.
    *   Calculate transmittance using the Beer-Lambert law:
        $$
        I(d) = I_0 d^{-kd}
        $$
        where $d$ is the thickness, $I_0$ is light intensity, and $k$ is the attenuation coefficient [S1].
    *   Apply Blinn-Phong illumination and refraction effects [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def anisotropic_screen_space_fluid_rendering(particles, camera, viewport):
    """
    particles: List of Particle objects with pos (x,y,z), density, volume
    camera: Camera object with projection matrix P, model matrix M, position
    viewport: Viewport dimensions
    """
    
    # 1. Pre-compute Anisotropy for each particle
    for particle in particles:
        # Get neighbors within support radius h
        neighbors = get_neighbors(particle, particles, h=particle.support_radius)
        
        # Compute Weighted Covariance Matrix
        cov_matrix = zeros((3, 3))
        total_weight = 0
        for neighbor in neighbors:
            r = neighbor.pos - particle.pos
            w = compute_weight(particle, neighbor) # e.g., Gaussian weight based on distance
            cov_matrix += w * outer_product(r, r)
            total_weight += w
            
        # Regularize and Decompose
        cov_matrix += epsilon * identity(3) # Regularization [G]
        eigenvalues, eigenvectors = svd(cov_matrix)
        
        # Construct C_r (Regularized Covariance Matrix for transformation)
        # Note: Source defines C_r as the regularized covariance matrix from WPCA
        C_r = cov_matrix 
        
        # Construct Homogeneous Anisotropic Transformation Matrix C_r,h
        particle.C_rh = matrix_4x4([
            [C_r[0,0], C_r[0,1], C_r[0,2], 0],
            [C_r[1,0], C_r[1,1], C_r[1,2], 0],
            [C_r[2,0], C_r[2,1], C_r[2,2], 0],
            [0, 0, 0, 1]
        ])
        
        # Construct Scaling Matrix T
        # rc is clip-space radius, determined by particle size and camera distance [G]
        rc = compute_clip_radius(particle.pos, camera)
        particle.T = matrix_4x4([
            [rc, 0, 0, particle.pos.x],
            [0, rc, 0, particle.pos.y],
            [0, 0, rc, particle.pos.z],
            [0, 0, 0, 1]
        ])

    # 2. Render Depth and Thickness Buffers
    depth_buffer = initialize_buffer(viewport)
    thickness_buffer = initialize_buffer(viewport)
    
    for particle in particles:
        # Transform particle to clip space using anisotropic matrix
        # The sprite is an ellipsoid defined by C_r,h in parameter space
        clip_transform = camera.P * camera.M * particle.T * particle.C_rh
        
        # Determine ellipse boundary in clip space
        # For a point x_c inside sprite, n_c^T x_c <= 0, n_c = (1, 0, 0, bx)^T
        # Trace bx back to parameter space: n_p = (P * M * T * C_r,h)^T * n_c
        # This defines the extent of the ellipse to rasterize [S1]
        
        rasterize_ellipse(particle, clip_transform, depth_buffer, thickness_buffer)

    # 3. Smooth Depth Buffer
    smoothed_depth = apply_gaussian_filter(depth_buffer)
    # Alternatively, apply anisotropic kernel W(r, B) if B is available per pixel [S1]
    
    # 4. Reconstruct Normals
    normals = reconstruct_normals(smoothed_depth)
    
    # 5. Shade
    final_image = zeros(viewport)
    for y in range(viewport.height):
        for x in range(viewport.width):
            d = thickness_buffer[y, x]
            if d > 0:
                # Beer-Lambert Law
                # I(d) = I_0 * d^(-k*d)
                # Note: Source formula I(d) = I_0 * d^(-kd) implies exponent is -k*d
                intensity = camera.light_intensity * (d ** (-camera.attenuation_coeff * d))
                normal = normals[y, x]
                final_image[y, x] = apply_blinn_phong(intensity, normal, camera)
                
    return final_image

def compute_clip_radius(pos, camera):
    """
    [G] Helper to estimate rc. 
    Source states rc is measured from bias between sphere center and surface point in clip coords.
    Typically rc = particle_radius * projection_scale.
    """
    # Simplified estimation
    dist = length(camera.pos - pos)
    return 1.0 / dist # Placeholder for actual projection math
```

### Edge Cases and Pitfalls
*   **Uneven Surfaces:** Directly using spherical point sprites produces uneven fluid surfaces [S1].
*   **Over-smoothing:** Simple Gaussian filters cause over-smoothing and loss of edges [S1].
*   **Bilateral Filter Artifacts:** Bilateral Gaussian filters are not separable; approximate separation results in visual artifacts [S1].
*   **Ellipse Boundary Derivation:** The boundary of the resulting ellipse in clip space cannot be derived using traditional methods because points on the resulting surface may not lie on the projected plane ($x_{c,z} \neq x_{ani,c,z}$) [S1].
*   **Performance:** Increasing simulation resolution for smoother edges requires complex computations that affect real-time performance [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by the WPCA calculation for each particle (neighbor search and SVD) and the screen-space filtering. The anisotropic transformation adds overhead to the rasterization step compared to simple spheres.
*   **Memory Complexity:** Requires storage for depth, thickness, and normal buffers, plus particle data.
*   **Performance:** Designed for real-time applications (video games, VR) [S1]. Experiments on AMD Ryzen 7 5800H and NVIDIA RTX 3060 show efficient performance [S1].
*   **Strengths:** Preserves sharp high-frequency details while smoothing jagged edges; avoids over-smoothing of simple Gaussian filters and excessive flattening of bilateral filters [S1]. WPCA handles non-uniform particle distributions better than traditional PCA [S1].
*   **Limitations:** Sensitive to the choice of smoothing kernel parameters; complex boundary derivation for ellipses [S1].

## Background
*   **Origin:** Proposed in 2022 by Ban and Wang [S1].
*   **History:** Builds on screen-space fluid rendering techniques by Müller et al. [2] and improvements by Truong and Yuksel [16] and Oliveira and Paiva [17]. It addresses the limitations of isotropic point sprites and standard filters in real-time fluid visualization.
*   **Usage:** Real-time graphics applications such as video games and virtual reality [S1].

## Sources
[S1] Anisotropic screen space rendering for particle-based fluid simulation, https://zenodo.org/records/10018402, fetched 2026-10-26
