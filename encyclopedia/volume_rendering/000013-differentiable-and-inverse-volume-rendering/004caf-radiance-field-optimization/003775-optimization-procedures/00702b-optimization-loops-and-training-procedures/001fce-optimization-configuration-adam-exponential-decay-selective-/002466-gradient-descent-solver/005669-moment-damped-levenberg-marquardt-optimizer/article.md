# Moment-Damped Levenberg-Marquardt Optimizer

Also called: Hybrid LM-Adam Optimizer, Moment-LM Optimizer, TurboGS Optimizer, hybrid moment-damped LM, sparse moment-damped Levenberg-Marquardt

## Overview
A hybrid optimization strategy for sparse-pixel training that partitions Gaussian parameters into geometry and base appearance blocks, solving them via small Levenberg-Marquardt (LM) systems with Adam-style moment damping, while updating Higher-order Spherical Harmonic (SH) coefficients separately with Adam [S1, unofficial]. It sets itself apart from its parent [Gradient Descent Solver](3280-gradient-descent-solver.md) by introducing Hessian-informed updates for geometry and appearance, and from [CGLS Solver (Conjugate Gradient Least Squares)](3285-cgls-solver-conjugate-gradient-least-squares.md) by using direct Cholesky decomposition on small blocks rather than iterative linear solving [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Current Gaussian parameters $\theta$, Adam first moment $m$, Adam second moment $v$, Jacobian $J$ for geometry and base appearance blocks, loss gradient $m$ (Adam first moment of loss).
- Output: Parameter update $\Delta \theta$.

**Data Structures:**
- Gaussian parameters $\theta$ are partitioned into:
  - Geometry block $g \in \mathbb{R}^{10}$: position (3), scale (3), rotation (4 quaternion).
  - Base appearance block $a \in \mathbb{R}^{4}$: RGB DC coefficients (3), opacity (1).
  - Higher-order SH coefficients: updated separately.

**Parameters:**
- $\epsilon$: Small constant for numerical stability (standard Adam value, e.g., $10^{-8}$) [G].
- $\eta$: Learning rate for Adam update of SH coefficients [G].
- $\sqrt{v}$: Element-wise square root of the second moment vector $v$, serving as diagonal damping.

**Steps:**
1. **Partition Parameters:** Separate $\theta$ into $g$ and $a$.
2. **Compute Jacobian:** Calculate the Jacobian $J_g$ for geometry and $J_a$ for base appearance with respect to the rendered loss.
3. **Solve LM System for $g$ and $a$:** For each block $b \in \{g, a\}$, solve the linear system:
   $$ (J_b^\top J_b + \text{diag}(\sqrt{v_b} + \epsilon)) \Delta b = -m_b $$
   where $m_b$ is the Adam first moment for block $b$, and $v_b$ is the Adam second moment for block $b$. The term $\sqrt{v_b}$ acts as a moment-based diagonal damping term to stabilize sparse gradients [S1, unofficial].
4. **Solve via Cholesky:** Perform Cholesky decomposition on the matrix $(J_b^\top J_b + \text{diag}(\sqrt{v_b} + \epsilon))$ to solve for $\Delta b$ efficiently [S1, unofficial].
5. **Update SH Coefficients:** Update higher-order SH coefficients $\theta_{SH}$ using Adam:
   $$ \Delta \theta_{SH} = -\eta \frac{\hat{m}_{SH}}{\sqrt{\hat{v}_{SH}} + \epsilon} $$
   where $\hat{m}_{SH}$ and $\hat{v}_{SH}$ are bias-corrected Adam moments [S1, unofficial].
6. **Apply Updates:** Apply $\Delta g$, $\Delta a$, and $\Delta \theta_{SH}$ to the respective parameters.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def moment_damped_lm_update(g, a, theta_sh, m_g, m_a, m_sh, v_g, v_a, v_sh, eta, epsilon):
    """
    Perform one step of Moment-Damped Levenberg-Marquardt optimization.
    
    Args:
        g: Geometry parameters (R^10)
        a: Base appearance parameters (R^4)
        theta_sh: Higher-order SH coefficients
        m_g, m_a, m_sh: Adam first moments for respective blocks
        v_g, v_a, v_sh: Adam second moments for respective blocks
        eta: Learning rate for SH update
        epsilon: Small constant for numerical stability
        
    Returns:
        Updated g, a, theta_sh
    """
    # Step 1: Solve LM system for geometry block g
    # J_g is Jacobian of loss w.r.t g
    # H_g = J_g^T J_g
    H_g = compute_hessian_approximation(J_g)
    # Damping term: sqrt(v_g) + epsilon
    damping_g = sqrt(v_g) + epsilon
    # System: (H_g + diag(damping_g)) * delta_g = -m_g
    A_g = H_g + diag(damping_g)
    b_g = -m_g
    # Solve via Cholesky decomposition
    delta_g = solve_cholesky(A_g, b_g)
    
    # Step 2: Solve LM system for base appearance block a
    # J_a is Jacobian of loss w.r.t a
    H_a = compute_hessian_approximation(J_a)
    damping_a = sqrt(v_a) + epsilon
    A_a = H_a + diag(damping_a)
    b_a = -m_a
    delta_a = solve_cholesky(A_a, b_a)
    
    # Step 3: Update SH coefficients using Adam
    # Bias-corrected moments
    m_hat_sh = bias_correct(m_sh)
    v_hat_sh = bias_correct(v_sh)
    delta_sh = -eta * (m_hat_sh / (sqrt(v_hat_sh) + epsilon))
    
    # Step 4: Apply updates
    g_new = g + delta_g
    a_new = a + delta_a
    theta_sh_new = theta_sh + delta_sh
    
    return g_new, a_new, theta_sh_new

def compute_hessian_approximation(J):
    """Compute J^T J."""
    return J.T @ J

def diag(v):
    """Create diagonal matrix from vector."""
    return np.diag(v)

def solve_cholesky(A, b):
    """Solve Ax = b using Cholesky decomposition."""
    L = cholesky(A)
    y = solve_triangular(L, b, lower=True)
    x = solve_triangular(L.T, y, lower=False)
    return x

def bias_correct(m, t):
    """Bias correction for Adam moments."""
    # Assuming t is the step number
    return m / (1 - 0.9**t) # Beta1 = 0.9
```

**Edge Cases and Pitfalls:**
- Ignoring SH cross-parameter correlations is a trade-off; joint optimization would substantially increase Jacobian size and memory overhead [S1, unofficial].
- Sparse gradients can be unstable; the moment-based damping $\sqrt{v}$ is critical for stabilization [S1, unofficial].

## Subtopics

### Hybrid Optimizer
A tailored optimizer that couples Hessian-informed updates with Adam moment damping to stabilize and improve convergence under sparse supervision [S1, unofficial]. It integrates Adam moments with Hessian-informed updates [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Solving small LM systems via Cholesky decomposition is efficient for the fixed small block sizes (10 and 4 dimensions). The overall complexity is dominated by Jacobian computation and Cholesky factorization, which is lightweight for these small blocks.
- **Memory Complexity:** Low memory overhead compared to full Newton methods due to block partitioning and ignoring SH correlations.
- **Measured Performance:** On Mip-NeRF 360, Moment-LM achieves 29.50dB PSNR vs 29.42dB for Sparse Adam-only, with identical training time (90s) and NGS (0.64M) [S1, unofficial].
- **Scaling:** Suitable for sparse-pixel training scenarios where full dense optimization is too costly.
- **Strengths:** Higher PSNR/SSIM, lower LPIPS, and more stable high-frequency details compared to sparse Adam-only optimization under similar training times [S1, unofficial].
- **Limitations:** Trade-off in accuracy due to ignoring SH cross-parameter correlations [S1, unofficial].

## Background
Originated in TurboGS: Accelerating 3D Gaussian Splatting via Error-Guided Sparse Pixel Sampling and Optimization [S1, unofficial]. It is used in the context of 3D Gaussian Splatting (3DGS) to accelerate training while maintaining or improving rendering quality.

## Sources
[S1] TurboGS: Accelerating 3D Gaussian Splatting via Error-Guided Sparse Pixel Sampling and Optimization, https://doi.org/10.48550/arXiv.2606.15924, fetched 2026-10-29 (unofficial)
