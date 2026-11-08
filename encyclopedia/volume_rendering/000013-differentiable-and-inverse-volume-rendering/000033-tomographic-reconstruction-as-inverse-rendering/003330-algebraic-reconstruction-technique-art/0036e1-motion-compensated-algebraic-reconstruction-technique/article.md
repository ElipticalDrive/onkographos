# Motion-Compensated Algebraic Reconstruction Technique

Also called: Dynamic Algebraic Reconstruction Technique (DynART), Functional extended from ART for motion compensation, Motion-ART, Motion-Compensated ART, Motion-Compensated ART Functional

## Overview
Motion-Compensated Algebraic Reconstruction Technique (DynART) is an iterative algebraic reconstruction method designed to reconstruct deforming volumes from tomographic projections acquired during motion [S1]. It extends the standard Algebraic Reconstruction Technique (ART) by jointly minimizing the discrepancy between acquired projections and the projections of a volume warped by a displacement field [S1]. Unlike standard ART, which assumes a static object, DynART warps the reconstructed volume according to a motion field before projection and unwarp the correction term before updating the static reference volume, allowing for the reconstruction of high-quality volumes from single scans of moving or deforming objects [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Acquired projections $p(r, t)$ for detector coordinates $r$ and acquisition times $t$; a permutation $\pi$ over the $N_t$ acquisition times; the maximum number of iterations $N_{DynART}$; an initial displacement field $u(x, t)$ (typically initialized to zero); and a reference volume $f$ [S1].
*   **Outputs:** Reconstructed volume $f(x)$ in the reference configuration and the displacement field $u(x, t)$ [S1].

**Data Structures:**
*   **Volume $f(x)$:** A 3D grid representing the density in the reference (undeformed) configuration.
*   **Displacement Field $u(x, t)$:** A 3D vector field defined at each voxel $x$ for each time $t$, representing the Lagrangian displacement from the reference configuration to the deformed state at time $t$ [S1].
*   **Projections $p(r, t)$:** 2D arrays of measured projection data [S1].

**Mathematical Formulation:**
The method minimizes the functional:
$$ \Gamma_{motion-ART}[\psi, v] = \sum_{r,t} \| R_{\theta(t)}[\psi(x + v(x,t))] - p(r,t) \|^2 $$
where $\psi$ is the volume, $v$ is the displacement field, $R_{\theta(t)}$ is the Radon transform at angle $\theta(t)$, and $p(r,t)$ is the projection [S1]. The solution is:
$$ (f, u) = \text{Argmin}_{\psi,v} \Gamma_{motion-ART}[\psi(x), v(x,t)] $$
[S1].

**Algorithm Steps:**
The procedure is a two-step alternating process:
1.  **Volume Reconstruction:** DynART updates the volume assuming a known motion field [S1].
2.  **Motion Update:** The displacement field is updated using Projection-based Digital Volume Correlation (P-DVC) given the current volume estimate [S1].

The DynART volume update loop (Algorithm 2 in [S1]) is as follows:

1.  Initialize iteration counter $n = 1$.
2.  Initialize volume $f^{(1)}(x) = 0$.
3.  Initialize displacement field $u(x, t) = 0$ for all $x, t$ [S1].
4.  Choose a permutation $\pi$ over the $N_t$ acquisition times [S1].
5.  Loop for $i$ from $1$ to $N_{DynART}$:
    6.  Loop for $t$ from $1$ to $N_t$:
        7.  Let $\tau = \pi(t)$.
        8.  **Warp:** Compute the warped volume at time $\tau$:
            $$ \tilde{f}^{(n)}(x, \tau) = f^{(n)}(x + u(x, \tau)) $$
            [S1].
        9.  **Forward Projection:** Compute the projection of the warped volume:
            $$ \tilde{p}(r, \tau) = R_{\theta(\tau)}[\tilde{f}^{(n)}(x, \tau)] $$
            [S1].
        10. **Residual:** Compute the residual between the acquired projection and the computed projection:
            $$ \rho(r, \tau) = p(r, \tau) - \tilde{p}(r, \tau) $$
            [S1].
        11. **Back-Projection:** Back-project the residual to the warped frame:
            $$ \Delta \tilde{f}^{(n+1)}(x) = B_{\theta(\tau)}[\rho(r, \tau)] $$
            where $B_{\theta(\tau)}$ is the back-projection operator [S1].
        12. **Unwarp:** Unwarp the correction term to the reference frame:
            $$ \Delta \hat{f}^{(n+1)}(x) = \Delta \tilde{f}^{(n+1)}(x - u(x, \tau)) $$
            [S1].
        13. **Update:** Update the volume in the reference frame:
            $$ f^{(n+1)}(x) = f^{(n)}(x) + \Delta \hat{f}^{(n+1)}(x) $$
            [S1].
        14. **Constraints:** Apply constraints to $f^{(n+1)}$ (e.g., positivity) [S1].
    15. Increment $n$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def DynART(projections, angles, N_t, N_DynART, u_initial):
    """
    projections: dict or array, p[r, t]
    angles: array, theta[t]
    N_t: int, number of time points
    N_DynART: int, max iterations
    u_initial: 3D array, initial displacement field u(x, t)
    """
    # Initialize volume and displacement
    f = zeros_like(volume_grid)
    u = u_initial.copy()
    
    # Permutation of acquisition times
    permutation = random_permutation(N_t)
    
    for i in range(1, N_DynART + 1):
        for t in range(1, N_t + 1):
            tau = permutation[t - 1] # 0-indexed access
            theta = angles[tau - 1]
            
            # 1. Warp volume
            # f_tilde(x, tau) = f(x + u(x, tau))
            # Note: Requires interpolation/resampling of f based on displacement u[:, :, tau]
            f_tilde = warp_volume(f, u[:, :, tau])
            
            # 2. Forward Projection
            # p_computed = R_theta[f_tilde]
            p_computed = radon_transform(f_tilde, theta)
            
            # 3. Residual
            # rho = p_acquired - p_computed
            p_acquired = projections[:, tau - 1]
            residual = p_acquired - p_computed
            
            # 4. Back-Projection
            # Delta_f_tilde = B_theta[rho]
            delta_f_tilde = back_projection(residual, theta)
            
            # 5. Unwarp correction
            # Delta_f_hat(x) = Delta_f_tilde(x - u(x, tau))
            # Note: Requires interpolation/resampling of delta_f_tilde
            delta_f_hat = unwarp_volume(delta_f_tilde, u[:, :, tau])
            
            # 6. Update Volume
            f = f + delta_f_hat
            
            # 7. Apply Constraints (e.g., Positivity)
            f = max(f, 0)
            
    return f, u
```
[G] The `warp_volume` and `unwarp_volume` functions involve resampling the 3D grid using trilinear or higher-order interpolation to handle the coordinate transformation $x \to x + u(x,t)$ and $x \to x - u(x,t)$. The `radon_transform` and `back_projection` are standard ART operators.

**Motion Update (P-DVC):**
After the volume reconstruction step, the displacement field $u$ is updated using Projection-based Digital Volume Correlation (P-DVC) [S1]. This step uses the current volume estimate to refine the motion field. The specific P-DVC algorithm is not detailed in the provided notes, but it is described as a separate step in the alternating procedure [S1].

**Pitfalls and Edge Cases:**
*   **Large Strains:** The unwarping formula $\Delta \hat{f}^{(n+1)}(x) = \Delta \tilde{f}^{(n+1)}(x - u(x, \tau))$ is equivalent to the implicit definition only for small strains and rotations [S1]. For large strains (e.g., engineering strains up to 20%), Eulerian registration must be used instead of Lagrangian displacement to achieve convergence [S1]. Ignoring this difference for large strains slows convergence but may not affect the final solution if the strain is not too large [S1].
*   **Convergence Criterion:** The standard ART convergence criterion based on the functional value cannot be used because the reconstruction is imperfect [S1]. A convergence criterion based on the variation of the functional or a maximum number of iterations $N_{DynART}$ should be set [S1].
*   **Multi-scale Approach:** To handle large displacement amplitudes and improve robustness, a multi-scale approach is coupled with the two-step procedure, resolving large-scale features first and progressively enriching the description with finer details [S1].

## Operating characteristics
*   **Complexity:** The method is iterative, with complexity proportional to $N_{DynART} \times N_t$ forward and back-projection operations, plus warping/unwarping costs [S1].
*   **Strengths:** Allows reconstruction of deforming volumes from a single scan; handles motion compensation explicitly [S1].
*   **Limitations:** Does not tolerate large displacement amplitudes without the multi-scale approach or Eulerian registration [S1]. Convergence can be slow for large strains if Lagrangian displacement is used incorrectly [S1].
*   **When to use:** When reconstructing volumes from moving or deforming objects where motion is significant but can be modeled by a displacement field [S1].
*   **When not to use:** For static objects, standard ART is simpler and sufficient [S1].

## Background
*   **Origin:** DynART was proposed in "Dynamic Tomographic Reconstruction of Deforming Volumes" (2018) [S1].
*   **History:** It extends the standard ART algorithm to account for motion by warping the volume with a displacement field before projection [S1]. The method combines ART reconstruction with P-DVC for motion identification [S1].
*   **Usage:** Used in materials science for dynamic tomographic reconstruction of deforming volumes [S1].

## Sources
[S1] Dynamic Tomographic Reconstruction of Deforming Volumes, https://doi.org/10.3390/ma11081395, fetched 2026-10-26
