# Local Eikonal Loss

Also called: Lle

## Overview
Local Eikonal Loss (Lle) is a regularization term that enforces the Eikonal equation ($\|\nabla s\| = 1$) specifically at the centers of active voxels in a hierarchical grid, rather than on randomly sampled points. It serves as a sparse alternative to dense global Eikonal loss, reducing computational overhead while maintaining geometric consistency near the surface. It is distinguished from its sibling [Surface SDF Loss](14235-surface-sdf-loss.md) by targeting voxel centers rather than surface points, and from [SHINE-Mapping Loss Function](18802-shine-mapping-loss-function.md) by its specific application to hierarchical voxel grids using analytical gradients of trilinear interpolation.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of active voxels $\mathcal{V}$ in a hierarchical grid (parent and child cells). For each voxel $v$, the SDF values at its eight corners, denoted as $geo_v = \{f_{i,j,k} \mid i,j,k \in \{0,1\}\}$, and the voxel size $h_v$.
*   **Output:** A scalar loss value $L_{le}$ and its gradient with respect to the corner SDF values, used for backpropagation.

**Data Structures:**
*   **Voxel Center ($c_v$):** The geometric center of voxel $v$.
*   **Corner Values ($f_{i,j,k}$):** The SDF values predicted by the network at the eight corners of the voxel. These are accessed via trilinear interpolation if the network is continuous, or directly if stored in a sparse voxel structure. In the context of Lle, these are treated as the immediate neighbors required for the analytical gradient.

**Algorithm Steps:**
1.  **Eligibility Check:** For each active voxel $v$, determine if it is eligible for Lle computation. Eligibility typically depends on the resolution level $L$ (e.g., $L > 9$) and the training iteration $\tau$.
2.  **Stochastic Sampling:** To manage computational cost, evaluate Lle on each eligible cell with a probability $p = 0.5$. If the voxel is not selected, skip it.
3.  **Gradient Calculation:** Compute the gradient $\nabla_x f(c_v)$ analytically by differentiating the trilinear interpolation function at the voxel center. The gradient components are computed by averaging the differences along the edges connected to the center.
    *   The x-component is calculated as:
        $$ \frac{\partial f}{\partial x} \approx \frac{1}{4h_v} \sum_{j,k \in \{0,1\}} (f_{1,j,k} - f_{0,j,k}) $$
    *   The y and z components are computed similarly by summing differences along the respective axes.
4.  **Loss Computation:** Compute the squared deviation of the gradient magnitude from 1 for the selected voxels:
    $$ L_{le} = \frac{1}{N_{vox}} \sum_{v \in \mathcal{V}_{selected}} (\|\nabla_x f(c_v)\|_2 - 1)^2 $$
    where $N_{vox}$ is the total number of voxels considered in the sum (or the number of eligible voxels, depending on normalization convention; the source uses $N_{vox}$ in the denominator of the final formula).
5.  **Weighting:** Multiply the loss by the hyperparameter $\lambda_{le}$. If stochastic sampling was used, scale the weight by $1/p$ (i.e., 2.0) to ensure the expected contribution matches $\lambda_{le}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def local_eikonal_loss(voxels, lambda_le, probability=0.5):
    """
    Compute Local Eikonal Loss for a set of active voxels.
    
    Args:
        voxels: List of active voxel objects. Each voxel has:
            - center: (x, y, z)
            - size: h_v (float)
            - corners: list of 8 SDF values [f_000, f_001, ..., f_111]
        lambda_le: Weight for the loss term.
        probability: Probability of evaluating a voxel (default 0.5).
        
    Returns:
        loss: Scalar loss value.
    """
    total_loss = 0.0
    count = 0
    
    # Scale weight to compensate for stochastic sampling
    scaled_lambda = lambda_le / probability
    
    for v in voxels:
        # Stochastic sampling
        if random() > probability:
            continue
            
        h_v = v.size
        f = v.corners # List of 8 floats
        
        # Compute gradient components analytically
        # Indices: 0=000, 1=001, 2=010, 3=011, 4=100, 5=101, 6=110, 7=111
        # x-component: average difference between x=1 and x=0 corners
        # Corners with x=0: 0,1,2,3; Corners with x=1: 4,5,6,7
        df_dx = (f[4] + f[5] + f[6] + f[7] - (f[0] + f[1] + f[2] + f[3])) / (4 * h_v)
        
        # y-component: average difference between y=1 and y=0 corners
        # Corners with y=0: 0,1,4,5; Corners with y=1: 2,3,6,7
        df_dy = (f[2] + f[3] + f[6] + f[7] - (f[0] + f[1] + f[4] + f[5])) / (4 * h_v)
        
        # z-component: average difference between z=1 and z=0 corners
        # Corners with z=0: 0,2,4,6; Corners with z=1: 1,3,5,7
        df_dz = (f[1] + f[3] + f[5] + f[7] - (f[0] + f[2] + f[4] + f[6])) / (4 * h_v)
        
        # Compute gradient magnitude
        grad_mag = sqrt(df_dx**2 + df_dy**2 + df_dz**2)
        
        # Compute Eikonal error
        error = (grad_mag - 1.0)**2
        
        total_loss += error
        count += 1
        
    if count == 0:
        return 0.0
        
    # Average over selected voxels
    avg_loss = total_loss / count
    
    # Apply scaled weight
    return scaled_lambda * avg_loss
