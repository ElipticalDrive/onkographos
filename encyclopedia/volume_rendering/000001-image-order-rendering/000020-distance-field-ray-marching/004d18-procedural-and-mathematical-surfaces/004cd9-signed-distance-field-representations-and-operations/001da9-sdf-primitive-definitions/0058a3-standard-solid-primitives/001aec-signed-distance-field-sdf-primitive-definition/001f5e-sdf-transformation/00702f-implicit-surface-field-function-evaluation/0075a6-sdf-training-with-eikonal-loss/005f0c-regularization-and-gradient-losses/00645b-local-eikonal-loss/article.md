# Local Eikonal Loss

Also called: Lle

## Overview
Local Eikonal Loss (Lle) is a regularization term for Signed Distance Field (SDF) representations that enforces the Eikonal equation ($||\nabla f|| = 1$) specifically at the centers of active voxels in a hierarchical grid [S1, unofficial]. Unlike global Eikonal loss which samples dense random coordinates, Lle operates on the underlying voxel structure, providing a sparse but geometrically aligned constraint that stabilizes SDF training near the surface [S1, unofficial]. It serves as a computationally efficient alternative to dense sampling by leveraging the analytical gradients of trilinear interpolation within the voxel hierarchy.

## How it works
**Inputs and Outputs**
*   **Input**: A hierarchical voxel grid (e.g., Octree) where each active voxel $v$ has a center $c_v$ and eight corner SDF values $geo_v = \{f_{i,j,k} \mid i,j,k \in \{0,1\}\}$.
*   **Output**: A scalar loss value $L_{le}$ representing the deviation of the gradient magnitude from unity at voxel centers.

**Data Structures**
*   **Voxel Grid**: Hierarchical structure (parent-child). Active voxels are those containing geometry or required for rendering.
*   **Corner Values**: For a voxel $v$ of size $h_v$, the SDF values at the 8 corners are retrieved via nearest-voxel search (NVS) and trilinear interpolation if necessary [S1, unofficial].

**Steps**
1.  **Eligibility Check**: Identify active voxels $v$ in the hierarchy.
2.  **Sampling Probability**: For each eligible voxel, evaluate the loss with probability $p=0.5$ [S1, unofficial]. If evaluated, scale the weight by $1/p = 2$ to maintain the expected contribution [S1, unofficial].
3.  **Gradient Computation**: Compute the spatial gradient $\nabla_x f(c_v)$ analytically using the corner SDF values. The gradient is derived from the trilinear interpolation function [S1, unofficial].
    *   The x-component $\frac{\partial f}{\partial x}$ is computed by averaging differences along the x-edges:
        $$ \frac{\partial f}{\partial x} \approx \frac{1}{4h_v} \sum_{j,k \in \{0,1\}} (f_{1,j,k} - f_{0,j,k}) $$
    *   Similarly, $\frac{\partial f}{\partial y}$ and $\frac{\partial f}{\partial z}$ are computed by averaging differences along the y and z edges respectively [S1, unofficial].
