# Motion-Compensated Algebraic Reconstruction Technique

Also called: Dynamic Algebraic Reconstruction Technique (DynART), Functional extended from ART for motion compensation, Motion-ART, Motion-Compensated ART, Motion-Compensated ART Functional

## Overview
Motion-Compensated Algebraic Reconstruction Technique (DynART) is an iterative algebraic reconstruction method designed to reconstruct deforming volumes from tomographic projection data acquired over time. It solves the problem of motion artifacts in dynamic tomography by jointly estimating the static reference volume and the time-dependent displacement field that describes the deformation. Unlike standard [Algebraic Reconstruction Technique (ART)](6944-algebraic-reconstruction-technique-art.md), which assumes a static object, DynART warps the current volume estimate according to a displacement field before projection and unwarps the correction term before updating the reference volume. This distinguishes it from siblings like [Simultaneous Algebraic Reconstruction Technique (SART)](10119-simultaneous-algebraic-reconstruction-technique-sart.md), which improves convergence speed but does not inherently handle motion, and [Direct Linear Inversion (ART)](8506-direct-linear-inversion-art.md), which solves the linear system directly without iterative correction.

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $p(r, t)$: Acquired projection data, where $r$ is the detector coordinate and $t \in \{1, \dots, N_t\}$ is the acquisition time index.
    *   $u(x, t)$: Displacement field mapping the reference configuration to the deformed configuration at time $t$. Initially $u(x, t) = 0$.
    *   $N_{DynART}$: Maximum number of iterations for the dynamic ART procedure.
    *   $\pi$: A permutation of the acquisition time indices $\{1, \dots, N_t\}$.
*   **Outputs:**
    *   $f(x)$: The reconstructed volume in the reference (undeformed) configuration.
    *   $u(x, t)$: The updated displacement field.

**Data Structures:**
*   $f(x)$: A 3D grid representing the voxel values of the reference volume.
*   $u(x, t)$: A 3D vector field grid, where each voxel stores a 3D displacement vector $(u_x, u_y, u_z)$.
*   $\tilde{f}(x, t)$: A temporary 3D grid for the warped volume at time $t$.
*   $\rho(r, t)$: A 1D array or grid for the projection residual.
*   $\Delta \tilde{f}(x)$: A 3D grid for the back-projected correction in the deformed frame.
*   $\Delta \hat{f}(x)$: A 3D grid for the unwarped correction in the reference frame.

**Mathematical Formulation:**
The method minimizes the functional [S1]:
$$ \Gamma_{\text{motion-ART}}[\psi, v] = \sum_{r,t} \| R_{\theta(t)}[\psi(x + v(x,t))] - p(r,t) \|^2 $$
where $\psi$ is the volume, $v$ is the displacement field, $R_{\theta(t)}$ is the Radon transform (projection) at angle $\theta(t)$, and $p(r,t)$ is the measured projection. The solution is:
$$ (f, u) = \text{Argmin}_{\psi,v} \Gamma_{\text{motion-ART}}[\psi(x), v(x,t)] $$

**Algorithm Steps:**
The procedure is a two-step alternating process:
1.  **Volume Reconstruction:** Update $f$ assuming $u$ is known, using the DynART loop.
2.  **Motion Update:** Update $u$ assuming $f$ is known, using Projection-based Digital Volume Correlation (P-DVC) [S1].

The DynART volume reconstruction loop is as follows:

