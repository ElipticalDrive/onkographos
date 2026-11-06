# Diffusion Approximation

Also called: Classical Diffusion Approximation, Diffusion Approximation of RTE, Diffusion Theory, Radiative Diffusion Equation

## Overview
The Diffusion Approximation (DA) is a method for approximating multiple scattering in participating media by replacing the computationally expensive integral form of the Radiative Transport Equation (RTE) with a partial differential equation (PDE) [S2]. It expresses the medium radiance as the first two angular moments (fluence and flux) of the radiation field, solving for the fluence field to determine in-scattering [S1]. It is distinct from its siblings because it operates in the spatial domain using a PDE solver rather than angular expansions like the [P-N Method](1395-p-n-method-spherical-harmonics.md) or stochastic sampling like the [Monte Carlo Method](1940-monte-carlo-method-for-multiple-scattering.md). It is most effective in dense, highly opaque media where photons undergo long random walks, making path tracing intractable [S1].

## How it works
**Inputs:**
*   Volume domain $\Omega$ with spatial coordinates $\mathbf{x}$.
*   Optical coefficients: absorption $\sigma_a(\mathbf{x})$, scattering $\sigma_s(\mathbf{x})$, extinction $\sigma_t(\mathbf{x}) = \sigma_a(\mathbf{x}) + \sigma_s(\mathbf{x})$.
*   Source fields: reduced incident light source $q_{ri}(\mathbf{x})$ (from unscattered external lighting) and self-emission $j(\mathbf{x})$.
*   Boundary conditions for fluence $\phi$ on $\partial \Omega$.

**Outputs:**
*   Fluence field $\phi(\mathbf{x})$ (scalar irradiance).
*   Flux field $\mathbf{E}(\mathbf{x})$ (vector irradiance).
*   Approximated medium radiance $L_m(\mathbf{x}, \boldsymbol{\omega})$.

**Data Structures:**
*   **Grid:** A regular 3D grid (typically cube-like boundaries) storing scalar and vector fields [S2].
*   **Fluence Field:** 3D array of floats, $\phi[i,j,k]$.
*   **Flux Field:** 3D array of 3-component vectors, $\mathbf{E}[i,j,k]$.
*   **Diffusion Coefficient:** 3D array of floats, $D[i,j,k]$.

**Assumptions:**
*   Isotropic scattering is assumed in the standard derivation [S1].
*   The method is limited to volume scenes without explicit surface models [S2].
*   Data types are restricted to regular volume data [S2].

**Steps:**

1.  **Compute Diffusion Coefficient:**
    Calculate the diffusion coefficient $D(\mathbf{x})$ at each grid point.
    $$ D(\mathbf{x}) = \frac{1}{3\sigma_t(\mathbf{x})} $$
    [S1]

2.  **Define Source Term:**
    Combine the reduced incident light source $q_{ri}(\mathbf{x})$ and self-emission $j(\mathbf{x})$ into the total source term for the diffusion equation.
    $$ S(\mathbf{x}) = q_{ri}(\mathbf{x}) + j(\mathbf{x}) $$
    [S1]

3.  **Solve the Radiative Diffusion Equation:**
    Solve the following partial differential equation for the fluence $\phi(\mathbf{x})$ over the volume domain $\Omega$:
    $$ \nabla \cdot (D(\mathbf{x})\nabla \phi(\mathbf{x})) = \sigma_a(\mathbf{x})\phi(\mathbf{x}) - S(\mathbf{x}) $$
    [S1]
    
    *Boundary Conditions:* Use Dirichlet boundary conditions where $\phi(\mathbf{x}) = 0$ on the boundary $\partial \Omega$ [S1]. Note: This is physically an approximation, as it assumes no diffuse light escapes, but it is standard for this method [S1].

