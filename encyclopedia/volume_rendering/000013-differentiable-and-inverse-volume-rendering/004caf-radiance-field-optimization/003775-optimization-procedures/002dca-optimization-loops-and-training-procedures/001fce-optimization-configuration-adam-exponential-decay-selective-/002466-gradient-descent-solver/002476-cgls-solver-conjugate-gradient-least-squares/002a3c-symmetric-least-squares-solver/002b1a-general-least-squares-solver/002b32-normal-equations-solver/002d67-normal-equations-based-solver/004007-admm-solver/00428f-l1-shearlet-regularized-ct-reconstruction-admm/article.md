# L1 Shearlet Regularized CT Reconstruction (ADMM)
Also called: ADMM Shearlet Reconstruction, Bubba et al. Algorithm

## Overview
This method solves the L1 shearlet regularized computed tomography (CT) reconstruction problem using the Alternating Direction Method of Multipliers (ADMM) [S1, unofficial]. It minimizes a composite objective consisting of an L1 norm of shearlet coefficients (weighted) and a quadratic data fidelity term, subject to non-negativity constraints on the image [S1, unofficial]. It sets itself apart from standard filtered backprojection by enforcing sparsity in the shearlet domain to handle limited-angle artifacts, and from its parent [ADMM Solver](10107-admm-solver.md) by specifically structuring the proximal steps for shearlet shrinkage and non-negativity clamping [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `y` (measured sinogram), `radon` (forward Radon projection operator $R$), `shearlet` (shearlet transform operator $SH$).
- Output: `f` (reconstructed image volume).

**Data Structures:**
- `f`: Image tensor, shape matching the backprojected image `bp`.
- `bp`: Backprojected image tensor, shape matching `f`.
- `sc`: Shearlet coefficients of `bp`, shape matching `sh_f`.
- `u1`, `z1`: Dual and primal variables for shearlet constraint, shape matching `sc`.
- `u2`, `z2`: Dual and primal variables for non-negativity constraint, shape matching `bp`.
- `cg_y`: Right-hand side vector for the conjugate gradient linear system.
- `w`: Weight vector for the L1 norm, shape matching `sh_f`.

**Parameters:**
- `p0`: Data fidelity weight. Typical value: $0.02$ [S1, unofficial].
- `p1`: Regularization weight. Typical value: $0.1$ [S1, unofficial].
- `num_iterations`: Number of ADMM iterations.
- `max_iter`: Maximum iterations for the inner Conjugate Gradient solver. Typical value: $50$ [S1, unofficial].
- `n_scales`: Number of shearlet scales. Typical value: $5$ [S1, unofficial].
- `w` calculation: $w = 3^{\text{scales}} / 400$ [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:**
   - Initialize `u2`, `z2`, `u1`, `z1`, `f` to zero tensors with shapes matching `bp` and `sc` respectively [S1, unofficial].
   - Compute initial backprojection `bp = radon.backward(y)` [S1, unofficial].
   - Compute initial shearlet coefficients of backprojection `sc = shearlet.forward(bp)` [S1, unofficial].
   - Compute weights `w = 3**n_scales / 400` and reshape to broadcastable dimensions [S1, unofficial].

2. **ADMM Loop (for `i` in `range(num_iterations)`):**
   - **Compute CG RHS:**
     $$ cg\_y = p_0 \cdot bp + p_1 \cdot \text{shearlet.backward}(z_1 - u_1) + (z_2 - u_2) $$
     [S1, unofficial]
   - **Solve for `f` (Primal Update):**
     Solve the linear system for `f` using Conjugate Gradient Least Squares (CGLS):
     $$ \min_f \left( p_0 \cdot \| R f - y \|_2^2 + (1 + p_1) \cdot \| f \|_2^2 \right) $$
     This is equivalent to solving:
     $$ (p_0 \cdot R^T R + (1 + p_1) \cdot I) f = cg\_y $$
     Using the solver: `f = cg(lambda x: p0 * radon.backward(radon.forward(x)) + (1 + p1) * x, f.clone(), cg_y, max_iter=50)` [S1, unofficial].
   - **Shearlet Transform:**
     Compute shearlet coefficients of the new `f`:
     $$ sh\_f = \text{shearlet.forward}(f) $$
     [S1, unofficial]
   - **Update `z1` (Shearlet Proximal Step):**
     Apply soft-thresholding (shrinkage) to update the primal variable for the shearlet constraint:
     $$ z_1 = \text{shrink}(sh\_f + u_1, \frac{p_0}{p_1} \cdot w) $$
     [S1, unofficial]
   - **Update `z2` (Non-negativity Proximal Step):**
     Project onto the non-negative orthant:
     $$ z_2 = \max(0, f + u_2) $$
     Implemented as `z2 = (f + u2).clamp_min(0)` [S1, unofficial].
   - **Update Dual Variables:**
     $$ u_1 = u_1 + sh\_f - z_1 $$
     $$ u_2 = u_2 + f - z_2 $$
     [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def l1_shearlet_admm(y, radon, shearlet, num_iterations=100, max_cg_iter=50):
    # Parameters
    p0 = 0.02
    p1 = 0.1
    n_scales = 5
    
    # Initialization
    bp = radon.backward(y)
    sc = shearlet.forward(bp)
    
    u2 = torch.zeros_like(bp)
    z2 = torch.zeros_like(bp)
    u1 = torch.zeros_like(sc)
    z1 = torch.zeros_like(sc)
    f = torch.zeros_like(bp)
    
    # Weights calculation
    w = (3 ** n_scales) / 400
    w = w.view(1, -1, 1, 1).cuda() # Assuming CUDA context
    
    # Pre-define linear operator for CG
    def linear_op(x):
        return p0 * radon.backward(radon.forward(x)) + (1 + p1) * x

    for i in range(num_iterations):
        # 1. Compute CG RHS
        cg_y = p0 * bp + p1 * shearlet.backward(z1 - u1) + (z2 - u2)
        
        # 2. Solve for f using Conjugate Gradient
        # Solving (p0 * R^T R + (1+p1) * I) f = cg_y
        f = cg(linear_op, f.clone(), cg_y, max_iter=max_cg_iter)
        
        # 3. Shearlet Transform
        sh_f = shearlet.forward(f)
        
        # 4. Update z1 (Shrinkage)
        threshold = (p0 / p1) * w
        z1 = shrink(sh_f + u1, threshold)
        
        # 5. Update z2 (Non-negativity)
        z2 = torch.clamp(f + u2, min=0.0)
        
        # 6. Update dual variables
        u1 = u1 + sh_f - z1
        u2 = u2 + f - z2
        
    return f

def shrink(a, b):
    """Soft-thresholding function."""
    return torch.clamp(torch.abs(a) - b, min=0.0) * torch.sign(a)
```
[S1, unofficial]

**Edge Cases and Pitfalls:**
- **Precision:** The implementation supports single and half precision. Relative error between single and half precision for the Shepp-Logan phantom is $1.40 \cdot 10^{-4}$, and reconstruction MSE error difference is negligible ($2.0566 \cdot 10^{-4}$ vs $2.0567 \cdot 10^{-4}$) [S1, unofficial].
- **Limited Angles:** The method is designed for limited-angle tomography where $R$ represents forward projection with limited angles [S1, unofficial].
- **Numerical Stability:** The inner CG solver requires a well-conditioned system. The term $(1+p_1)I$ acts as a regularization term to ensure invertibility [S1, unofficial].

## Subtopics

### [TomoAnchor V3 Z-Sharded ADMM-GMRES](12610-tomoanchor-v3-z-sharded-admm-gmres.md)
A GPU-based cone-beam CT reconstruction method that uses TIGRE for forward/backward projections and an ADMM solver with restarted GMRES for the x-update step to handle numerical mismatch between TIGRE's $Ax$ and $A^Tb$ operators. It sets itself apart from its parent [L1 Shearlet Regularized CT Reconstruction (ADMM)](10811-l1-shearlet-regularized-ct-reconstruction-admm.md) by returning to TIGRE reconstruction physics, using matched $A$ and $A^T$ operators from TIGRE, and employing GMRES to solve the linear system arising from the ADMM x-update due to potential numerical mismatch between the forward and adjoint operators.

## Operating characteristics
- **Time Complexity:** The dominant cost per iteration is the forward/backward Radon projections and the shearlet transform, plus the conjugate gradient solve. The CG solve converges in up to `max_iter` (50) steps per ADMM iteration [S1, unofficial].
- **Memory Complexity:** Requires storage for `f`, `bp`, `sc`, `u1`, `z1`, `u2`, `z2`, and `cg_y`. All operations are performed on GPU, avoiding CPU-GPU memory copies in the main loop [S1, unofficial].
- **Measured Performance:** On a Tesla V100 GPU, the implementation runs in 1.6 seconds per reconstruction, compared to 2.5 minutes on an Intel i7 CPU in the original paper, representing a 125x speedup [S1, unofficial]. With batch processing, the average time is 1.2 seconds per image [S1, unofficial].
- **Strengths:** All operations (including shearlet transforms) are performed on the GPU, maximizing execution speed. No CPU-GPU memory copies occur in the main loop. Supports batch processing for parallel sinogram handling [S1, unofficial].
- **Limitations:** Performance depends on the convergence of the inner CG solver. The method is specific to L1 shearlet regularization and may not be optimal for other priors.

## Background
The method is based on the work of Bubba et al. [4], who proposed solving the L1 shearlet regularized CT reconstruction problem as a preprocessing step for deep learning-based inverse problems [S1, unofficial]. The original implementation was on CPU, but a GPU-accelerated version using TorchRadon was developed to leverage modern hardware [S1, unofficial]. The algorithm uses the Alternating Direction Method of Multipliers (ADMM) to decouple the data fidelity term from the sparsity-promoting shearlet prior and the non-negativity constraint [S1, unofficial].

## Sources
[S1] TorchRadon: Fast Differentiable Routines for Computed Tomography, https://doi.org/10.48550/arxiv.2009.14788, fetched 2026-10-26 (unofficial)