```

**Parameters:**
*   $\lambda_{le}$: Weight for the Local Eikonal loss. Typical value: $10^{-11}$ for DTU scenes [S1, unofficial].
*   Activation Window: The loss is active only during specific iteration ranges. For DTU: $6,000 \le \tau \le 8,000$. For Tanks-and-Temples: $6,000 \le \tau \le 10,000$ [S1, unofficial].
*   Resolution Threshold: Typically applied only for resolution levels $L > 9$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Voxels:** If no voxels are selected due to stochastic sampling, the loss is zero.
*   **Precision:** Analytical gradients avoid numerical differentiation errors but rely on accurate corner values. If corner values are noisy, the gradient estimate may be inaccurate.
*   **Boundary Conditions:** The method assumes all eight corners are available. For voxels at the boundary of the active region, missing neighbors must be handled (e.g., by clamping or padding), though the source does not specify this detail [G].

## Subtopics

### Global Eikonal Loss
A dense Eikonal term evaluated on all dense grid cells to enforce unit gradient magnitude. Unlike Lle, it does not use stochastic sampling or voxel-center-specific analytical gradients. The base weight is $\lambda_e = 10^{-8}$, decaying by a factor of 0.25 every 2,000 iterations. It is applied only up to resolution level 29 (iteration $\tau = 6,000$). The computational cost scales significantly, with the number of samples roughly quadrupling for each increase in dense resolution level [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_{vox})$ per iteration, where $N_{vox}$ is the number of active voxels. Due to stochastic sampling (50% probability), the effective complexity is $O(0.5 N_{vox})$.
*   **Memory Complexity:** $O(1)$ auxiliary memory per voxel, as it only requires the 8 corner SDF values and the voxel size.
*   **Performance:** Significantly faster than Global Eikonal Loss, which scales with the dense grid resolution. It provides a balance between regularization strength and computational cost.
*   **Strengths:** Efficient, stable, and effective for hierarchical voxel representations.
*   **Limitations:** Only enforces the Eikonal constraint at voxel centers, potentially missing errors in between. Requires careful tuning of the activation window and weight.
*   **When to Use:** When using hierarchical voxel grids (e.g., SVRecon) and computational resources are limited. It is particularly useful in the fine-tuning stages of training.
*   **When Not to Use:** When high-fidelity gradient information is required everywhere in the volume, or when using non-hierarchical representations.

## Background
Local Eikonal Loss was introduced in the context of sparse voxel rasterization for surface reconstruction, specifically in the SVRecon framework [S1, unofficial]. It addresses the computational bottleneck of dense Eikonal regularization by restricting the constraint to voxel centers and leveraging analytical gradients. The method is part of a broader family of regularization techniques for neural implicit surfaces, building on the principles of [Regularization and Gradient Losses](17793-regularization-and-gradient-losses.md).

## Sources
[S1] SVRecon: Sparse Voxel Rasterization for Surface Reconstruction, https://doi.org/10.48550/arxiv.2511.17364, fetched 2026-10-26 (unofficial)