1.  Initialize $n=1$, $f^{(1)}(x) = 0$.
2.  Choose a permutation $\pi$ over $N_t$ acquisition times.
3.  Loop for $i$ from $1$ to $N_{DynART}$:
    4.  Loop for $t$ from $1$ to $N_t$:
        5.  Let $\tau = \pi(t)$.
        6.  **Warp:** Compute the warped volume $\tilde{f}^{(n)}(x, \tau)$ by interpolating $f^{(n)}$ at the displaced coordinates:
            $$ \tilde{f}^{(n)}(x, \tau) = f^{(n)}(x + u(x, \tau)) $$
        7.  **Project:** Compute the simulated projection of the warped volume:
            $$ \hat{p}(r, \tau) = R_{\theta(\tau)}[\tilde{f}^{(n)}(x, \tau)] $$
        8.  **Residual:** Compute the projection residual:
            $$ \rho(r, \tau) = p(r, \tau) - \hat{p}(r, \tau) $$
        9.  **Back-project:** Compute the correction in the deformed frame:
            $$ \Delta \tilde{f}^{(n+1)}(x) = B_{\theta(\tau)}[\rho(r, \tau)] $$
            where $B_{\theta(\tau)}$ is the back-projection operator.
        10. **Unwarp:** Map the correction back to the reference frame. The correction is evaluated at the inverse displacement:
            $$ \Delta \hat{f}^{(n+1)}(x) = \Delta \tilde{f}^{(n+1)}(x - u(x, \tau)) $$
            *Note: Theoretically, the implicit definition $\Delta \hat{f}^{(n+1)}(x + u(x, \tau)) = \Delta \tilde{f}^{(n+1)}(x)$ is equivalent only for small strains. For large strains, Eulerian registration is required for convergence [S1].*
        11. **Update:** Add the unwarped correction to the volume:
            $$ f^{(n+1)}(x) = f^{(n)}(x) + \Delta \hat{f}^{(n+1)}(x) $$
        12. **Constraint:** Apply constraints (e.g., positivity) to $f^{(n+1)}(x)$ [S1].
    13. Increment $n$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def DynART(p, u, N_DynART, N_t, projection, backprojection, permutation):
    """
    p: array of shape (N_detectors, N_t) containing acquired projections.
    u: 3D array of shape (Nx, Ny, Nz, 3) containing displacement vectors for each time step.
       Note: In this implementation, u is assumed to be indexed by time step t.
       If u is a single field for all t, it is used directly.
       If u varies, the caller must handle the time-indexing of u.
       Here we assume u is passed as u[t][x,y,z] or similar structure accessible via u[t].
    N_DynART: int, max iterations.
    N_t: int, number of time steps.
    projection: function f(volume, angle) -> projection.
    backprojection: function f(projection, angle) -> volume.
    permutation: list of indices 0..N_t-1.
    """
    # Initialize volume to zero
    f = zeros((Nx, Ny, Nz))
    
    # Pre-calculate angles if not provided in p
    # Assume p is indexed by time step t
    
    for i in range(1, N_DynART + 1):
        for t_idx in permutation:
            # Get displacement field for time t_idx
            # u is assumed to be a list/array of 3D vector fields, one per time step
            displacement = u[t_idx] 
            
            # 1. Warp the volume
            # Create a grid of coordinates x, y, z
            # Compute warped coordinates: x_warped = x + u_x, etc.
            # Interpolate f at these warped coordinates to get f_tilde
            f_tilde = interpolate(f, displacement)
            
            # 2. Project the warped volume
            # Assume angle theta is determined by t_idx or passed separately
            theta = get_angle(t_idx)
            p_sim = projection(f_tilde, theta)
            
            # 3. Compute residual
            p_meas = p[:, t_idx]
            residual = p_meas - p_sim
            
            # 4. Back-project residual
            delta_f_tilde = backprojection(residual, theta)
            
            # 5. Unwarp the correction
            # Evaluate delta_f_tilde at x - u(x, t)
            # This requires interpolating delta_f_tilde at the inverse displacement
            delta_f_hat = interpolate_inverse(delta_f_tilde, displacement)
            
            # 6. Update volume
            f = f + delta_f_hat
            
            # 7. Apply constraints (e.g., positivity)
            f = max(f, 0)
            
    return f
