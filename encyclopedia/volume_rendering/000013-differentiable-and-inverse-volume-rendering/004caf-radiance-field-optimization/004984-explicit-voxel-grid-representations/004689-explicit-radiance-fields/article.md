# Explicit Radiance Fields

Also called: Explicit Radiance Fields Reconstruction via Gauss-Newton, Non-linear least-squares volumetric reconstruction, PERF (Performant Explicit Radiance Fields), PERF reconstruction, Volumetric reconstruction with explicit voxels

## Overview
Explicit Radiance Fields (specifically the PERF method) solve the problem of image-based 3D reconstruction by formulating volumetric reconstruction as a non-linear least-squares optimization problem solved directly on a discrete voxel grid, bypassing neural networks [S2]. Unlike implicit neural representations or methods relying on stochastic gradient descent, this approach uses a Gauss-Newton solver with Preconditioned Conjugate Gradient (PCG) to achieve convergence rates 2–3 orders of magnitude faster than NeRF, while maintaining comparable reconstruction quality [S2]. It distinguishes itself by combining an explicit voxel grid for the foreground with a hierarchical envelope of environment maps to handle 360° scenes and transparent boundaries [S2].

## How it works
**Inputs:** A set of calibrated images of a scene.
**Outputs:** A 3D voxel grid storing color and opacity values, and a set of hierarchical environment maps storing color and opacity.

### Data Structures
1.  **Voxel Grid:** A 3D array where each cell stores:
    *   Color: 3 floating-point values ($C = \{c_r, c_g, c_b\}$).
    *   Differential Opacity: 1 floating-point value ($\sigma$).
    *   Storage: Stored in a CUDA 3D surface for hardware trilinear interpolation [S2]. Backing storage is copied to regular memory for kernels requiring atomics [S2].
2.  **Environment Maps:** A hierarchical set of cubemaps surrounding the scene.
    *   Structure: Each cubemap face stores color and opacity per texel.
    *   Hierarchy: Distance between maps increases with the square of the distance to the voxel grid [S2].
    *   Storage: Stored in CUDA cube maps [S2].

### Formulas
The method minimizes the sum of squared differences between accumulated ray colors and actual pixel colors.

**Discrete Volume Rendering Equation (Accumulated Color $\hat{H}$):**
$$ \hat{H} = \sum_{t=t_n}^{t_f} \hat{T}(t)(1 - e^{-\sigma(t)\delta(t)})c(t) $$
where $\hat{T}(t) = e^{-\sum_{s=t}^{t_f} \sigma(s)\delta(s)}$ is the accumulated transmittance [S2].

**Objective Function (Non-linear Least-Squares):**
$$ S(x) = 0.5 \sum_{i=1}^{m} r_i(x)^2 $$
where $r_i$ are residuals. For volumetric reconstruction, the specific objective function is:
$$ F[c(t), \sigma(t)] = 0.5 \sum_{t=t_n}^{t_f} \sum_{i=1}^{3} (\hat{H}_i[c_i(t), \sigma(t)] - C_i)^2 $$
where $\hat{H}_i$ is the accumulated color for channel $i$, and $C_i$ is the actual pixel color [S2].

**Gauss-Newton Update:**
$$ x_{i+1} = x_i - (J_r^T J_r)^{-1} J_r^T r(x_i) = x_i + \Delta $$
where $J_r$ is the Jacobian of the residuals, and $\Delta$ is the update step [S2].

**Auxiliary Opacity Penalty ($L$):**
To mitigate view-dependent highlight artifacts (smoke-like artifacts), an auxiliary term penalizes accumulated opacity $\hat{T}$ if it deviates from 0 (transparent) or 1 (opaque):
$$ L = \lambda [-4(\hat{T} - 0.5)^2 + 1] $$
where $\lambda = 0.1$ [S2].

