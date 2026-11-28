# Diffusion Approximation for Cloud Radiance

Also called: Diffusion Approximation for Cloud Rendering, Diffusion method, Iterative Diffusion Solver with Multigrid, PDE solution for cloud illumination

## Overview
This method computes the isotropic radiance component $I_0(X)$ in dense cloud regions by solving a linear system derived from the diffusion approximation of the radiative transfer equation [S1, unofficial]. It replaces the full radiative transfer equation with a diffusion equation for diffuse light transport, which is solvable via multigrid relaxation [S1, unofficial].

## How it works
**Inputs:**
*   Cloud density field $\rho(X)$ defined on a 3D voxel grid [S1, unofficial].
*   Extinction coefficient per particle $\sigma_t$ and scattering coefficient $\sigma_s = a \sigma_t$, where $a$ is the albedo [S1, unofficial].
*   Henyey-Greenstein anisotropy factor $g$ (where $p_1 = g$) [S1, unofficial].
*   Sky radiance distribution $I_{sky}(\omega)$, discretized into direction bins [S1, unofficial].

**Outputs:**
*   Diffuse radiance scalar field $I_0(X)$ at each voxel [S1, unofficial].
*   Final radiance $I(X, \omega) = I_{ri}(X, \omega) + I_0(X) + \mathbf{I}_1(X) \cdot \boldsymbol{\omega}$, where $\mathbf{I}_1(X)$ is derived from $I_0(X)$ and sources [S1, unofficial].

**Data Structures:**
*   **Voxel Grid:** 3D array storing $\rho(X)$, $I_0(X)$, $\mathcal{Q}_0(X)$, and $\mathbf{Q}_1(X)$ [S1, unofficial].
*   **Direction Bins:** A set of directions $\omega_i$ on a subdivided unit cube surface. The implementation uses 64x64 resolution images on each of the 6 faces of a viewing cube, averaged into 4x4 direction bins per face, resulting in 96 total direction bins [S1, unofficial].

