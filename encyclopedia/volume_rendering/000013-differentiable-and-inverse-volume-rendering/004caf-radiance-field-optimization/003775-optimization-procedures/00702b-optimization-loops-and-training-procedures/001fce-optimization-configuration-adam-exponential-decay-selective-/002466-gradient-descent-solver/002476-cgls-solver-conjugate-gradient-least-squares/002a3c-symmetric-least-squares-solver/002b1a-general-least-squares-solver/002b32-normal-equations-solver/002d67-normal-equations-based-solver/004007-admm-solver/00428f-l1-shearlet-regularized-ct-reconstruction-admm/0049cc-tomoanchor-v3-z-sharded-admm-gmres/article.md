# TomoAnchor V3 Z-Sharded ADMM-GMRES

Also called: V3 solver

## Overview
A GPU-based cone-beam CT reconstruction method that uses TIGRE for forward/backward projections and an ADMM solver with restarted GMRES for the x-update step to handle numerical mismatch between TIGRE's $Ax$ and $A^Tb$ operators [S1, unofficial]. It sets itself apart from its parent [L1 Shearlet Regularized CT Reconstruction (ADMM)](10811-l1-shearlet-regularized-ct-reconstruction-admm.md) by returning to TIGRE reconstruction physics, using matched $A$ and $A^T$ operators from TIGRE, and employing GMRES to solve the linear system arising from the ADMM x-update due to potential numerical mismatch between the forward and adjoint operators [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Projection data $y$, forward projection operators $A_s$, backprojection operators $B_s$, weight matrix $W$, regularization parameter $\rho$, and initial volume estimate $x$ [S1, unofficial].
- Output: Reconstructed volume $x$, split into shards $x_s$ [S1, unofficial].

**Data Structures:**
- `ShardedVolume`: A dataclass with fields `shards` (tuple of arrays), `volume_shards` (tuple of `VolumeShard` objects), and `xp` (array library, default `np`) [S1, unofficial].

**Numbered Steps:**
1. **Initialization**: Initialize volume $x$ and shards $x_s$. Verify that the number of arrays matches the number of volume shards, raising `ValueError` if mismatched. Verify that shard local shape matches array shape, raising `ValueError` if `tuple(int(value) for value in array.shape) != tuple(int(value) for value in shard.local_shape)` [S1, unofficial].
2. **ADMM Loop**: For each ADMM iteration:
   a. **x-update**: Solve the linear system using restarted GMRES. The core matrix-vector product computes the global projection first, then the shard-wise residual [S1, unofficial].
   b. **z-update**: Update auxiliary variables based on current $x$ and dual variable $u$ [G].
   c. **u-update**: Update dual variable $u$ based on current $x$ and $z$ [G].
3. **Termination**: Check convergence criteria (e.g., relative change in $x$, residual norm). If converged, stop; otherwise, return to step 2 [G].

**Formulas:**
Each matvec computes the global projection first, then the shard output:
$$ y = \sum_s A_s x_s $$
$$ \text{out}_s = B_s W^2 y + \rho (G^T G x)_s $$
where $A_s$ is the forward projection for shard $s$, $B_s$ is the backprojection for shard $s$, $W$ is the weight matrix, $\rho$ is the regularization parameter, $G$ is a linear operator, $x_s$ is the shard of volume $x$, and $y$ is the global projection [S1, unofficial].

**Parameters:**
- Residency mode: Determines host/device data management [S1, unofficial].
  - `staged_tigre`: Production default. Uses stock TIGRE NumPy wrappers with pinned/staged host buffers. Preserves TIGRE physics but is not strict GPU-resident [S1, unofficial].
  - `device_tigre`: Strict native bridge backend. Keeps ADMM-GMRES solver vectors host-resident, streams one z-shard at a time to GPU, and calls TIGRE CUDA kernels through the TomoAnchor native CuPy device-pointer bridge. Requires running `python tools/build_tigre_device_backend.py --preflight-only` then `python tools/build_tigre_device_backend.py --tigre-source tools/TIGRE` [S1, unofficial].
  - `device_tigre_resident`: Full CuPy-resident backend. Keeps all V3 solver shards on selected GPUs. Only appropriate when the whole ADMM-GMRES state fits in aggregate VRAM [S1, unofficial].
  - `auto`: Uses `device_tigre` only when available, otherwise `staged_tigre` unless strict residency is required [S1, unofficial].
- Number of GMRES restarts: Not given in the sources [S1, unofficial].
- Tolerance for GMRES convergence: Not given in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
- Mismatch between the number of arrays and volume shards raises a `ValueError` [S1, unofficial].
- Mismatch between shard local shape and array shape raises a `ValueError` [S1, unofficial].
- Numerical mismatch between TIGRE's $Ax$ and $A^Tb$ operators necessitates GMRES instead of CG [S1, unofficial].
- The volume is split only along the z-axis; slabs are not reconstructed independently [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def admm_gmres_x_update(x_shards, A_s, B_s, W, rho, G):
    # Compute global projection y = sum_s A_s x_s
    y = sum(A_s[i](x_shards[i]) for i in range(len(x_shards)))
    
    # Compute RHS for each shard: out_s = B_s W^2 y + rho (G^T G x)_s
    rhs = []
    for i in range(len(x_shards)):
        rhs_i = B_s[i](W**2 @ y) + rho * (G.T @ G @ x_shards[i])
        rhs.append(rhs_i)
    
    # Define matvec for the linear system
    def matvec(v_shards):
        v_proj = sum(A_s[i](v_shards[i]) for i in range(len(v_shards)))
        return [B_s[i](W**2 @ v_proj) + rho * (G.T @ G @ v_shards[i]) for i in range(len(v_shards))]
    
    # Note: The sources do not provide the GMRES solver implementation.
    # A standard restarted GMRES implementation should be used here.
    x_new = gmres_restarted(matvec, rhs, x_shards)
    return x_new
```

## Operating characteristics
- **Strengths**: Uses TIGRE for accurate forward and backward projections, handles numerical mismatch between operators using GMRES, supports GPU acceleration via residency modes [S1, unofficial].
- **Limitations**: Numerical mismatch between operators may require careful tuning of GMRES parameters, residency modes may impact performance and memory usage [S1, unofficial].
- **When to Use**: Suitable for cone-beam CT reconstruction where TIGRE's physics are desired and GPU acceleration is needed [S1, unofficial].
- **When Not to Use**: Not suitable when strict GPU residency is required and the whole ADMM-GMRES state does not fit in aggregate VRAM, or when TIGRE's numerical mismatch cannot be handled by GMRES [S1, unofficial].

## Background
Originated as part of the TomoAnchor V3 project, which aims to provide a flexible and efficient framework for CT reconstruction [S1, unofficial]. The method builds on the L1 Shearlet Regularized CT Reconstruction (ADMM) algorithm, adapting it to use TIGRE for projections and GMRES for the x-update step [S1, unofficial]. It is used in research and production environments for high-quality CT reconstruction [S1, unofficial].

## Sources
[S1] alidarbandi/TomoAnchor-V3, https://github.com/alidarbandi/TomoAnchor-V3, fetched 2026-10-27 (unofficial)