4.  **Compute Flux (Fick's First Law):**
    Once $\phi(\mathbf{x})$ is known, compute the flux $\mathbf{E}(\mathbf{x})$ using the gradient of the fluence.
    $$ \mathbf{E}(\mathbf{x}) = -D(\mathbf{x})\nabla \phi(\mathbf{x}) $$
    [S1]

5.  **Reconstruct Radiance:**
    Approximate the medium radiance $L_m(\mathbf{x}, \boldsymbol{\omega})$ for any direction $\boldsymbol{\omega}$ using the first two angular moments.
    $$ L_m(\mathbf{x}, \boldsymbol{\omega}) \approx \frac{1}{4\pi} \phi(\mathbf{x}) + \frac{3}{4\pi} \boldsymbol{\omega} \cdot \mathbf{E}(\mathbf{x}) $$
    [S1]

6.  **Final Rendering (Ray Marching):**
    For a camera ray, compute the total radiance at the endpoint $\mathbf{x}_e$ by combining the attenuated incident light and the accumulated in-scattering from the diffusion approximation.
    $$ L(\mathbf{x}_e, \boldsymbol{\omega}) = L(\mathbf{x}_s, \boldsymbol{\omega}) T(\mathbf{x}_s, \mathbf{x}_e) + \Delta x \sum_n T(\mathbf{x}_n, \mathbf{x}_e) Q(\mathbf{x}_n) $$
    [S1]
    Where $T$ is transmittance, $\mathbf{x}_s$ is the start point, and $Q(\mathbf{x}_n)$ is the source term derived from the diffusion solution (specifically the in-scattering component). Note: In practice, the diffusion solution provides the *multiply-scattered* component $L_m$, which is integrated along the ray.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def solve_diffusion_approximation(sigma_a, sigma_s, q_ri, j, grid_spacing):
    """
    Solves the Classical Diffusion Approximation for a regular 3D volume.
    
    Args:
        sigma_a: 3D numpy array, absorption coefficient.
        sigma_s: 3D numpy array, scattering coefficient.
        q_ri: 3D numpy array, reduced incident light source.
        j: 3D numpy array, self-emission source.
        grid_spacing: float, spacing between grid cells (dx, dy, dz assumed equal).
        
    Returns:
        phi: 3D numpy array, fluence field.
        E: 3D numpy array, flux field (shape same as sigma_a, with 3 channels).
        L_m_func: A function that returns radiance for a given direction omega.
    """
    # 1. Compute extinction and diffusion coefficient
    sigma_t = sigma_a + sigma_s
    # Guard against division by zero in vacuum regions (pitfall)
    # [G] In CDA, this is undefined; FLD handles it. Here we clamp or handle as error.
    # For implementation safety, we assume valid opaque media per source constraints.
    D = 1.0 / (3.0 * sigma_t)
    
    # 2. Define Source Term
    S = q_ri + j
    
    # 3. Solve PDE: div(D * grad(phi)) = sigma_a * phi - S
    # This is a linear system A * phi = b.
    # We use a finite difference discretization on a regular grid.
    # Laplacian operator with variable coefficient D.
    
    N = sigma_a.shape
    # Flatten grid for linear solver
    num_cells = N[0] * N[1] * N[2]
    
    # Construct sparse matrix A and vector b
    # [G] Standard 7-point stencil for div(D grad phi) in 3D.
    # For cell (i,j,k), neighbors are (i+/-1,j,k), (i,j+/-1,k), (i,j,k+/-1).
    # Coefficient for neighbor (i+1,j,k): D_avg * Area / dx^2
    # D_avg between i and i+1 is (D[i,j,k] + D[i+1,j,k]) / 2
    
    import scipy.sparse as sp
    
    rows = []
    cols = []
    vals = []
    b = np.zeros(num_cells)
    
    dx = grid_spacing
    
    for i in range(N[0]):
        for j in range(N[1]):
            for k in range(N[2]):
                idx = i * N[1] * N[2] + j * N[2] + k
                
                # Diagonal term: -div(D grad phi) + sigma_a phi = S
                # The Laplacian part contributes negative values to diagonal if moved to RHS?
                # Equation: div(D grad phi) - sigma_a phi = -S
                # Discretization of div(D grad phi) at (i,j,k):
                # 1/dx^2 * [ D_x+ * (phi_i+1 - phi_i) - D_x- * (phi_i - phi_i-1) + ... ]
                # = 1/dx^2 * [ - (D_x+ + D_x-) phi_i + D_x+ phi_i+1 + D_x- phi_i-1 + ... ]
                
                # Calculate average D for neighbors
                # Right (i+1)
                if i < N[0] - 1:
                    D_right = (D[i,j,k] + D[i+1,j,k]) / 2.0
                    idx_right = (i+1) * N[1] * N[2] + j * N[2] + k
                    rows.append(idx); cols.append(idx_right); vals.append(D_right / dx**2)
                else:
                    # Boundary condition: phi = 0. 
                    # This term moves to RHS as 0, but effectively removes the neighbor.
                    pass
                
                # Left (i-1)
                if i > 0:
                    D_left = (D[i,j,k] + D[i-1,j,k]) / 2.0
                    idx_left = (i-1) * N[1] * N[2] + j * N[2] + k
                    rows.append(idx); cols.append(idx_left); vals.append(D_left / dx**2)
                else:
                    pass

                # Front (j+1)
                if j < N[1] - 1:
                    D_front = (D[i,j,k] + D[i,j+1,k]) / 2.0
                    idx_front = i * N[1] * N[2] + (j+1) * N[2] + k
                    rows.append(idx); cols.append(idx_front); vals.append(D_front / dx**2)
                else:
                    pass

                # Back (j-1)
                if j > 0:
                    D_back = (D[i,j,k] + D[i,j-1,k]) / 2.0
                    idx_back = i * N[1] * N[2] + (j-1) * N[2] + k
                    rows.append(idx); cols.append(idx_back); vals.append(D_back / dx**2)
                else:
                    pass

                # Top (k+1)
                if k < N[2] - 1:
                    D_top = (D[i,j,k] + D[i,j,k+1]) / 2.0
                    idx_top = i * N[1] * N[2] + j * N[2] + (k+1)
                    rows.append(idx); cols.append(idx_top); vals.append(D_top / dx**2)
                else:
                    pass

                # Bottom (k-1)
                if k > 0:
                    D_bottom = (D[i,j,k] + D[i,j,k-1]) / 2.0
                    idx_bottom = i * N[1] * N[2] + j * N[2] + (k-1)
                    rows.append(idx); cols.append(idx_bottom); vals.append(D_bottom / dx**2)
                else:
                    pass

                # Diagonal
                diag_val = -(vals[-6] if len(vals)>=6 else 0) # Sum of off-diagonals is not quite right here, recalculate sum
                # Recalculate sum of neighbors for this cell
                sum_neighbors = 0
                if i < N[0]-1: sum_neighbors += (D[i,j,k] + D[i+1,j,k])/2.0
                if i > 0: sum_neighbors += (D[i,j,k] + D[i-1,j,k])/2.0
                if j < N[1]-1: sum_neighbors += (D[i,j,k] + D[i,j+1,k])/2.0
                if j > 0: sum_neighbors += (D[i,j,k] + D[i,j-1,k])/2.0
                if k < N[2]-1: sum_neighbors += (D[i,j,k] + D[i,j,k+1])/2.0
                if k > 0: sum_neighbors += (D[i,j,k] + D[i,j,k-1])/2.0
                
                # Equation: div(D grad phi) - sigma_a phi = -S
                # Discrete: (1/dx^2) * (-sum_neighbors * phi_i + sum(neighbor_terms)) - sigma_a_i * phi_i = -S_i
                # Rearranged for A*phi=b:
                # (sum_neighbors/dx^2 + sigma_a_i) * phi_i - (1/dx^2)*sum(neighbor_terms) = S_i
                
                A_diag = sum_neighbors / dx**2 + sigma_a[i,j,k]
                rows.append(idx); cols.append(idx); vals.append(A_diag)
                
                b[idx] = S[i,j,k]

    # Create sparse matrix
    A = sp.csr_matrix((vals, (rows, cols)), shape=(num_cells, num_cells))
    
    # Solve linear system
    phi_flat = np.linalg.solve(A.toarray(), b) # Use sparse solver in practice
    phi = phi_flat.reshape(N)
    
    # 4. Compute Flux E = -D grad phi
    # Central differences for gradient
    E = np.zeros((*N, 3))
    
    # Gradient in x
    dphi_dx = np.zeros(N)
    dphi_dx[1:-1, :, :] = (phi[2:, :, :] - phi[:-2, :, :]) / (2 * dx)
    # Boundary gradients are 0 due to phi=0 boundary? Or extrapolate. 
    # [G] Simple central diff leaves boundaries undefined. Set to 0 or use one-sided.
    dphi_dx[0, :, :] = (phi[1, :, :] - phi[0, :, :]) / dx
    dphi_dx[-1, :, :] = (phi[-1, :, :] - phi[-2, :, :]) / dx
    
    # Gradient in y
    dphi_dy = np.zeros(N)
    dphi_dy[:, 1:-1, :] = (phi[:, 2:, :] - phi[:, :-2, :]) / (2 * dx)
    dphi_dy[:, 0, :] = (phi[:, 1, :] - phi[:, 0, :]) / dx
    dphi_dy[:, -1, :] = (phi[:, -1, :] - phi[:, -2, :]) / dx
    
    # Gradient in z
    dphi_dz = np.zeros(N)
    dphi_dz[:, :, 1:-1] = (phi[:, :, 2:] - phi[:, :, :-2]) / (2 * dx)
    dphi_dz[:, :, 0] = (phi[:, :, 1] - phi[:, :, 0]) / dx
    dphi_dz[:, :, -1] = (phi[:, :, -1] - phi[:, :, -2]) / dx
    
    E[:,:,0] = -D * dphi_dx
    E[:,:,1] = -D * dphi_dy
    E[:,:,2] = -D * dphi_dz
    
    # 5. Define Radiance Reconstruction Function
    def get_radiance(x, y, z, omega):
        # Interpolate phi and E at (x,y,z)
        # [G] Bilinear/Trilinear interpolation assumed for continuous coordinates
        phi_val = interpolate(phi, x, y, z)
        E_val = interpolate(E, x, y, z)
        
        # L_m = 1/(4pi) * phi + 3/(4pi) * omega . E
        L_m = (1.0 / (4.0 * np.pi)) * phi_val + (3.0 / (4.0 * np.pi)) * np.dot(omega, E_val)
        return L_m
        
    return phi, E, get_radiance

def interpolate(field, x, y, z):
    # Placeholder for trilinear interpolation logic
    # [G] Standard trilinear interpolation on grid coordinates
    pass
```

## Subtopics

### [Flux-Limited Diffusion](3239-flux-limited-diffusion.md)
A variant that modifies the diffusion coefficient to handle heterogeneous media, vacuum regions, and transitions to the transport regime, preventing the divergence of $D$ when $\sigma_t \to 0$. Choose this when the scene contains transparent regions or vacuum.

## Operating characteristics
*   **Time Complexity:** Solving the PDE is generally $O(N)$ to $O(N \log N)$ depending on the linear solver (e.g., conjugate gradient), where $N$ is the number of grid cells. This is significantly faster than path tracing for high-opacity media, which scales poorly with optical depth.
*   **Memory Complexity:** $O(N)$ to store the fluence, flux, and coefficient fields.
*   **Strengths:** Extremely efficient for dense, highly scattering media (e.g., fog, clouds, skin) where multiple scattering dominates. Avoids the noise and long computation times of Monte Carlo methods in these regimes.
*   **Limitations:** Breaks down in regions of low extinction (high mean free path) or vacuum, where the diffusion coefficient $D(\mathbf{x})$ diverges as $\sigma_t \to 0$ [S1]. It also assumes isotropic scattering in its basic form [S1]. It cannot easily handle sharp shadows or single-scattering effects accurately.
*   **When to use:** High-opacity, homogeneous or smoothly varying media where multiple scattering is the dominant light transport mechanism.
*   **When not to use:** Scenes with vacuum, low-opacity regions, or where single scattering/shadows are visually critical.

## Background
The Diffusion Approximation originates from radiative transfer theory in physics and astrophysics, where it is used to model light transport in dense media [S1]. In computer graphics, it was adapted to accelerate the simulation of multiple scattering in participating media, particularly for effects like subsurface scattering and volumetric fog [S1]. The "Classical Diffusion Approximation" (CDA) refers to the standard form using $D = 1/(3\sigma_t)$, which was found to be problematic for general volumetric media with varying opacity, leading to the development of variants like Flux-Limited Diffusion [S1].

## Sources
[S1] Flux-Limited Diffusion for Multiple Scattering in Participating Media, https://archive.org/details/arxiv-1403.8105, fetched 2026-10-26
[S2] A Differential Diffusion Theory for Participating Media, https://doi.org/10.1111/cgf.14956, fetched 2026-10-30
