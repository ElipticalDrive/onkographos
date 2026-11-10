# Coupled Density and Velocity Estimation

Also called: Coupled Estimation, Coupled Estimation Strategy, Physics-Based Fluid Reconstruction

## Overview
Coupled Density and Velocity Estimation is a method for reconstructing fluid flow fields from sparse multi-view video by simultaneously estimating density and velocity fields while enforcing physical constraints [S1][S2, unofficial]. It solves the ill-posed tomography problem that arises when the number of cameras is reduced, enabling efficient interactive reconstruction of realistic and smooth fluid motion [S1][S2, unofficial]. Unlike standard tomographic reconstruction which treats density as static, this method couples density and velocity updates using transport and incompressibility priors, significantly reducing reconstruction time compared to earlier coupled methods (e.g., from 50–60 minutes per frame to interactive speeds) [S1][S2, unofficial].

## How it works
The method operates on a sequence of time steps $t$. It assumes density fields $\rho_{t-1}$ and $\rho_{t-2}$ are available from previous reconstruction steps [S1][S2, unofficial]. The core process involves estimating the velocity field $u_{t-1}$ from these densities, predicting the next state $(\tilde{u}_t, \tilde{\rho}_t)$ using physical advection, and then refining the density estimate [S1][S2, unofficial].

### Inputs and Outputs
- **Inputs**: Density fields $\rho_{t-1}$ and $\rho_{t-2}$ at time steps $t-1$ and $t-2$; time step size $\Delta t$; regularization parameters $\alpha, \beta$; primal-dual solver parameters $\sigma, \gamma, \theta$ [S2, unofficial].
- **Outputs**: Predicted density field $\tilde{\rho}_t$ and velocity field $\tilde{u}_t$ for time step $t$ [S1][S2, unofficial].

### Data Structures
- **Density Field**: A 3D voxel grid $\rho \in \mathbb{R}^{N_x \times N_y \times N_z}$ [S2, unofficial].
- **Velocity Field**: A 3D vector field $u \in \mathbb{R}^{3 \times N_x \times N_y \times N_z}$, where each voxel stores $(u_x, u_y, u_z)$ [S2, unofficial].
- **Helper Variables**: For the primal-dual solver, variables $X, Y$ (same shape as velocity field) and iterates $u^{t-1}_k$ [S2, unofficial].

### Velocity Reconstruction (Algorithm 5)
The velocity field $u_{t-1}$ is estimated by solving a constrained optimization problem [S2, unofficial].

**Objective Function**:
$$
\min_{u_{t-1}} f(u_{t-1}) = \left\| \frac{\rho_{t-1} - \rho_{t-2}}{\Delta t} + u_{t-1} \cdot \nabla \rho_{t-1} \right\|^2 + E_{smooth}(u_{t-1}) + E_{kinetic}(u_{t-1})
$$
subject to:
$$
\nabla \cdot u_{t-1} = 0
$$
where:
- The first term enforces the optical flow constraint (continuity equation linearized): $\frac{\partial \rho}{\partial t} + \nabla \cdot (\rho u) \approx 0$ [S2, unofficial].
- $E_{smooth}(u_{t-1})$ is a smoothness regularizer (Tikhonov) weighted by $\alpha$ [S2, unofficial].
- $E_{kinetic}(u_{t-1})$ is a kinetic energy regularizer weighted by $\beta$ [S2, unofficial].
- Depth regularization is explicitly **not** applied to preserve the sparsity of the optimization matrix [S1][S2, unofficial].

**Primal-Dual Solver (Algorithm 5)**:
The problem is solved using a fast primal-dual method [S2, unofficial]. The updates are:
$$
X_{k+1} = X_k + \sigma Y_k - \sigma \text{prox}_{f, \sigma}\left(\frac{1}{\sigma} X_k + Y_k\right)
$$
$$
u_{k+1}^{t-1} = \text{prox}_{g, \frac{1}{\gamma}}\left(u_{k+1}^{t-1} - \gamma X_{k+1}\right)
$$
$$
Y_{k+1} = u_{k+1}^{t-1} + \theta \left(u_{k+1}^{t-1} - u_k^{t-1}\right)
$$
where:
- $k$ is the iteration index [S2, unofficial].
- $\sigma, \gamma, \theta$ are convergence parameters [S2, unofficial].
- $\text{prox}_{f, \sigma}(\xi) = \arg \min_{u_{t-1}} f(u_{t-1}) + \frac{\sigma}{2} \| u_{t-1} - \xi \|^2$ [S2, unofficial].
- $\text{prox}_{g, \frac{1}{\gamma}}$ handles the divergence-free constraint $\nabla \cdot u = 0$ via projection onto the space of divergence-free velocities $\Pi_{DIV}$ [S2, unofficial].