```
*Note on Interpolation:* `interpolate(volume, displacement)` computes $V(x + u(x))$. `interpolate_inverse(volume, displacement)` computes $V(x - u(x))$. For large strains, the unwarping step must use Eulerian registration logic to ensure convergence [S1].

**Parameters:**
*   $N_{DynART}$: Maximum number of iterations. A convergence criterion based on the variation of the functional or a maximum number of iterations is used, as the functional value cannot be used directly due to imperfect reconstruction [S1].
*   $u(x, t)$: Displacement field. Initially $u(x, t) = 0$ [S1].
*   Constraints: Positivity is commonly applied [S1]. Other priors like Total Variation or dictionary learning can be added but are independent of the core DynART motion compensation [S1].

**Edge Cases and Pitfalls:**
*   **Large Strains:** The unwarping formula $\Delta \hat{f}(x) = \Delta \tilde{f}(x - u)$ is equivalent to the implicit definition only for small strains and rotations. For large strains (e.g., engineering strains up to 20%), Eulerian registration must be used instead of Lagrangian displacement to achieve convergence [S1]. Ignoring this difference slows convergence but may not affect the final solution if strains are not too large [S1].
*   **Convergence Criterion:** The standard ART convergence criterion based on the functional value cannot be used because the reconstruction is imperfect when motion is present. A criterion based on the variation of the functional or a fixed maximum number of iterations ($N_{DynART}$) must be set [S1].
*   **Multi-scale Approach:** To handle large displacement amplitudes and improve robustness, a multi-scale approach is coupled with the two-step procedure. This resolves large-scale features first and progressively enriches the description with finer details [S1].

## Subtopics

### [Adaptive Algebraic Reconstruction Technique (AART)](14387-adaptive-algebraic-reconstruction-technique-aart.md)
Adaptive Algebraic Reconstruction Technique (AART) is an optimization of the Algebraic Reconstruction Technique (ART) that employs adaptive updates to improve convergence and reconstruction quality. It addresses the limitations of standard ART by dynamically adjusting the correction magnitude based on the discrepancy between measured and calculated projections, rather than using a fixed step size. Unlike its sibling [Motion-Compensated Algebraic Reconstruction Technique](7422-motion-compensated-algebraic-reconstruction-technique.md), which handles deformation via warping, AART focuses on the algebraic update rule itself to enhance accuracy in tomographic reconstruction.

## Operating characteristics
*   **Time Complexity:** The method is iterative, with complexity proportional to $N_{DynART} \times N_t \times (\text{cost of warping} + \text{cost of projection} + \text{cost of backprojection} + \text{cost of unwarping})$. The warping and unwarping steps involve interpolation, which is typically $O(1)$ per voxel but requires accessing neighboring voxels.
*   **Memory Complexity:** Requires storage for the volume $f$, the displacement field $u$ for all time steps, and temporary buffers for warped volumes and residuals. Memory usage scales linearly with the volume size and the number of time steps $N_t$.
*   **Measured Performance:** The method is designed for dynamic tomography where the object deforms during scanning. It allows reconstruction from a single scan performed on a moving and deforming object [S1].
*   **Strengths:** Handles large displacements when combined with a multi-scale approach and Eulerian registration. Jointly estimates volume and motion, reducing artifacts caused by motion.
*   **Limitations:** Computationally expensive due to the iterative nature and the need for warping/unwarping at each step. Convergence can be slow for large strains if Eulerian registration is not used. The method assumes the displacement field is known or can be estimated via P-DVC in the alternating step.
*   **When to Use:** Use when reconstructing deforming volumes from dynamic tomographic data where motion artifacts are significant and standard static reconstruction methods fail.
*   **When Not to Use:** Use standard ART or SART if the object is static. Use if computational resources are limited and the motion is negligible.

## Background
The Motion-Compensated Algebraic Reconstruction Technique (DynART) was introduced in the paper "Dynamic Tomographic Reconstruction of Deforming Volumes" published in *Materials* in 2018 [S1]. It extends the standard Algebraic Reconstruction Technique (ART) to account for motion by warping the volume with a displacement field before projection and unwarping the correction term before updating the static reference volume. The method is part of a two-step algorithm that alternates between volume reconstruction using DynART and motion update using Projection-based Digital Volume Correlation (P-DVC). This approach allows for the identification of both complex, large displacement fields and volume texture from a single scan of a moving and deforming object. The authors note that the standard unwarping formula is only valid for small strains, and for larger strains, Eulerian registration is necessary for convergence. A multi-scale approach is also proposed to handle large displacement amplitudes and ensure fast convergence.

## Sources
[S1] Dynamic Tomographic Reconstruction of Deforming Volumes, https://doi.org/10.3390/ma11081395, fetched 2026-10-26