4.  **Loss Calculation**: Compute the squared error between the gradient magnitude and 1 for all evaluated voxels:
    $$ L_{le} = \frac{1}{N_{vox}} \sum_{i=1}^{N} (\|\nabla_x f(c_{v_i})\|_2 - 1)^2 $$
    where $N_{vox}$ is the number of voxels including all parents and children, and $N$ is the number of voxels actually evaluated in the current step [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def local_eikonal_loss(voxel_grid, lambda_le, iteration):
    """
    Computes the Local Eikonal Loss for a hierarchical voxel grid.
    
    Args:
        voxel_grid: Hierarchical structure containing active voxels.
        lambda_le: Weight for the loss term.
        iteration: Current training iteration.
        
    Returns:
        Scalar loss value.
    """
    total_loss = 0.0
    count = 0
    
    # Iterate over all active voxels (parents and children)
    for voxel in voxel_grid.active_voxels:
        # Probability sampling: evaluate with 50% chance
        if random() < 0.5:
            # Retrieve SDF values at the 8 corners of the voxel
            # f_{i,j,k} corresponds to corner (i, j, k) where i,j,k in {0,1}
            corners = get_corner_sdf_values(voxel)
            
            # Voxel size h_v
            h_v = voxel.size
            
            # Compute gradient components analytically from trilinear interpolation
            # x-component: average difference along x-edges
            df_dx = 0.0
            for j in [0, 1]:
                for k in [0, 1]:
                    # f_{1,j,k} is corner with x=1, f_{0,j,k} is corner with x=0
                    df_dx += (corners[1, j, k] - corners[0, j, k])
            df_dx = df_dx / (4.0 * h_v)
            
            # y-component: average difference along y-edges
            df_dy = 0.0
            for i in [0, 1]:
                for k in [0, 1]:
                    df_dy += (corners[i, 1, k] - corners[i, 0, k])
            df_dy = df_dy / (4.0 * h_v)
            
            # z-component: average difference along z-edges
            df_dz = 0.0
            for i in [0, 1]:
                for j in [0, 1]:
                    df_dz += (corners[i, j, 1] - corners[i, j, 0])
            df_dz = df_dz / (4.0 * h_v)
            
            # Compute gradient magnitude
            grad_mag = sqrt(df_dx**2 + df_dy**2 + df_dz**2)
            
            # Compute squared error from 1
            error = (grad_mag - 1.0)**2
            
            # Scale weight by 2 to compensate for 0.5 probability
            # Note: The formula divides by N_vox, so we accumulate the scaled error
            total_loss += 2.0 * error
            count += 1
            
    # Normalize by total number of voxels (including those not evaluated)
    # N_vox is the total count of active voxels in the hierarchy
    N_vox = voxel_grid.total_active_voxels_count
    
    if N_vox > 0:
        return (lambda_le / N_vox) * total_loss
    else:
        return 0.0
```

**Parameters**
*   **Weight ($\lambda_{le}$)**: Typically $10^{-11}$ for DTU scenes [S1, unofficial].
*   **Activation Window**:
    *   DTU scenes: Iterations $6,000 \le \tau \le 8,000$ [S1, unofficial].
    *   Tanks-and-Temples: Iterations $6,000 \le \tau \le 10,000$ [S1, unofficial].
*   **Probability**: 0.5 [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Precision**: The analytical gradient assumes linear interpolation between corners. If the SDF varies non-linearly within the voxel, the gradient approximation may be inaccurate [S1, unofficial].
*   **Boundaries**: Corner values must be retrieved correctly. If a corner lies outside the defined grid, appropriate padding or clamping is required [G].
*   **Degenerate Input**: If a voxel has zero size or invalid geometry, the gradient computation may fail. Guard against division by zero in $h_v$ [G].

## Subtopics

### [Global Eikonal Loss](19124-global-eikonal-loss.md)
Enforces the Eikonal equation across a dense grid of samples rather than sparse voxel centers. It is the dense counterpart to Lle, ensuring unit gradient magnitude throughout the volume but with higher computational cost sensitive to resolution changes.

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of active voxels $N_{vox}$, but reduced by the 0.5 sampling probability. Computation per voxel is constant (analytical gradient).
*   **Memory Complexity**: Minimal additional memory, only requiring storage of corner SDF values during the forward pass.
*   **Performance**: Efficient for hierarchical representations where the number of active voxels is much smaller than the total volume resolution.
*   **Strengths**: Geometrically aligned with the voxel structure; computationally cheaper than dense random sampling; stabilizes training in fine stages [S1, unofficial].
*   **Limitations**: Only constrains gradients at voxel centers, potentially missing errors in between; accuracy depends on voxel size relative to surface curvature [S1, unofficial].
*   **When to Use**: When using hierarchical voxel grids (e.g., Octrees) and computational resources are limited compared to dense sampling methods.
*   **When Not to Use**: When high-fidelity gradient information is required everywhere in the volume, not just at voxel centers.

## Background
Local Eikonal Loss was introduced in the context of video pose engines for 3D geometric perception, specifically in the Vipe framework [S1, unofficial]. It addresses the need for efficient regularization in sparse voxel rasterization for surface reconstruction, where dense sampling is prohibitive. The method leverages the analytical properties of trilinear interpolation to compute gradients without numerical differentiation, improving stability and speed [S1, unofficial].

## Sources
[S1] SVRecon: Sparse Voxel Rasterization for Surface Reconstruction, https://doi.org/10.48550/arxiv.2511.17364, fetched 2026-10-26 (unofficial)