### Coupled Estimation (Algorithm 4)
1. **Velocity Estimation**: Compute $u_{t-1}$ using the Velocity Reconstruction algorithm above [S1][S2, unofficial].
2. **Velocity Prediction**: Advect $u_{t-1}$ with itself to create a velocity guess $\tilde{u}_t$, then project onto $\Pi_{DIV}$ to ensure incompressibility [S1][S2, unofficial].
3. **Density Prediction**: Advect $\rho_{t-1}$ forward using the predicted velocity $\tilde{u}_t$ (or $\tilde{u}_{t-1}$ as per some descriptions, but logically $\tilde{u}_t$ for step $t$) to create the density guess $\tilde{\rho}_t$ [S1][S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def coupled_density_velocity_estimation(rho_t_minus_1, rho_t_minus_2, dt, alpha, beta, sigma, gamma, theta, num_iters):
    """
    Reconstructs velocity and predicts density for time step t.
    
    Args:
        rho_t_minus_1: Density field at t-1 (3D array)
        rho_t_minus_2: Density field at t-2 (3D array)
        dt: Time step size
        alpha: Smoothness regularization weight
        beta: Kinetic energy regularization weight
        sigma, gamma, theta: Primal-dual solver parameters
        num_iters: Number of iterations for velocity reconstruction
        
    Returns:
        rho_t_pred: Predicted density field at t
        u_t_pred: Predicted velocity field at t
    """
    # Step 1: Velocity Reconstruction (Algorithm 5)
    # Initialize velocity estimate and dual variables
    u_est = zeros_like(rho_t_minus_1, shape=(3, *rho_t_minus_1.shape))
    X = zeros_like(u_est)
    Y = zeros_like(u_est)
    
    for k in range(num_iters):
        # Compute gradient of data fidelity term f w.r.t u
        # f(u) = || (rho_{t-1} - rho_{t-2})/dt + u . nabla rho_{t-1} ||^2 + alpha * E_smooth + beta * E_kinetic
        # The proximal operator for f involves solving a linear system or iterative step.
        # For simplicity, we assume a gradient-based proximal step or closed form if sparse.
        # Note: The exact implementation of prox_{f, sigma} depends on the specific form of E_smooth and E_kinetic.
        # Typically, this involves solving a sparse linear system due to the Tikhonov terms.
        
        # Update X
        # prox_{f, sigma}(xi) is the minimizer of f(u) + sigma/2 * ||u - xi||^2
        # This subproblem is solved efficiently because f is convex and smooth parts are quadratic.
        xi = (1/sigma) * X + Y
        u_prox_f = solve_prox_f(u_est, xi, sigma, rho_t_minus_1, rho_t_minus_2, dt, alpha, beta)
        X = X + sigma * Y - sigma * u_prox_f
        
        # Update u
        # prox_{g, 1/gamma} projects u onto the divergence-free space
        u_new = u_est - gamma * X
        u_new = project_divergence_free(u_new) # Projection onto Pi_DIV
        
        # Update Y (extrapolation)
        Y = u_new + theta * (u_new - u_est)
        
        # Update u_est for next iteration
        u_est = u_new
        
    u_t_minus_1 = u_est
    
    # Step 2: Velocity Prediction (Algorithm 4, lines 3-4)
    # Advect u_{t-1} with itself to get u_t_pred
    u_t_pred = advect_velocity(u_t_minus_1, u_t_minus_1)
    # Ensure incompressibility
    u_t_pred = project_divergence_free(u_t_pred)
    
    # Step 3: Density Prediction
    # Advect rho_{t-1} with u_t_pred to get rho_t_pred
    rho_t_pred = advect_density(rho_t_minus_1, u_t_pred)
    
    return rho_t_pred, u_t_pred

def solve_prox_f(u, xi, sigma, rho_t_minus_1, rho_t_minus_2, dt, alpha, beta):
    """
    Solves: min_u f(u) + sigma/2 * ||u - xi||^2
    f(u) = || (rho_{t-1} - rho_{t-2})/dt + u . nabla rho_{t-1} ||^2 + alpha * ||grad u||^2 + beta * ||u||^2
    This is a quadratic minimization problem, solvable via conjugate gradients or direct sparse solver.
    """
    # Construct the linear system A u = b
    # A = 2 * (Jacobian of optical flow term)^T * (Jacobian) + 2 * sigma * I + 2 * alpha * Laplacian + 2 * beta * I
    # b = 2 * (Jacobian of optical flow term)^T * (rho_{t-1} - rho_{t-2})/dt + sigma * xi
    # Note: The exact Jacobian and Laplacian operators depend on the discretization of nabla rho and grad u.
    # This is a sparse linear system.
    A, b = build_linear_system(u, xi, sigma, rho_t_minus_1, rho_t_minus_2, dt, alpha, beta)
    return solve_sparse_linear_system(A, b)

def project_divergence_free(u):
    """
    Projects velocity field u onto the space of divergence-free fields Pi_DIV.
    This is typically done by solving a Poisson equation for a pressure field p:
    Laplacian(p) = divergence(u)
    Then u_new = u - grad(p).
    """
    div_u = compute_divergence(u)
    p = solve_poisson_equation(div_u)
    grad_p = compute_gradient(p)
    return u - grad_p

def advect_velocity(u, u_source):
    """
    Advects velocity field u_source by itself using u.
    """
    # Use semi-Lagrangian advection or similar method
    return semi_lagrangian_advection(u_source, u)

def advect_density(rho, u):
    """
    Advects density field rho by velocity field u.
    """
    return semi_lagrangian_advection(rho, u)
```

### Parameters
- $\Delta t$: Time step size [S2, unofficial].
- $\alpha, \beta$: Weights for smoothness and kinetic energy regularization [S2, unofficial].
- $\sigma, \gamma, \theta$: Primal-dual solver parameters [S2, unofficial]. Typical values are not specified in the sources, but $\theta$ is often set to 1 for acceleration [S1].

### Edge Cases and Pitfalls
- **Depth Regularization**: Applying depth regularization breaks the sparsity of the optimization matrix, making the problem inefficient to solve. Therefore, it is excluded [S1][S2, unofficial].
- **Sparse Cameras**: The method is specifically designed to handle the ill-posedness caused by a reduced number of cameras [S1][S2, unofficial].
- **Incompressibility**: The projection onto $\Pi_{DIV}$ is critical for physical plausibility [S1][S2, unofficial].

## Operating characteristics
- **Time Complexity**: The method is highly efficient compared to prior coupled methods (e.g., Eckert et al. [4]), enabling interactive reconstruction [S1][S2, unofficial]. The velocity reconstruction uses a fast primal-dual solver, which is efficient for convex problems [S2, unofficial].
- **Memory Complexity**: Depends on the size of the 3D voxel grid and the sparsity of the linear systems solved in the proximal operators [S2, unofficial].
- **Strengths**: Handles sparse camera setups; enforces physical constraints (incompressibility, smoothness); fast enough for interactive use [S1][S2, unofficial].
- **Limitations**: Relies on accurate previous density estimates; assumes incompressible flow; does not use depth regularization which might limit accuracy in depth for some flows [S1][S2, unofficial].

## Background
The method was developed to address the limitations of earlier coupled density and velocity estimation techniques, such as those by Eckert et al. [4], which were too slow (50–60 minutes per frame) for practical use [S1][S2, unofficial]. The simplified strategy proposed in [S1][S2, unofficial] enables interactive reconstruction by removing depth regularization and using a fast primal-dual solver [S1][S2, unofficial]. It builds on the principles of differentiable rendering and tomographic reconstruction as inverse rendering [S2, unofficial].

## Sources
[S1] Physics-Based Differentiable Rendering for Efficient and Plausible Fluid Modeling from Monocular Video, https://doi.org/10.3390/e25091348, fetched 2026-10-26
[S2] Physics-Based Differentiable Rendering for Efficient and ... - MDPI, https://www.mdpi.com/1099-4300/25/9/1348, fetched 2026-10-27 (unofficial)