### Parameters
*   **Hierarchical Levels:** 4–5 levels. Resolution doubles per level.
*   **Initial Resolution:** $32^3$ for voxel grid, $32 \times 32$ for cubemaps.
*   **Final Grid Resolution:** Typically $256^3$ or $512^3$ [S2].
*   **Iterations per Level:** $N=30$ [S2].
*   **PCG Iterations:** Typically 1–3 iterations. Exit early if convergence rate is low ($R/R_{prev} > 0.85$) [S2].
*   **Geometrical Decimation Factor ($\alpha$):** 0.7 [S2].
*   **Opacity Threshold:** During rendering of converged scene, if $(1.0 - \hat{T}) < 0.7$, volume is considered fully transparent [S2].
*   **Image Resolution:** Synthetic scenes at half resolution; 360° scenes at quarter resolution [S2].
*   **Environment Maps:** 10 maps surrounding 360° scenes. Resolution $256^2$ or $512^2$ per face [S2].

### Algorithm Steps

1.  **Initialization:**
    *   Initialize voxel grid with random colors and low opacity.
    *   Initialize environment maps similarly [S2].

2.  **Hierarchical Optimization Loop:**
    *   For each hierarchical level (from lowest to highest resolution):
        *   For $k = 1$ to $N$ (iterations):
            1.  **Ray Casting & Jittering:**
                *   For each pixel, choose ray origin using two uniformly distributed floats [S2].
                *   Ray-march through the voxel grid. Step length equals the shortest side of a voxel cell.
                *   Jitter sample positions within half a raymarch step before and after uniform steps [S2].
                *   Compute accumulated color $\hat{H}$ using the discrete volume rendering equation [S2].
            2.  **Environment Map Integration:**
                *   Upon exiting the voxel grid, perform intersection tests with environment maps from inner to outer.
                *   Accumulate contribution similarly to the voxel grid [S2].
            3.  **Residual Calculation:**
                *   Compute residuals $r_i = \hat{H}_i - C_i$ for each color channel [S2].
                *   Compute auxiliary penalty $L$ if applicable [S2].
            4.  **Jacobian Computation (Analytical Derivatives):**
                *   Compute partial derivatives $\frac{\partial \hat{H}}{\partial c_i}$ and $\frac{\partial \hat{H}}{\partial \sigma_i}$ using the iterative backward algorithm (see Pseudocode below) [S2].
            5.  **Gauss-Newton Step via PCG:**
                *   Solve $A \Delta = b$ where $A = J_r^T J_r$ and $b = -J_r^T r$ using Preconditioned Conjugate Gradient (PCG).
                *   Compute $Ap$ using the identity $Ap = \sum_i \nabla r_i (\nabla r_i^T p)$ to avoid explicit matrix formation [S2].
                *   Use diagonal Jacobi preconditioner $M = \text{diag}(J_r^T J_r)$ [S2].
                *   Apply line search with backtracking (Algorithm 3) to determine step size $\gamma$ [S2].
                *   Update voxel grid and environment map values: $x_{new} = x_{old} + \gamma \Delta$ [S2].
        *   **Level Up:**
            *   Store result temporarily.
            *   Reallocate buffers for the next higher resolution level.
            *   Populate new buffers with old data via linear interpolation [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_derivatives(sigma, delta, c):
    """
    Computes partial derivatives of accumulated color H with respect to 
    color c_i and opacity sigma_i for a single ray traversal.
    sigma: list of opacities along the ray (N elements)
    delta: list of step lengths along the ray (N elements)
    c: list of colors along the ray (N elements, each is a vector)
    Returns:
        dH_dc: list of gradients w.r.t color
        dH_dsigma: list of gradients w.r.t opacity
    """
    N = len(sigma)
    dH_dc = [None] * N
    dH_dsigma = [None] * N
    
    # V represents exp(-sum(sigma_j * delta_j)) for j > i
    # Initialize V = exp(-sum_{j=1 to N} sigma_j * delta_j)
    V = 0.0
    for i in range(N):
        V -= sigma[i] * delta[i]
    V = math.exp(V)
    
    G = 0.0
    
    # Iterate backwards from N to 1 (index N-1 to 0)
    for i in range(N - 1, -1, -1):
        # V corresponds to exp(-sum_{j=i+1 to N} sigma_j * delta_j)
        # Note: In the source algorithm, V is updated by subtracting sigma_i * delta_i
        # before computing derivatives for i. 
        # Let's align with source Algorithm 1 exactly.
        
        # Source: V = V - sigma_i * delta_i
        # This implies V entering the loop body is exp(-sum_{j=i+1 to N} ...)
        # But the initialization V = exp(-sum_{j=1 to N} ...) suggests V starts at the end.
        # Let's trace:
        # Start: V = exp(-sum_{all})
        # i=N-1 (last element): V_new = V_old - sigma[N-1]*delta[N-1] = exp(-sum_{1..N-1})
        # This matches the transmittance BEFORE the last sample.
        
        V = V - sigma[i] * delta[i]
        transmittance_before = math.exp(V)
        absorption = 1.0 - math.exp(-sigma[i] * delta[i])
        
        # dH/dc_i = T_{before} * (1 - exp(-sigma_i * delta_i))
        dH_dc[i] = transmittance_before * absorption
        
        # dH/dsigma_i = G + T_{before} * (1 - exp(-sigma_i * delta_i)) * c_i
        # where G is updated as: G = T_{before} * delta_i * exp(-sigma_i * delta_i) * c_i - delta_i * G
        # Note: The source formula for G update is:
        # G = e^{-V} delta_i e^{-sigma_i delta_i} c_i - delta_i G
        # Here e^{-V} is transmittance_before.
        
        term = transmittance_before * math.exp(-sigma[i] * delta[i]) * delta[i]
        G = term * c[i] - delta[i] * G
        
        dH_dsigma[i] = G + transmittance_before * absorption * c[i]
        
    return dH_dc, dH_dsigma

def pcg_solve(A_p_function, b, x0, M_inv, max_iter=3, eps=1e-6):
    """
    Preconditioned Conjugate Gradient solver for Ax = b.
    A_p_function: computes A*p = J_r^T J_r p
    M_inv: function computing M^-1 * r (Jacobi preconditioner)
    """
    r = b - A_p_function(x0)
    R_prev = dot(r, r)
    z = M_inv(r)
    p = z
    k = 0
    
    while True:
        Ap = A_p_function(p)
        alpha = dot(r, z) / dot(p, Ap)
        x_new = x0 + alpha * p
        r_new = r - alpha * Ap
        R = dot(r_new, r_new)
        
        if R < eps or R / R_prev > 0.85:
            return x_new
            
        R_prev = R
        z_new = M_inv(r_new)
        beta = dot(r_new, z_new) / dot(r, z)
        p = z_new + beta * p
        r = r_new
        x0 = x_new
        k += 1

def line_search_step_size(initial_gamma, alpha_dec, objective_function):
    """
    Backtracking line search.
    """
    gamma = initial_gamma
    mu_prev = float('inf')
    
    while True:
        mu = objective_function(gamma)
        if mu > mu_prev:
            return gamma / alpha_dec
        gamma = alpha_dec * gamma
        mu_prev = mu

def optimize_level(voxel_grid, env_maps, images, iterations=30):
    """
    Optimizes a single hierarchical level.
    """
    for k in range(iterations):
        # 1. Forward Pass: Compute residuals and gradients
        residuals = []
        gradients = [] # List of gradients for each residual
        
        for img in images:
            for pixel in img:
                ray = cast_ray(pixel)
                
                # Ray march voxel grid
                samples = ray_march_grid(ray, voxel_grid)
                
                # Intersect env maps
                env_samples = intersect_env_maps(ray, env_maps)
                
                # Combine samples
                all_samples = samples + env_samples
                
                # Compute accumulated color H_hat
                H_hat = volume_render(all_samples)
                
                # Compute residual
                r = H_hat - pixel.color
                
                # Compute derivatives analytically
                dH_dc, dH_dsigma = compute_derivatives(
                    [s.sigma for s in all_samples],
                    [s.delta for s in all_samples],
                    [s.color for s in all_samples]
                )
                
                # Store residual and gradient info
                # Gradient vector for this residual w.r.t all parameters
                grad = build_gradient_vector(dH_dc, dH_dsigma, all_samples)
                
                residuals.append(r)
                gradients.append(grad)
        
        # 2. Solve Gauss-Newton Step
        # b = -J_r^T r
        b = -sum(dot(g, r) for g, r in zip(gradients, residuals))
        
        # A = J_r^T J_r
        # We solve A * delta = b using PCG
        # A_p(p) = J_r^T J_r p = sum_i grad_i * (grad_i^T p)
        def A_p(p):
            Ap = zeros_like(voxel_grid)
            for grad in gradients:
                dot_prod = dot(grad, p)
                Ap += grad * dot_prod
            return Ap
            
        # Preconditioner M = diag(J_r^T J_r)
        # M_inv(r) = r / diag(A)
        diag_A = sum(dot(g, g) for g in gradients)
        def M_inv(r):
            return r / diag_A
            
        # Initial guess for delta
        x0 = zeros_like(voxel_grid)
        
        # Solve for delta
        delta = pcg_solve(A_p, b, x0, M_inv)
        
        # 3. Line Search
        def objective_with_step(gamma):
            # Update grid with gamma * delta
            # Compute new objective function value
            # (Simplified: assume objective_function handles the update and evaluation)
            return compute_objective(voxel_grid + gamma * delta, images)
            
        gamma = line_search_step_size(1.0, 0.7, objective_with_step)
        
        # 4. Update Grid
        voxel_grid += gamma * delta
        # Update env maps similarly if they are part of the optimization vector x
        # (In PERF, env maps are also optimized)
        
    return voxel_grid, env_maps

def main():
    # Hierarchical levels
    levels = 5
    current_res = 32
    voxel_grid = init_grid(current_res)
    env_maps = init_env_maps(current_res)
    
    for level in range(levels):
        voxel_grid, env_maps = optimize_level(voxel_grid, env_maps, images, iterations=30)
        
        if level < levels - 1:
            # Interpolate to next level
            next_res = current_res * 2
            voxel_grid = interpolate(voxel_grid, next_res)
            env_maps = interpolate_env_maps(env_maps, next_res)
            current_res = next_res
            
    # Post-processing: Apply opacity threshold for rendering
    return voxel_grid, env_maps
```

### Edge Cases and Pitfalls
*   **Matrix Size:** The matrix $J_r^T J_r$ is too large to form explicitly (e.g., 4TB for a $32^3$ grid). The method must use the iterative PCG solver and the identity $Ap = \sum_i \nabla r_i (\nabla r_i^T p)$ [S2].
*   **View-Dependence:** The method does not consider view-dependent effects. It mitigates artifacts (smoke-like highlights) using the auxiliary opacity penalty term $L$ [S2].
*   **Sparsity:** $J_r$ is very sparse. Only voxels/texels hit by a ray contribute to the gradient. This sparsity is exploited for parallelization [S2].
*   **Convergence:** PCG may converge slowly. The algorithm exits PCG early if $R/R_{prev} > 0.85$ and proceeds to the next Gauss-Newton iteration [S2].
*   **Transparent Boundaries:** Without environment maps, transparent voxels at the boundary can cause artifacts. The hierarchical environment maps separate foreground and background [S2].

## Operating characteristics
*   **Time Complexity:** 2–3 orders of magnitude faster than NeRF. Example: Pinecone scene ($512^3$ grid) takes 24m 54s compared to NeRF's 23h 21m [S2].
*   **Memory Complexity:** High memory usage due to explicit voxel storage. A $32^3$ grid results in a 4TB Jacobian matrix if formed explicitly, necessitating iterative solvers [S2].
*   **Quality:** PSNR scores on NERF synthetic holdout views are comparable to NeRF without view-dependence. Lego: 31.08 (Ours) vs 30.48 (NeRF no view-dep); Ficus: 27.54 (Ours) vs 26.89 (NeRF no view-dep) [S2].
*   **Strengths:** Fast convergence, real-time rendering during optimization, explicit representation allows direct inspection.
*   **Limitations:** No view-dependent effects, high memory usage, lower resolution compared to implicit methods for the same memory budget [S1, S2].
*   **When to use:** When fast reconstruction of 360° scenes is required and view-dependent effects are not critical.
*   **When not to use:** When high-resolution view-dependent effects are needed, or memory is severely constrained.

## Background
Explicit Radiance Fields, specifically the PERF method, were introduced by Sverker Rasmuson, Erik Sintorn, and Ulf Assarsson from Chalmers University of Technology in 2022 [S2]. The work formulates volumetric reconstruction as a non-linear least-squares problem, leveraging the Gauss-Newton solver for faster convergence than the stochastic gradient descent used in NeRF [S2]. It builds on the concept of explicit voxel grids, similar to Plenoxels, but adds a hierarchical environment map structure for 360° scenes [S1, S2].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] PERF: Performant, Explicit Radiance Fields, https://doi.org/10.3389/fcomp.2022.871808, fetched 2026-10-26