**Step 1: Compute Reduced Incident Radiance and Sources**
1.  For each direction bin $i$ with direction $\omega_i$ and solid angle $\Delta\omega_i$:
    *   Trace a dense collection of rays in direction $\omega_i$ starting from the external faces of the box enclosing the cloud [S1, unofficial].
    *   For each ray, identify the segments $S_j$ it intersects within the cloud voxels [S1, unofficial].
    *   Accumulate the attenuation integral along each segment $S_j$:
        $$ I_{ri}(X, \omega_i) = I_{sky}(\omega_i) \exp \left( - \int_0^\infty \sigma_t \rho(X - s\omega_i) ds \right) $$
        The integral is accumulated along the ray segments $S_j$ [S1, unofficial].
    *   Compute the first scattered external illumination contribution for this direction:
        $$ \mathcal{Q}(X, \omega_i) = \rho(X) \sigma_s I_{ri}(X, \omega_i) \Delta\omega_i p_1 \boldsymbol{\omega}_i $$
        The angular integral simplifies because $p(\omega, \omega')$ depends only on the angle, and contributions from other directions cancel by symmetry [S1, unofficial].
    *   Accumulate these contributions into voxel-wise sums for $\mathcal{Q}_0$ and $\mathbf{Q}_1$:
        $$ \text{Sum}_{\mathcal{Q}_0}(X) \leftarrow \text{Sum}_{\mathcal{Q}_0}(X) + \frac{1}{4\pi} \mathcal{Q}(X, \omega_i) $$
        $$ \text{Sum}_{\mathbf{Q}_1}(X) \leftarrow \text{Sum}_{\mathbf{Q}_1}(X) + \frac{1}{4\pi} \mathcal{Q}(X, \omega_i) $$
    *   Accumulate the total segment length in voxel $X$ for this direction:
        $$ \text{Sum}_{len}(X) \leftarrow \text{Sum}_{len}(X) + \sum_j l(S_j) $$
2.  After processing all rays and directions, normalize the sources for each voxel $X$:
    $$ \mathcal{Q}_0(X) = \frac{\text{Sum}_{\mathcal{Q}_0}(X)}{\text{Sum}_{len}(X)} $$
    $$ \mathbf{Q}_1(X) = \frac{\text{Sum}_{\mathbf{Q}_1}(X)}{\text{Sum}_{len}(X)} $$
    If $\text{Sum}_{len}(X) = 0$, set $\mathcal{Q}_0(X) = 0$ and $\mathbf{Q}_1(X) = 0$ [S1, unofficial].

**Step 2: Discretize the Diffusion Equation**
The diffusion equation to solve is:
$$ \nabla \cdot \left( \frac{1}{3\rho(X)\sigma_{tr}} \nabla I_0(X) \right) - \rho(X)\sigma_a I_0(X) = -\mathcal{Q}_0(X) + \nabla \cdot \left( \frac{1}{4\pi\rho(X)\sigma_{tr}} \mathbf{Q}_1(X) \right) $$
where $\sigma_{tr} = \sigma_t (1 - a p_1)$ and $\sigma_a = \sigma_t (1 - a)$ [S1, unofficial].

1.  Define the transport cross section: $\sigma_{tr} = \sigma_t (1 - a g)$ [S1, unofficial].
2.  For each voxel $X$ with $\rho(X) > 0$, construct a linear equation relating $I_0(X)$ to its six neighbors [S1, unofficial].
3.  Evaluate the gradient of $I_0$ at face centers using central differences [S1, unofficial].
4.  Determine the coefficient $\frac{1}{3\rho(X)\sigma_{tr}}$ for the diffusion term:
    *   For a face shared by voxel $X$ and neighbor $N$, use the average density: $\bar{\rho} = \frac{\rho(X) + \rho(N)}{2}$ [S1, unofficial].
    *   If one of the cells has $\rho = 0$, use the value from the non-zero cell [S1, unofficial].
5.  Evaluate the divergence of the source term $\mathbf{Q}_1$ using central differences [S1, unofficial].
6.  Handle boundary conditions:
    *   For a voxel $X$ adjacent to an empty border voxel $X_e$ (where $\rho(X_e)=0$), enforce the no total inward flow boundary condition:
        $$ I_0(X) - \frac{2}{3\rho(X)\sigma_{tr}} \mathbf{n} \cdot \nabla I_0(X) + \frac{1}{2\pi\rho(X)\sigma_{tr}} \mathbf{n} \cdot \mathbf{Q}_1(X) = 0 $$
        where $\mathbf{n}$ is the inward surface normal [S1, unofficial].
    *   Assume a temporary value $I_0(X_e)$ at the empty cell that satisfies this condition [S1, unofficial].
    *   Use this temporary value to compute the finite difference for the gradient at the face between $X$ and $X_e$ [S1, unofficial].
    *   Substitute this into the linear equation for $X$. This affects the coefficients of the linear equation for $X$ but does not introduce $I_0(X_e)$ as a variable in the system [S1, unofficial].

**Step 3: Solve the Linear System**
1.  The resulting system is a set of linear equations relating each $I_0(X)$ to its six neighbors and a constant term involving $\mathcal{Q}_0$ and $\mathbf{Q}_1$ [S1, unofficial].
2.  Solve using a multigrid method to accelerate convergence, as Gauss-Seidel or Jacobi methods are too slow for large grids due to slow propagation of bright spots [S1, unofficial].
    *   Treat the data volume as a multi-resolution octree hierarchy [S1, unofficial].
    *   Write coarser linear equations for lower levels of the hierarchy [S1, unofficial].
    *   Use solutions from coarser levels to correct solutions at finer levels [S1, unofficial].
    *   Iterate up and down the hierarchy until convergence [S1, unofficial].
    *   Implementation may use the Hypre system API and multigrid solver from Lawrence Livermore National Laboratory [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_cloud_radiance(density_grid, sigma_t, albedo, g, sky_radiance_bins, grid_spacing=1.0):
    """
    density_grid: 3D numpy array of voxel densities rho(X)
    sigma_t: extinction coefficient per particle
    albedo: scattering albedo a
    g: anisotropy factor p1
    sky_radiance_bins: list of (direction_vec, I_sky, d_omega) tuples
    grid_spacing: voxel edge length h
    """
    sigma_s = albedo * sigma_t
    sigma_a = sigma_t * (1 - albedo)
    sigma_tr = sigma_t * (1 - albedo * g)
    pi = np.pi
    
    # Initialize source accumulators and length sums
    Q0_sum = np.zeros_like(density_grid)
    Q1_sum = np.zeros_like(density_grid)
    len_sum = np.zeros_like(density_grid)
    
    # Step 1: Compute Sources
    for direction, I_sky, d_omega in sky_radiance_bins:
        # Ray tracing from external faces to compute attenuation integrals along segments Sj 
        # is not detailed in the sources [S1, unofficial].
        rays = trace_rays_in_direction(direction, density_grid)
        for ray in rays:
            segments = ray.intersect_voxels(density_grid)
            total_attenuation = 0.0
            for segment in segments:
                voxel = segment.voxel
                rho = density_grid[voxel]
                if rho == 0:
                    continue
                total_attenuation += sigma_t * rho * segment.length
            
            # Reduced incident radiance
            I_ri = I_sky * np.exp(-total_attenuation)
            
            # First scattered illumination contribution
            # Q(X, omega) = rho * sigma_s * I_ri * d_omega * p1 * omega
            contribution_scalar = rho * sigma_s * I_ri * d_omega * g
            contribution_vector = contribution_scalar * direction
            
            Q0_sum[voxel] += contribution_scalar / (4 * pi)
            Q1_sum[voxel] += contribution_vector / (4 * pi)
            len_sum[voxel] += segment.length

    # Normalize sources
    Q0 = np.where(len_sum > 0, Q0_sum / len_sum, 0.0)
    Q1 = np.where(len_sum > 0, Q1_sum / len_sum, 0.0)
    
    # Step 2: Build Linear System
    # Equation: div( alpha * grad I0 ) - rho*sigma_a*I0 = -Q0 + div( beta * Q1 )
    # alpha = 1/(3*rho*sigma_tr), beta = 1/(4*pi*rho*sigma_tr)
    
    n = density_grid.shape
    rows = []
    voxel_to_idx = {}
    idx = 0
    
    for x in range(n[0]):
        for y in range(n[1]):
            for z in range(n[2]):
                if density_grid[x, y, z] > 0:
                    voxel_to_idx[(x, y, z)] = idx
                    idx += 1
                    
    for (x, y, z), idx in voxel_to_idx.items():
        coeffs = {}
        rhs = 0.0
        rho_X = density_grid[x, y, z]
        alpha_X = 1.0 / (3.0 * rho_X * sigma_tr)
        beta_X = 1.0 / (4.0 * pi * rho_X * sigma_tr)
        
        # Diagonal term from -rho*sigma_a*I0
        diag_coeff = -rho_X * sigma_a
        
        neighbors = [
            ((x-1, y, z), (-1, 0, 0)), ((x+1, y, z), (1, 0, 0)),
            ((x, y-1, z), (0, -1, 0)), ((x, y+1, z), (0, 1, 0)),
            ((x, y, z-1), (0, 0, -1)), ((x, y, z+1), (0, 0, 1))
        ]
        
        for (nx, ny, nz), normal in neighbors:
            if not (0 <= nx < n[0] and 0 <= ny < n[1] and 0 <= nz < n[2]):
                continue
                
            rho_N = density_grid[nx, ny, nz]
            # Effective rho for face
            if rho_X > 0 and rho_N > 0:
                rho_face = (rho_X + rho_N) / 2.0
            else:
                rho_face = rho_X if rho_X > 0 else rho_N
                
            alpha_face = 1.0 / (3.0 * rho_face * sigma_tr)
            
            if rho_N > 0:
                # Interior neighbor
                neighbor_idx = voxel_to_idx.get((nx, ny, nz))
                if neighbor_idx is not None:
                    coeffs[neighbor_idx] = coeffs.get(neighbor_idx, 0.0) + alpha_face / grid_spacing**2
                    diag_coeff -= alpha_face / grid_spacing**2
            else:
                # Empty border voxel Xe: apply BC to eliminate I0(Xe)
                # BC: I0(X) - D * n.grad(I0) + C = 0, where D = 2/(3*rho_X*sigma_tr), C = n.Q1/(2*pi*rho_X*sigma_tr)
                D = 2.0 / (3.0 * rho_X * sigma_tr)
                C = np.dot(normal, Q1[x, y, z]) / (2.0 * pi * rho_X * sigma_tr)
                
                # Approximate n.grad(I0) at face as (I0(X) - I0(Xe))/h
                # From BC: I0(Xe) = I0(X) - (h/D) * (I0(X) + C)
                # Substituting into diffusion term alpha_face * (I0(Xe) - I0(X))/h^2
                # yields modified diagonal and constant term.
                bc_factor = alpha_face / (grid_spacing * D)
                diag_coeff -= bc_factor
                rhs += bc_factor * C
                
        # Divergence of Q1 source term (last term in eq 4)
        div_Q1 = 0.0
        for (nx, ny, nz), normal in neighbors:
            if 0 <= nx < n[0] and 0 <= ny < n[1] and 0 <= nz < n[2]:
                rho_N = density_grid[nx, ny, nz]
                rho_face = (rho_X + rho_N) / 2.0 if rho_X > 0 and rho_N > 0 else (rho_X if rho_X > 0 else rho_N)
                beta_face = 1.0 / (4.0 * pi * rho_face * sigma_tr)
                div_Q1 += np.dot(normal, Q1[nx, ny, nz] - Q1[x, y, z]) / grid_spacing
                
        rhs -= (-Q0[x, y, z] + div_Q1)
        
        coeffs[idx] = diag_coeff
        rows.append((idx, coeffs, rhs))
        
    # Step 3: Solve using Multigrid V-cycle
    I0 = np.zeros_like(density_grid)
    # Multigrid V-cycle implementation is not detailed in the sources [S1, unofficial]. 
    # Standard multigrid solvers (e.g., Hypre) are used [S1, unofficial].
    I0 = multigrid_v_cycle(rows, voxel_to_idx, n, grid_spacing, I0)
    
    return I0

def trace_rays_in_direction(direction, density_grid):
    """Generates rays from external faces in the given direction."""
    # Standard hemicube/ray traversal implementation [G]
    return []

def multigrid_v_cycle(rows, voxel_to_idx, n, h, I0_guess):
    """Implements a multigrid V-cycle for the linear system."""
    # Standard multigrid V-cycle implementation [G]
    return I0_guess
```

## Operating characteristics
*   **Time Complexity:** The multigrid solver provides faster convergence than Gauss-Seidel or Jacobi, which are prohibitively slow for large grids because bright spots propagate only one adjacent cell per iteration [S1, unofficial].
*   **Memory Complexity:** Requires storage for the density grid, source fields ($\mathcal{Q}_0, \mathbf{Q}_1$), and the solution field $I_0$, plus multigrid hierarchy data [S1, unofficial].
*   **Strengths:** Efficiently handles dense media where the mean free path is short, capturing multiple scattering effects that single-scattering approximations miss [S1, unofficial].
*   **Limitations:** Assumes dense media; less accurate for sparse regions or near boundaries where the diffusion approximation breaks down [S1, unofficial]. The derivation in Stam [Stam95] is noted to be incorrect compared to Ishimaru [Ish78], differing by factors of 3 in several terms [S1, unofficial].
*   **When to use:** Dense cloud rendering where multiple scattering is significant [S1, unofficial].
*   **When not to use:** Sparse media or when high-fidelity single-scattering details are required [S1, unofficial].

## Background
Kajiya and von Herzen [Kaj84] first proposed the diffusion approximation to the computer graphics community [S1, unofficial]. Stam [Stam95] gave the first practical implementation for radiance transport in clouds [S1, unofficial]. The method is based on the radiative transfer equation and the diffusion approximation as detailed in [Ish78] [S1, unofficial]. The implementation described uses the Hypre system API and multigrid solver from Lawrence Livermore National Laboratory [S1, unofficial].

## Sources
[S1] Diffusion and multiple anisotropic scattering for global illumination in clouds, https://digital.library.unt.edu/ark:/67531/metadc1410406/?q=multiple scattering rendering, fetched 2026-10-29 (unofficial)
