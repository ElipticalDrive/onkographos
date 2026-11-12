# Explicit Radiance Fields

Also called: Explicit Radiance Fields Reconstruction via Gauss-Newton, Non-linear least-squares volumetric reconstruction, PERF (Performant Explicit Radiance Fields), PERF reconstruction, Volumetric reconstruction with explicit voxels

## Overview
Explicit Radiance Fields (specifically the PERF method) reconstruct 3D scenes by formulating volumetric reconstruction as a non-linear least-squares problem solved explicitly using a Gauss-Newton solver, avoiding neural networks [S2]. It represents the scene as a discrete voxel grid for the foreground and a hierarchy of environment maps for the background [S2]. Unlike implicit radiance fields that use MLPs and Adam optimization, this method achieves significantly faster convergence (2-3 orders of magnitude) by leveraging analytical derivatives and iterative linear solvers, while maintaining quality comparable to state-of-the-art neural methods [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $M$ images with known camera poses. Each image has resolution $W \times H$.
*   **Outputs:** A voxel grid storing color and opacity values, and a hierarchy of environment maps storing color and opacity.

**Data Structures:**
1.  **Voxel Grid:** A 3D grid where each cell stores:
    *   Color: $C = \{c_r, c_g, c_b\}$ (3 floats).
    *   Differential Opacity: $\sigma$ (1 float).
    *   Storage: Stored in a 3D CUDA surface for hardware trilinear interpolation [S2]. Backing storage is copied to regular memory for kernels using atomics [S2].
2.  **Environment Maps:** A hierarchical set of cubemaps surrounding the scene.
    *   Structure: 10 maps for 360° scenes [S2].
    *   Content: Color and opacity per texel.
    *   Layout: Distance of maps increases with the square of the distance to the voxel grid [S2].
    *   Storage: Cubemaps [S2].

**Rendering (Forward Model):**
For a ray $\mathbf{r}$, the accumulated color $\hat{H}$ is computed by ray-marching the voxel grid and intersecting environment maps [S2].

1.  **Voxel Grid Integration:** Ray-march through the grid with step length equal to the shortest side of a voxel cell. Samples are jittered: ray origin is chosen randomly using two uniform floats; sample positions are randomized within half a raymarch step [S2].
    The accumulated color is:
    $$
    \hat{H} = \sum_{t=t_n}^{t_f} \hat{T}(t)(1 - e^{-\sigma(t)\delta(t)})c(t)
    $$
    where $\hat{T}(t) = e^{-\sum_{s=t_n}^{t} \sigma(s)\delta(s)}$ [S2].

2.  **Environment Map Integration:** Upon exiting the voxel grid, intersect the ray with environment maps in order from inner to outer. Accumulate contribution similarly to the voxel grid [S2].

**Optimization Objective:**
Minimize the sum of squared differences between accumulated ray colors and actual pixel colors [S2]:
$$
F[c(t), \sigma(t)] = 0.5 \sum_{t=t_n}^{t_f} \sum_{i=1}^{3} (\hat{H}_i[c_i(t), \sigma(t)] - C_i)^2
$$
where $\hat{H}_i$ is the accumulated color for channel $i$, and $C_i$ is the actual pixel color [S2]. This is a non-linear least-squares problem $S(x) = 0.5 \sum r_i(x)^2$ [S2].

**Auxiliary Term:**
To mitigate view-dependent highlight artifacts (smoke-like artifacts), an opacity penalty is added [S2]:
$$
L = \lambda [-4(\hat{T} - 0.5)^2 + 1]
$$
where $\lambda = 0.1$ and $\hat{T}$ is the accumulated opacity [S2].

**Solver (Gauss-Newton with PCG):**
The update step is $x_{i+1} = x_i + \Delta$, where $\Delta$ solves $A \Delta = b$ with $A = J_r^T J_r$ and $b = -J_r^T r$ [S2]. $J_r$ is the Jacobian of residuals. Since $A$ is too large to form explicitly (e.g., 4TB for $32^3$ grid), the Preconditioned Conjugate Gradient (PCG) algorithm is used [S2].

1.  **Derivative Computation:** Partial derivatives $\frac{\partial H}{\partial c_i}$ and $\frac{\partial H}{\partial \sigma_i}$ are computed analytically using an iterative backward algorithm [S2].

2.  **PCG Iteration:**
    *   Preconditioner $M$: Diagonal Jacobi preconditioner, $M = \text{diag}(J_r^T J_r) = \sum_i (\frac{\partial r_i}{\partial x_j})^2$ [S2].
    *   Matrix-vector product $Ap$: Computed via $Ap = \sum_i \nabla r_i (\nabla r_i^T p)$ to exploit sparsity and enable parallelization [S2].
    *   Iterations: Typically 1-3 PCG iterations per Gauss-Newton step. Early exit if convergence rate is low ($R/R_{prev} > 0.85$) [S2].

3.  **Line Search:** Backtracking line search with geometric decimation factor $\alpha = 0.7$ to ensure step decrease [S2].

**Hierarchical Optimization:**
*   Levels: 4-5 levels [S2].
*   Resolution: Doubles per level. Initial: $32^3$ voxels, $32 \times 32$ cubemaps. Final: $256^3$ or $512^3$ [S2].
*   Process: Run $N=30$ iterations per level. Store result, reallocate buffers, populate new buffers with old data via linear interpolation, resume [S2]. Initial values are random colors with low opacity [S2].

**Rendering Post-Processing:**
For converged scenes, apply opacity threshold: if $(1.0 - \hat{T}) < 0.7$, volume is considered fully transparent [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Algorithm 1: Compute partial derivatives for a single ray
def compute_derivatives(sigma, delta, c, N):
    """
    sigma: array of densities [sigma_1, ..., sigma_N]
    delta: array of step sizes [delta_1, ..., delta_N]
    c: array of colors [c_1, ..., c_N]
    N: number of samples
    Returns: gradients dH_dc, dH_dsigma
    """
    V = exp(-sum(sigma[i] * delta[i] for i in range(N)))
    G = 0.0
    dH_dc = [0.0] * N
    dH_dsigma = [0.0] * N
    
    for i in range(N - 1, -1, -1):
        V = V - sigma[i] * delta[i]
        term = exp(-V) * (1 - exp(-sigma[i] * delta[i]))
        dH_dc[i] = term
        dH_dsigma[i] = G + term * c[i]
        G = exp(-V) * delta[i] * exp(-sigma[i] * delta[i]) * c[i] - delta[i] * G
        
    return dH_dc, dH_dsigma

# Algorithm 2: Preconditioned Conjugate Gradient (PCG)
def pcg_solve(A_func, b, M_inv, x0, max_iter=3, eps=1e-6):
    """
    A_func: function that computes A*p = J_r^T J_r * p
    b: right hand side vector (-J_r^T r)
    M_inv: function that applies inverse preconditioner (vector addition/division)
    x0: initial guess
    max_iter: max PCG iterations
    Returns: solution x
    """
    r = b - A_func(x0)
    R_prev = dot(r, r)
    z = M_inv(r)
    p = z
    k = 0
    
    while True:
        Ap = A_func(p)
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

# Algorithm 3: Backtracking Line Search
def line_search(gamma, alpha, error_func):
    """
    gamma: initial step size
    alpha: decimation factor (0.7)
    error_func: function evaluating objective F for step gamma
    Returns: scaled step size
    """
    mu_prev = float('inf')
    while True:
        mu = error_func(gamma)
        if mu > mu_prev:
            return gamma / alpha
        gamma = alpha * gamma
        mu_prev = mu

# Main Optimization Loop
def optimize_scene(images, initial_voxels, initial_env_maps):
    """
    images: list of (pixel_colors, camera_poses)
    initial_voxels: voxel grid data
    initial_env_maps: environment map data
    """
    # Hierarchical levels
    levels = 5
    current_voxels = initial_voxels
    current_env_maps = initial_env_maps
    
    for level in range(levels):
        # Setup grid resolution for this level
        # ... (resolution doubling logic)
        
        # Initialize x (all color and opacity values)
        x = flatten_data(current_voxels, current_env_maps)
        
        # Gauss-Newton iterations
        for gn_iter in range(30):
            # Compute residuals r and Jacobian components
            # r_i = H_hat_i - C_i
            # Compute gradients for each ray using Algorithm 1
            # Accumulate J_r^T r (b) and J_r^T J_r (A) implicitly
            
            b = -sum(gradient_r_i for all rays)
            
            # Define A_func for PCG: A*p = sum(gradient_r_i * dot(gradient_r_i, p))
            def A_func(p):
                Ap = zeros_like(p)
                for ray in rays:
                    grad = compute_derivatives(...) # Algorithm 1
                    dot_prod = dot(grad, p)
                    Ap += grad * dot_prod
                return Ap
            
            # Preconditioner M_inv: diagonal inverse
            M_diag = sum(gradient_r_i^2 for all rays)
            def M_inv(v):
                return v / M_diag
            
            # Solve for delta
            delta = pcg_solve(A_func, b, M_inv, x)
            
            # Line search
            step = line_search(1.0, 0.7, lambda g: evaluate_error(x + g * delta))
            
            # Update
            x = x + step * delta
            
        # Interpolate result to next level resolution
        current_voxels = interpolate_to_next_level(current_voxels)
        current_env_maps = interpolate_to_next_level(current_env_maps)
        
    return current_voxels, current_env_maps
```

## Operating characteristics
*   **Time Complexity:** 2-3 orders of magnitude faster than NERF reference [S2]. Example: Pinecone scene $512^3$ grid: 24m 54s vs 23h 21m [S2].
*   **Memory Complexity:** High memory usage due to explicit voxel storage. A $32^3$ grid requires 4TB for the Hessian matrix $J_r^T J_r$ if formed explicitly, but PCG avoids this [S2]. Final resolutions are typically $256^3$ or $512^3$ [S2].
*   **Performance:** Measured on Nvidia Titan V GPU [S2]. Synthetic scenes at half resolution, 360° scenes at quarter resolution [S2].
*   **Strengths:** Fast convergence, real-time rendering during training, explicit separation of foreground/background for 360° scenes [S2].
*   **Limitations:** Does not consider view-dependent effects [S2]. High memory usage for high-resolution grids [S1]. Potential for smoke-like artifacts if opacity threshold is not applied [S2].
*   **When to use:** When fast reconstruction is prioritized over view-dependent appearance, and for 360° scenes requiring background separation [S2].
*   **When not to use:** When view-dependent effects are critical [S2].

## Background
Originated from the paper "PERF: Performant, Explicit Radiance Fields" by Sverker Rasmuson, Erik Sintorn, and Ulf Assarsson (Chalmers University of Technology, 2022) [S2]. It addresses the slow convergence of neural radiance fields (NeRF) by identifying volumetric reconstruction as a non-linear least-squares problem solvable with Gauss-Newton methods [S2]. It builds on the principles of differentiable volume rendering [44-radiance-field-optimization.md].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] PERF: Performant, Explicit Radiance Fields, https://doi.org/10.3389/fcomp.2022.871808, fetched 2026-10-26
