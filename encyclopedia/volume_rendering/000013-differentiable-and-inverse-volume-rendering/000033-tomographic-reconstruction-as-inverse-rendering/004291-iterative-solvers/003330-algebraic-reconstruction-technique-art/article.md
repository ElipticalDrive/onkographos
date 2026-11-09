# Algebraic Reconstruction Technique (ART)
Also called: Kaczmarz method

## Overview
Algebraic Reconstruction Technique (ART) is an iterative algebraic method for solving the linear inverse problem of tomographic reconstruction, minimizing the quadratic norm of the difference between acquired projections and the projected reconstructed volume [S1]. It avoids explicit storage of the system matrix by iteratively updating the volume estimate using back-projection of residuals [S1]. Unlike [Simultaneous Algebraic Reconstruction Technique (SART)](10119-simultaneous-algebraic-reconstruction-technique-sart.md), which aggregates updates from a full projection image, ART updates the volume one angle (or ray set) at a time [S1].

## How it works
**Inputs:** Sinogram data $p(r, \theta)$, where $r$ is the detector coordinate and $\theta$ is the projection angle.
**Outputs:** Reconstructed volume $f(x)$.
**Parameters:**
- $\epsilon$: Convergence threshold, determined by noise and artifact levels [S1].
- $N_{ART}$: Maximum number of iterations [S1].
- $\pi$: Permutation of angle indices to order angles non-consecutively [S1].

**Data Structures:**
- $f^{(n)}(x)$: Volume estimate at iteration $n$, initialized to zero.
- $\rho(r, \theta)$: Residual sinogram, initialized to $p(r, \theta)$.
- $B_{\theta}$: Back-projection operator for angle $\theta$.
- $R_{\theta}$: Forward projection (Radon transform) operator for angle $\theta$.

**Formulas:**
The functional to minimize is:
$$ \Gamma_{ART}[\psi] = \sum_{r,\theta} \| \rho(r, \theta) \|^2 $$
where $\rho(r, \theta) = p(r, \theta) - R_{\theta}[\psi(x)]$ [S1].

**Steps:**
1. Initialize iteration counter $n=1$, volume $f^{(1)}(x) = 0$, and residual $\rho(r, \theta) = p(r, \theta)$ [S1].
2. Choose a permutation $\pi$ over the $N_\theta$ angle indices to ensure successive angles are not consecutive, which accelerates convergence [S1].
3. While $\|\rho(r, \theta)\| > \epsilon$:
    a. For $k = 1$ to $N_\theta$:
        i. Let $\theta_m = \pi(k)$ be the current angle [S1].
        ii. Compute the forward projection of the current volume estimate: $P_{\theta_m} = R_{\theta_m}[f^{(n)}(x)]$ [S1].
        iii. Update the residual for this angle: $\rho(r, \theta_m) = p(r, \theta_m) - P_{\theta_m}$ [S1].
        iv. Back-project the residual: $\Delta f^{(n+1)}(x) = B_{\theta_m}[\rho(r, \theta_m)]$ [S1].
        v. Update the volume: $f^{(n+1)}(x) = f^{(n)}(x) + \Delta f^{(n+1)}(x)$ [S1].
        vi. Apply constraints (e.g., positivity: $f^{(n+1)}(x) = \max(f^{(n+1)}(x), 0)$) [S1].
    b. Increment $n \leftarrow n + 1$ [S1].

**Pitfalls:**
- Convergence is slower if angles are sampled consecutively; using a permutation with large differences between successive angles is required for faster convergence [S1].
- The method may require regularization (e.g., Total Variation, binary constraints) to handle limited angle ranges or coarse sampling, as standard ART alone may produce artifacts in these cases [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ART(p, R_theta, B_theta, epsilon, max_iterations):
    """
    p: sinogram data, shape (num_detectors, num_angles)
    R_theta: function that projects volume f to sinogram for a given angle
    B_theta: function that back-projects residual to volume for a given angle
    epsilon: convergence threshold
    max_iterations: maximum number of full angle passes
    """
    num_angles = p.shape[1]
    # Initialize volume to zero
    f = initialize_volume_zero()
    
    # Initialize residual to sinogram data
    rho = p.copy()
    
    # Create permutation of angle indices
    pi = np.random.permutation(num_angles)
    
    for n in range(max_iterations):
        for k in range(num_angles):
            m = pi[k]
            theta_m = m # Assuming pi maps directly to angle indices
            
            # Forward project current volume estimate
            # Note: R_theta typically takes volume and angle
            p_est = R_theta(f, theta_m)
            
            # Update residual for this specific angle
            rho[:, m] = p[:, m] - p_est
            
            # Back-project residual to get correction
            delta_f = B_theta(rho[:, m], theta_m)
            
            # Update volume
            f = f + delta_f
            
            # Apply constraints (e.g., positivity)
            f = np.maximum(f, 0)
            
        # Check convergence based on residual norm
        if np.linalg.norm(rho) < epsilon:
            break
            
    return f
```

## Subtopics

### [Motion-Compensated Algebraic Reconstruction Technique](7422-motion-compensated-algebraic-reconstruction-technique.md)
Extends ART to deforming volumes by warping the volume with a displacement field before projection and unwarping the correction term.

### [Direct Linear Inversion (ART)](8506-direct-linear-inversion-art.md)
Solves the linear system via pseudo-inverse in a single step, avoiding iterative updates.

### [Simultaneous Algebraic Reconstruction Technique (SART)](10119-simultaneous-algebraic-reconstruction-technique-sart.md)
Updates the volume using all rays in a projection concurrently, offering faster convergence and better noise stability than ART.

## Operating characteristics
- **Time Complexity:** Iterative; convergence speed depends on the angle permutation and data conditioning. Generally requires a few iterations for convergence [S1].
- **Memory Complexity:** Avoids explicit storage of the large system matrix $M$, storing only the volume and sinogram data [S1].
- **Strengths:** Handles non-ideal conditions (e.g., limited angles, coarse sampling) better than Fourier-based methods when combined with regularization [S1].
- **Limitations:** Higher computational cost than Filtered Back Projection (FBP) [S1]. Convergence can be slow if angle ordering is not optimized [S1].

## Background
ART is based on the Kaczmarz method for solving linear systems [S1]. It was adapted for tomographic reconstruction to exploit the linear structure of the problem while avoiding the computational burden of writing out the full linear system [S1]. It is widely used in materials science and medical imaging where regularization can improve quality over FBP [S1].

## Sources
[S1] Dynamic Tomographic Reconstruction of Deforming Volumes, https://doi.org/10.3390/ma11081395, fetched 2026-10-26
