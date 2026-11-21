# Heat Kernel Textures (HKTex)

Also called: HKTex, Geodesic Gaussians

## Overview
Heat Kernel Textures (HKTex) is an intrinsic, UV-free texture representation that uses anisotropic heat kernels as geodesic equivalents to Gaussians, defined directly on any manifold surface discretized as a triangular mesh [S1]. It solves the problems of UV mapping—such as seams, distortions, wasted memory, and vertex duplication—by operating natively on the manifold rather than in ambient Euclidean space [S1]. Unlike standard 3D Gaussian Splatting, which detaches from geometry, HKTex kernels bend to conform to the surface, and unlike UV mapping, it eliminates artifacts by construction [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangular mesh $M$ (geometry), and optionally a target UV texture or a set of multi-view images for optimization [S1].
*   **Output:** A set of optimized heat kernel parameters $\mathcal{K} = \{\kappa_s\}_{s=1}^S$ representing the texture, integrated into a physically based rendering (PBR) pipeline [S1].

**Data Structures:**
Each kernel $\kappa_s$ is defined by the following parameters:
*   $p_\star$: Position of the heat source on the manifold [S1].
*   $\theta$: Angle of anisotropy [S1].
*   $\eta$: Anisotropy strength [S1].
*   $\tau$: Threshold controlling the scale of the kernel [S1].
*   $\varsigma$: Sharpness of the kernel border [S1].
*   $c$: RGB color [S1].

**Mathematical Background:**
The heat diffusion on a manifold is governed by the heat equation $u_t = e^{-tL} u_0$, where $L$ is the Laplace-Beltrami Operator (LBO) [S1]. The solution is approximated spectrally as:
$$
\mathbf{u}_t = e^{-t\mathbf{L}} \mathbf{u}_0 \approx \mathbf{\Phi} e^{-t\mathbf{\Lambda}} \mathbf{\Phi}^T \mathbf{M} \mathbf{u}_0
$$
where $\mathbf{M}$ is the mass matrix, $\mathbf{\Phi}$ are eigenvectors, and $\mathbf{\Lambda}$ are eigenvalues [S1]. The stiffness matrix weights $w_{ij}$ for the LBO are defined as [S1]:
$$
w_{ij} = \begin{cases} 
-\frac{1}{2} \left[ \frac{\langle \hat{e}_{kj}, \hat{e}_{ki} \rangle_{\mathbf{H}_{\eta, \theta}}}{\sin(\alpha_{ij})} + \frac{\langle \hat{e}_{hj}, \hat{e}_{hi} \rangle_{\mathbf{H}_{\eta, \theta}}}{\sin(\beta_{ij})} \right] & (i,j) \in \text{Edges}, \\
-\sum_{k \neq i} w_{ik} & i=j, \\
0 & \text{else}.
\end{cases}
$$
The inner products are weighted by a shear matrix $\mathbf{H}_{\eta,\theta} = \mathbf{R}_\theta \mathbf{U}_{ijk} \mathbf{D}_\eta \mathbf{U}_{ijk}^T \mathbf{R}_\theta^T$, where $\mathbf{U}_{ijk}$ is a local reference frame, $\mathbf{R}_\theta$ is a rotation matrix, and $\mathbf{D}_\eta = \text{diag}(\frac{1+\eta}{1}, 1, 1)$ controls anisotropy [S1].

**Algorithm Steps:**

1.  **Initialization:**
    *   **Multi-view fitting:** Initialize $S$ kernel positions uniformly over the mesh surface. Initialize colors $c$ randomly from a uniform distribution [S1].
    *   **UV-texture fitting:** Initialize 50% of positions uniformly and 50% based on UV-texture image gradients, transferring sampled 2D points to the manifold. Initialize colors from the UV texture at corresponding coordinates [S1].
    *   **Parameter Initialization:** Let $u \sim U(0,1)$.
        *   $\theta = u \cdot \pi$ [S1].
        *   $\eta = 1.0 + 99.0 \cdot u$ [S1].
        *   $\varsigma = 10 + 40 \cdot u^{0.5}$ (range [10, 50]) [S1].
        *   $\tau = \tau_{\min} + (1.0 - \tau_{\min} - 10^{-8}) \cdot u^{0.7}$ (range $[\tau_{\min}, 1.0)$), where $\tau_{\min}=0.7$ for UV and $0.9$ for multi-view [S1].

2.  **Mass Estimation (Kernel Density Estimation):**
    *   Compute pairwise biharmonic distances $D$ between all sources [S1].
    *   Compute adaptive bandwidth $\sigma_m$ as the median of all non-zero pairwise distances [S1].
    *   Compute local density for each source $s$: $\varrho_s = \sum_{i=1}^S \exp(-D_{i,s}^2 / \sigma_m^2)$ [S1].
    *   Define masses as inverse density: $m_s = 1/\varrho_s$ [S1].
    *   Normalize masses so $\sum m_s$ equals the total surface area [S1].

3.  **Optimization Loop:**
    *   **Importance Sampling:** Pre-sample a pool of 10M points uniformly on the mesh [S1].
        *   For the first 250 steps, draw samples uniformly from the pool [S1].
        *   After warm-up, update sampling weights $\epsilon(p)$ using exponential moving average: $\epsilon_{\text{new}}(p) = \gamma \epsilon_{\text{old}}(p) + (1 - \gamma) L(p)$ with $\gamma = 0.99$, where $L(p)$ is photometric loss [S1].
    *   **Ray Sampling (Multi-view):**
        *   Sample random cameras (azimuth, elevation, distance) [S1].
        *   Generate primary rays per pixel; resample until intersection with the object [S1].
        *   Concatenate, shuffle, and partition rays into mini-batches of $B=1024$ rays [S1].
        *   Re-evaluate each ray with $N_{\text{spp}}=8$ gradient samples [S1].
    *   **Rendering & Loss:**
        *   Evaluate kernels dynamically at ray intersections (not splatted) [S1].
        *   Compute predicted linear RGB $\hat{\mathbf{c}}_i = \frac{1}{N_{\text{spp}}} \sum_{s=1}^{N_{\text{spp}}} \mathbf{L}_{i,s}$ [S1].
        *   Clip prediction and target to $[0, 1]$ and convert to gamma space (unless HDR) [S1].
        *   Compute per-ray L1 loss averaged across the mini-batch [S1].
    *   **Update & Projection:**
        *   Perform gradient descent step [S1].
        *   **Projected Gradient Descent (PGD):** Clamp parameters to valid ranges:
            *   $\theta \in [0, \pi]$ [S1].
            *   $\varsigma \in [10, 200]$ [S1].
            *   $\tau \in [0.3, 1.0 - 10^{-8}]$ [S1].
            *   $\eta \in [1, 200]$ [S1].
            *   Individual kernel colors $c \in [-1, 1]^3$ [S1].
            *   Global mean color $c_{\text{base}} \in [0, 1]^3$ [S1].

4.  **Adaptive Density Control (Kernel Splitting):**
    *   If a kernel requires splitting, interpolate the principal direction $\hat{v}_M$ at the old position $p_{\text{old}}$ using barycentric interpolation from mesh vertices [S1].
    *   Define new major axis $\hat{d}$ by rotating $\hat{v}_M$ by $\theta_{\text{old}} + \pi/2$ around face normal $\hat{n}$:
        $$
        \hat{d} = \hat{v}_M \cos(\theta_{\text{old}} + \pi/2) + (\hat{n} \times \hat{v}_M) \sin(\theta_{\text{old}} + \pi/2)
        $$
    *   Remove old kernel. Create two new kernels displaced along $\hat{d}$ using the exponential map:
        $$
        p_{\text{new}}^{(\pm)} = \text{Exp}(p_{\text{old}}, \pm \hat{d} (1 - \tau_{\text{old}}) r_{\text{base}})
        $$
        where $r_{\text{base}}$ is a base scaling radius [S1].
    *   Shrink new kernels by reducing spatial extents by factor 1.6:
        $$
        \tau_{\text{new}} = 1.0 - 1.0^{-\tau_{\text{old}}/1.6}
        $$
    *   Zero out optimizer states (momentum/variance) for new kernels [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_hktex(mesh, num_kernels, mode='multi_view'):
    """
    Initialize HKTex parameters on a mesh.
    mode: 'multi_view' or 'uv_texture'
    """
    kernels = []
    for s in range(num_kernels):
        u = random_uniform()
        
        # Position
        if mode == 'multi_view':
            p_star = sample_uniform_on_surface(mesh)
            c = random_uniform_color()
        else: # uv_texture
            if s < num_kernels / 2:
                p_star = sample_uniform_on_surface(mesh)
                c = random_uniform_color()
            else:
                # Sample based on UV texture gradients
                uv_point = sample_uv_gradient(mesh.uv_map)
                p_star = uv_to_surface_point(mesh, uv_point)
                c = sample_uv_color(mesh.uv_map, uv_point)
        
        # Parameters
        theta = u * pi
        eta = 1.0 + 99.0 * u
        varsigma = 10 + 40 * (u ** 0.5)
        
        if mode == 'uv_texture':
            tau_min = 0.7
        else:
            tau_min = 0.9
            
        tau = tau_min + (1.0 - tau_min - 1e-8) * (u ** 0.7)
        
        kernels.append({
            'p_star': p_star,
            'theta': theta,
            'eta': eta,
            'tau': tau,
            'varsigma': varsigma,
            'c': c,
            'optimizer_state': None # Placeholder for momentum/variance
        })
    return kernels

def compute_masses(kernels, mesh):
    """
    Compute masses for kernels using KDE on biharmonic distances.
    """
    # Compute pairwise biharmonic distances D
    D = compute_biharmonic_distances([k['p_star'] for k in kernels])
    
    # Adaptive bandwidth
    nonzero_dists = [d for d in D.flatten() if d > 0]
    sigma_m = median(nonzero_dists)
    
    # Local density
    densities = []
    for s in range(len(kernels)):
        # Sum exp(-D_i_s^2 / sigma_m^2)
        dists_to_s = D[:, s]
        density = sum(exp(-(d ** 2) / (sigma_m ** 2)) for d in dists_to_s)
        densities.append(density)
        
    # Inverse density
    masses = [1.0 / d for d in densities]
    
    # Normalize to surface area
    total_area = mesh.total_surface_area()
    current_sum = sum(masses)
    masses = [m * (total_area / current_sum) for m in masses]
    
    return masses

def optimize_hktex(mesh, kernels, masses, num_steps, num_kernels):
    """
    Main optimization loop.
    """
    # Pre-sample pool for importance sampling
    pool = sample_uniform_points_on_surface(mesh, count=10_000_000)
    epsilon = zeros(len(pool)) # Sampling weights
    
    for step in range(num_steps):
        # 1. Sampling
        if step < 250:
            # Uniform sampling from pool
            ray_indices = random_sample(pool, size=1024)
        else:
            # Importance sampling based on epsilon
            ray_indices = weighted_sample(pool, weights=epsilon, size=1024)
            # Update epsilon
            # Note: L(p) must be computed during rendering
            pass 
            
        # 2. Camera/Ray Generation (Multi-view specific)
        rays = []
        for _ in range(1024):
            cam = sample_random_camera()
            ray = generate_ray_from_camera(cam, pixel=random_pixel())
            while not ray_intersects_mesh(ray, mesh):
                cam = sample_random_camera()
                ray = generate_ray_from_camera(cam, pixel=random_pixel())
            rays.append(ray)
            
        # 3. Rendering and Loss
        loss = 0.0
        for i, ray in enumerate(rays):
            # Evaluate HKTex at ray intersection
            # This involves dynamic evaluation of heat kernels, not splatting
            # N_spp = 8 gradient samples
            samples = []
            for _ in range(8):
                # Evaluate kernel contribution at intersection point
                # Using spectral heat kernel solution or continuous manifold formulation
                color = evaluate_hktex_at_point(mesh, kernels, masses, ray.intersection_point)
                samples.append(color)
                
            predicted_color = mean(samples)
            
            # Get target color (from GT image or UV texture)
            target_color = get_target_color(ray)
            
            # Preprocessing
            pred_clipped = clip(predicted_color, 0, 1)
            target_clipped = clip(target_color, 0, 1)
            pred_gamma = linear_to_gamma(pred_clipped)
            target_gamma = linear_to_gamma(target_clipped)
            
            # L1 Loss
            loss += l1_loss(pred_gamma, target_gamma)
            
            # Update importance sampling weights for the intersection point
            # (Simplified: assume we track loss per pool point)
            # epsilon[pool_index] = gamma * epsilon[pool_index] + (1-gamma) * loss
            
        loss /= len(rays)
        
        # 4. Backpropagation and Update
        gradients = compute_gradients(loss, kernels)
        update_optimizer(kernels, gradients)
        
        # 5. Projected Gradient Descent (Clamping)
        for k in kernels:
            k['theta'] = clamp(k['theta'], 0, pi)
            k['varsigma'] = clamp(k['varsigma'], 10, 200)
            k['tau'] = clamp(k['tau'], 0.3, 1.0 - 1e-8)
            k['eta'] = clamp(k['eta'], 1, 200)
            k['c'] = clamp(k['c'], -1, 1)
            
        # 6. Adaptive Density Control (Splitting)
        # Check if splitting is needed based on density/loss criteria
        # If split:
        #   interpolate principal direction v_hat_M
        #   compute new axis d_hat
        #   create new kernels at Exp(p_old, +/- d_hat * (1-tau) * r_base)
        #   set new tau = 1.0 - 1.0^(-tau_old/1.6)
        #   zero optimizer states for new kernels
        pass

def evaluate_hktex_at_point(mesh, kernels, masses, point):
    """
    Dynamically evaluate the texture color at a point on the mesh.
    """
    # Find contributing kernels (e.g., within a certain geodesic distance)
    contributing = []
    for k in kernels:
        # Compute geodesic distance from point to k['p_star']
        dist = geodesic_distance(mesh, point, k['p_star'])
        # Check if within effective radius based on tau and eta
        if dist < effective_radius(k['tau'], k['eta']):
            contributing.append(k)
            
    if not contributing:
        return [0.0, 0.0, 0.0]
        
    # Compute heat kernel value for each contributing kernel
    # This uses the spectral or continuous manifold formulation
    total_color = [0.0, 0.0, 0.0]
    total_weight = 0.0
    
    for k in contributing:
        # Evaluate heat kernel Phi(x, p_star, t)
        # t is related to tau and varsigma
        # Anisotropy is handled by the shear matrix H_eta,theta in the LBO
        value = evaluate_heat_kernel(mesh, k['p_star'], point, k['eta'], k['theta'], k['varsigma'])
        
        # Weight by mass
        weight = value * k['mass'] # mass is precomputed or passed
        total_color += weight * k['c']
        total_weight += weight
        
    if total_weight == 0:
        return [0.0, 0.0, 0.0]
        
    return [c / total_weight for c in total_color]
```

## Operating characteristics
*   **Time Complexity:** The spectral approach for solving heat diffusion is one order of magnitude faster than simulating diffusion by solving a sparse linear system, scaling to high-resolution meshes [S1].
*   **Memory Complexity:** Eliminates UV coordinates and vertex duplication, lowering memory footprint compared to UV mapping [S1].
*   **Strengths:** Intrinsic to geometry, no UV seams/distortions, differentiable, integrates with PBR [S1].
*   **Limitations:** Requires a triangular mesh; optimization is constrained to the surface [S1].
*   **When to use:** When UV mapping artifacts are unacceptable, or for compressing existing UV textures and inverse rendering from 2D photos [S1].
*   **When not to use:** If the input is not a manifold mesh or if ambient Euclidean operations are required [S1].

## Background
HKTex was introduced by Foti et al. [S1]. It addresses the limitations of UV mapping and standard 3D Gaussian Splatting (which operates in Euclidean space) by using discrete Riemannian geometry to define textures on manifolds [S1]. It is applicable to games, VR, AR, and VFX [S1].

## Sources
[S1] Heat Kernel Textures the Geodesic Gaussians That Do Not Splat, https://doi.org/10.1007/978-3-032-37595-7_17, fetched 2026-10-29
