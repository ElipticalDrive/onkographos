# Polynomial Motion Field

Also called: Hierarchical motion model, Parametric Motion Field, Polynomial warp field

## Overview
The Polynomial Motion Field is a parametric deformation model used in dynamic volume rendering to represent temporal motion as a continuous function of time. It solves the problem of reconstructing 4D (3D space + time) scenes from sparse or limited view measurements by warping a static template volume to arbitrary time points using polynomial coefficients. Unlike neural deformation fields that map $(x, y, z, t)$ to displacement via a network, this method explicitly parameterizes the motion trajectory at each voxel as a polynomial, allowing for efficient optimization and arbitrary temporal upsampling. It differs from spline-based methods by using global polynomial bases rather than piecewise continuous splines, and from neural fields by avoiding the computational overhead of network inference during rendering [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ sparse or limited-view 2D projections (sinograms) sampled at discrete time points $t_{i=0 \dots N-1}$. A static template volume $\hat{\sigma}$ (or an initial estimate).
*   **Output:** A Motion Coefficient Tensor $C$ and the reconstructed dynamic volume $\sigma(x, y, z, t)$ for any $t \in [0, 1]$.

**Data Structures:**
*   **Motion Coefficient Tensor $C$:** A tensor of shape $\mathbb{R}^{\beta^3 \times k \times 3}$.
    *   $\beta^3$: The spatial resolution of the scene (number of voxels).
    *   $k$: The polynomial order (degree).
    *   $3$: The three spatial dimensions (x, y, z).
    *   Each voxel stores $k$ coefficients for each of the 3 spatial displacement components.
*   **Time Samples:** $N$ time points $t_i$ linearly spaced in $[0, 1]$.

**Algorithm Steps:**

1.  **Initialization:** Initialize the Motion Coefficient Tensor $C$. To mitigate poor motion reconstruction at full resolution, initialize $C$ at a base spatial resolution $\alpha$ (where $\alpha < \beta$), resulting in $C_\alpha \in \mathbb{R}^{\alpha^3 \times k \times 3}$ [S1].
2.  **Hierarchical Upsampling (Coarse-to-Fine):** Iteratively increase the spatial resolution $\alpha$ (e.g., $2^3 \to 4^3 \to 8^3 \to 16^3$). At each step, upsample the current coefficient tensor $C_{\alpha_i}$ to the next resolution $C_{\alpha_{i+1}}$ using linear upsampling $U$:
    $$ C_{\alpha_{i+1}} = U(C_{\alpha_i}) $$
    where $U: \mathbb{R}^{\alpha_i^3} \to \mathbb{R}^{\alpha_{i+1}^3}$ [S1].
3.  **Warp Field Computation:** For a specific time $t_i$, compute the 3D displacement vector $W(C, t_i)$ for all voxels using the polynomial expansion:
    $$ W(C, t_i) = \sum_{j=0}^{k} C_{:, j, :} t_i^j $$
    where $W(C, t_i) \in \mathbb{R}^{\beta^3 \times 3}$ is the warp field at time $t_i$, and $C_{:, j, :}$ denotes the slice of coefficients for the $j$-th polynomial order across all voxels and spatial dimensions [S1].
4.  **Differentiable Grid Sampling:** Warp the template volume $\hat{\sigma}$ using the computed displacement field. The volume at time $t_i$ is obtained via a differentiable grid sampling function $fn$:
    $$ \sigma(x, y, z, t_i) = fn(W(C, t_i), \hat{\sigma}) $$
    This function interpolates the template values at the displaced coordinates [S1].
5.  **Optimization:** Minimize the reconstruction error between the rendered projections of $\sigma(x, y, z, t_i)$ and the measured sinograms. The optimization encourages the polynomial coefficients to fit a continuous representation of motion, allowing the model to be evaluated at arbitrary time points $t$ not present in the training set [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def compute_warp_field(C, time_t):
    """
    Compute the displacement field W for a given time t.
    C: Tensor of shape (num_voxels, k, 3)
    time_t: Scalar or Tensor of time values in [0, 1]
    Returns: W of shape (num_voxels, 3)
    """
    # C[:, j, :] are the coefficients for order j
    # We compute sum(C[:, j, :] * t^j for j in 0..k)
    W = torch.zeros_like(C[:, 0, :]) # Shape (num_voxels, 3)
    t_power = torch.ones_like(time_t) # t^0 = 1
    
    # Ensure time_t is broadcastable to (num_voxels, 1) if needed
    # Assuming time_t is a scalar or 1D tensor matching the batch/time dimension
    # For a single time point evaluation:
    t_val = time_t if isinstance(time_t, float) else time_t.item()
    
    for j in range(C.shape[1]): # Iterate over polynomial order k
        if j > 0:
            t_power = t_power * t_val
        # Add C[:, j, :] * t^j to W
        # C[:, j, :] is (num_voxels, 3)
        W = W + C[:, j, :] * t_power
    return W

def hierarchical_upsample(C_alpha, target_alpha):
    """
    Upsample the coefficient tensor from resolution alpha to target_alpha.
    C_alpha: Tensor of shape (alpha^3, k, 3)
    target_alpha: Integer target resolution
    Returns: Tensor of shape (target_alpha^3, k, 3)
    """
    # Reshape to 3D grid for interpolation
    # alpha^3 voxels -> reshape to (alpha, alpha, alpha, k, 3)
    # Note: The source specifies linear upsampling U. 
    # In practice, this involves reshaping, interpolating the spatial dims, 
    # and reshaping back.
    alpha = int(C_alpha.shape[0] ** (1/3))
    k = C_alpha.shape[1]
    
    # Reshape to (alpha, alpha, alpha, k, 3)
    C_grid = C_alpha.view(alpha, alpha, alpha, k, 3)
    
    # Interpolate spatial dimensions (alpha -> target_alpha)
    # Using trilinear interpolation on the first 3 dims
    # PyTorch grid_sample or F.interpolate can be used.
    # F.interpolate expects (N, C, D, H, W) or similar.
    # Here we treat k*3 as channels.
    C_interp = F.interpolate(
        C_grid.permute(0, 1, 2, 4, 3).contiguous().view(1, 1, alpha, alpha, alpha, k*3), # Hacky reshape for interp
        size=(target_alpha, target_alpha, target_alpha),
        mode='trilinear',
        align_corners=False
    )
    
    # Reshape back to (target_alpha^3, k, 3)
    # This step is conceptual; exact tensor manipulation depends on framework
    # The source states: "constrained to simple trilinear interpolation"
    # We assume a helper `trilinear_upsample` exists for the tensor structure
    C_new = trilinear_upsample_tensor(C_grid, target_alpha)
    return C_new

def trilinear_upsample_tensor(C_grid, target_alpha):
    """
    Helper to perform trilinear interpolation on the spatial dimensions of C_grid.
    C_grid: (alpha, alpha, alpha, k, 3)
    """
    # Reshape to (alpha, alpha, alpha, k*3)
    C_flat = C_grid.view(C_grid.shape[0], C_grid.shape[1], C_grid.shape[2], -1)
    # Add batch and channel dims for F.interpolate
    C_4d = C_flat.unsqueeze(0).unsqueeze(0) # (1, 1, alpha, alpha, alpha, k*3) -> needs correct dim order
    # Correct dim order for F.interpolate in 3D: (N, C, D, H, W)
    # C_flat is (alpha, alpha, alpha, k*3). Transpose to (k*3, alpha, alpha, alpha)
    C_trans = C_flat.permute(3, 0, 1, 2) # (k*3, alpha, alpha, alpha)
    C_batch = C_trans.unsqueeze(0).unsqueeze(0) # (1, 1, k*3, alpha, alpha, alpha) - wait, F.interpolate 3D is (N, C, D, H, W)
    # Actually, F.interpolate for 3D takes (N, C, D, H, W).
    # So C_trans should be (k*3, alpha, alpha, alpha).
    # Let's use grid_sample or a simpler view.
    # For simplicity, assuming a standard library call for 3D trilinear upsample of the spatial grid
    # while keeping k and 3 dimensions intact.
    
    # Re-evaluating based on "simple trilinear interpolation for upsampling the field itself"
    # The field is C. We upsample the spatial grid.
    # We can view C as a volume of vectors.
    
    # Reshape to (alpha, alpha, alpha, k*3)
    # Interpolate the first 3 dims.
    # Using F.interpolate on (N, C, D, H, W)
    # We treat k*3 as the channel dimension C.
    # Shape: (1, k*3, alpha, alpha, alpha)
    
    C_view = C_flat.permute(3, 0, 1, 2).unsqueeze(0).unsqueeze(0) # (1, 1, k*3, alpha, alpha, alpha) -> Wrong.
    # F.interpolate 3D: (N, C, D, H, W)
    # C_flat is (alpha, alpha, alpha, k*3).
    # Permute to (k*3, alpha, alpha, alpha).
    # Add N, C dims: (1, k*3, alpha, alpha, alpha).
    
    C_interp_4d = F.interpolate(
        C_flat.permute(3, 0, 1, 2).unsqueeze(0), # (1, k*3, alpha, alpha, alpha)
        size=(target_alpha, target_alpha, target_alpha),
        mode='trilinear',
        align_corners=False
    )
    
    # Remove N, C dims -> (k*3, target, target, target)
    C_interp_flat = C_interp_4d.squeeze(0).squeeze(0)
    
    # Reshape to (target, target, target, k, 3)
    C_interp_grid = C_interp_flat.view(target_alpha, target_alpha, target_alpha, k, 3)
    
    # Flatten spatial dims to (target^3, k, 3)
    return C_interp_grid.view(-1, k, 3)

def render_dynamic_volume(C, template_sigma, time_t, resolution_beta):
    """
    Render the volume at time t.
    C: (beta^3, k, 3)
    template_sigma: (beta^3,) or (beta, beta, beta)
    time_t: float
    resolution_beta: int
    """
    # 1. Compute Warp
    W = compute_warp_field(C, time_t) # (beta^3, 3)
    
    # 2. Reshape W to grid coordinates
    # W is displacement. We need source coordinates.
    # Assuming template_sigma is defined on a grid.
    # We need to sample template_sigma at (grid_coords + W).
    
    # Create grid coordinates for resolution_beta
    # This part depends on the specific differentiable grid sampling implementation [S1]
    # Typically involves creating a meshgrid and adding W.
    
    # Placeholder for the specific `fn` implementation from [34]
    # fn(W, sigma) = sigma(x + Wx, y + Wy, z + Wz)
    
    # For the purpose of this article, we assume `fn` handles the grid sampling.
    # The key is that W is computed from C and t.
    
    # Return the warped volume
    # In a real implementation, this would return the 3D tensor
    pass
```

**Parameters:**
*   **Polynomial Order $k$:** The degree of the polynomial. Typically $k=5$ is sufficient for deformable and periodic motion [S1].
*   **Base Resolution $\alpha$:** The initial resolution for coarse-to-fine optimization. Examples include $2^3, 4^3, 8^3, 16^3$ [S1].
*   **Target Resolution $\beta$:** The final spatial resolution of the scene [S1].

**Edge Cases and Pitfalls:**
*   **Upsampling Accuracy:** The parametric representation constrains the upsampling of the motion field itself to simple trilinear interpolation. This may limit accuracy compared to other interpolation methods, especially for high-frequency motion details [S1].
*   **Resolution Scaling:** Attempting to estimate the warp field directly at the full resolution $\beta^3$ without hierarchical initialization results in poor motion reconstruction [S1].

## Operating characteristics
*   **Memory Complexity:** The memory usage is dominated by the Motion Coefficient Tensor $C$, which scales as $O(\beta^3 \cdot k \cdot 3)$. This is significantly lower than storing explicit displacement fields for every time step. The method allows optimizing on low-resolution measurements (e.g., $80^3$) and upsampling to high resolutions (e.g., $256^3$) post-optimization, bypassing substantial GPU memory requirements [S1].
*   **Time Complexity:** The warp field computation is $O(\beta^3 \cdot k)$ per time point. Rendering involves grid sampling, which is typically $O(\beta^3)$. The hierarchical optimization adds overhead but improves convergence.
*   **Strengths:**
    *   **Arbitrary Temporal Resolution:** The polynomial representation allows evaluation at any time $t \in [0, 1]$, enabling arbitrary frame rate upsampling (e.g., from 10 to 90 frames) [S1].
    *   **Memory Efficiency:** Parametric representation reduces memory footprint compared to storing discrete time steps.
    *   **Differentiability:** The method is fully differentiable, allowing gradient-based optimization of the coefficients [S1].
*   **Limitations:**
    *   **Interpolation Constraint:** Trilinear interpolation for upsampling the field itself may not capture complex motion details as well as learned upsamplers [S1].
    *   **Polynomial Order:** High-order polynomials may oscillate (Runge's phenomenon) if not regularized, though $k=5$ is found sufficient for typical dynamic scenes [S1].
*   **When to Use:** Suitable for dynamic CT reconstruction from sparse/limited views where temporal continuity is expected and memory is a constraint.
*   **When Not to Use:** When motion is highly discontinuous or non-polynomial in nature, or when extremely high-fidelity temporal upsampling is required without the interpolation limitations.

## Background
The Polynomial Motion Field was introduced in the context of dynamic CT reconstruction from limited views. It leverages implicit neural representations and parametric motion fields to warp a template volume to different time values. The method was proposed to address the challenges of reconstructing objects from limited views, which is traditionally more challenging than from sparse views. It assumes only that the scene is sparse view sampled, without requiring specialized sampling schemes or phase information [S1]. The hierarchical coarse-to-fine estimation strategy was introduced to overcome poor motion reconstruction at full resolution. The approach builds on differentiable grid sampling functions [S1].

## Sources
[S1] Dynamic CT Reconstruction from Limited Views with Implicit Neural Representations and Parametric Motion Fields, https://doi.org/10.1109/iccv48922.2021.00226, fetched 2026-10-26
