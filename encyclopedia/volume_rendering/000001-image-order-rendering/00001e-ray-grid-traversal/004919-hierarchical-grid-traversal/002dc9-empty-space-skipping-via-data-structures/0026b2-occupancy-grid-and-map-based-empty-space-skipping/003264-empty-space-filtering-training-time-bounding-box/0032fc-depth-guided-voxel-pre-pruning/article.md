# Depth-guided Voxel Pre-pruning

Also called: Depth prior voxel pruning, Free space pruning via depth prior

## Overview
Depth-guided voxel pre-pruning is a preprocessing strategy that removes voxels located in free space using a dense depth prior before the optimization phase begins [S1]. It reduces the number of samples in free space and the parameters to be optimized, thereby accelerating training while implicitly regularizing scene geometry without requiring explicit depth supervision [S1]. Unlike methods that rely on explicit depth supervision using sparse Structure-from-Motion (SfM) or imperfect monocular depth estimators, this method leverages dense, multi-view consistent depth priors from 3D vision foundation models to prune the voxel grid directly [S1].

## How it works
**Inputs:**
*   Sparse multi-view images $I = \{I_n\}_{n=1}^S$ [S1].
*   Camera intrinsics $K = \{K_n\}_{n=1}^S$ and extrinsics $P = \{P_n\}_{n=1}^S$ [S1].
*   Voxel grid $V$ (either density grid $V_D$ or spherical harmonics grid $V_{SH}$) with resolution $[N_x, N_y, N_z]$ [S1].
*   Scene bounding box defined by corners $(x_a, y_a, z_a)$ and $(x_b, y_b, z_b)$ [S1].
*   Ray marching step size $s$ [S1].
*   Threshold $\tau$ for confidence filtering [S1].

**Outputs:**
*   Pruned voxel grid $V$ [S1].

**Data Structures:**
*   **Grid Mask ($M_g$):** A boolean array of shape $[N_x, N_y, N_z]$. `True` indicates the voxel is active (to be kept/optimized); `False` indicates the voxel is pruned (free space) [S1]. Initialized to all `True` [S1].
*   **Ray Sets:**
    *   World frame: Origin set $O_w$, Direction set $Q_w$ [S1].
    *   Grid frame: Origin set $O_g$, Direction set $Q_g$ [S1].
    *   Distance parameter $T$: Array of scalar values representing distance along each ray [S1].
*   **Depth Data:**
    *   Depth maps $D = \{D_n\}_{n=1}^S$ [S1].
    *   Confidence maps $E = \{E_n\}_{n=1}^S$ [S1].
    *   Binary masks $M = \{M_n\}_{n=1}^S$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# 1. Depth Acquisition and Filtering
D, E = VGGT_forward(I)  # Obtain depths and confidences from foundation model [S1]
M = [np.array(e >= tau, dtype=bool) for e in E]  # Filter low-confidence depths [S1]

# 2. Ray Generation
O_w, Q_w = generate_rays(K, P)  # Generate rays in world frame [S1]

# 3. Masking and Aggregation
D_filtered = [Dn[Mn] for Dn, Mn in zip(D, M)]
R_w_filtered = [Rw[Mn] for Rw, Mn in zip([O_w, Q_w], M)]  # Aggregate valid rays/depths [S1]

# 4. Coordinate Transformation
O_g, Q_g, D_g = transform_to_grid_frame(R_w_filtered, D_filtered, bounding_box, resolution) [S1]

# 5. Ray-Grid Intersection
T_n, T_f = compute_ray_grid_intersection(O_g, Q_g, bounding_box)  # Near and far bounds [S1]
valid = T_n <= T_f  # Filter non-intersecting rays [S1]
O_g, Q_g, D_g, T_n, T_f = [arr[valid] for arr in [O_g, Q_g, D_g, T_n, T_f]]

# 6. Initialization
M_g = np.ones((Nx, Ny, Nz), dtype=bool)  # Grid mask [S1]
M_r = np.ones(len(O_g), dtype=bool)      # Ray mask [S1]
T = T_n.copy()                           # Current marching distance [S1]

# 7. Iterative Pruning
while np.any(M_r):
    P = O_g + T[:, None, None] * Q_g    # Compute ray positions [S1]
    I, J, K = voxel_indices(P)          # Determine occupied voxel indices [S1]
    M_g[I, J, K] = False                # Mark occupied voxels as pruned [S1]
    T = T + s                           # Advance marching distance [S1]
    M_r[T >= D_g] = False               # Deactivate rays past depth prior [S1]
    # Filter active rays and data
    O_g, Q_g, D_g, T, M_r = [arr[M_r] for arr in [O_g, Q_g, D_g, T, M_r]]

# 8. Finalization
V = V[M_g]  # Apply grid mask to voxel grid [S1]
return V
```

**Parameters:**
*   $\tau$: Manually set threshold for filtering low-confidence depths. If confidence $e < \tau$, the depth is discarded [S1].
*   $s$: Ray marching step size used in the iterative voxel marking process [S1].

**Pitfalls and Edge Cases:**
*   **Overpruning:** If depth priors are larger than ground truth, scene voxels may be incorrectly removed [S1].
*   **Discreteness Error:** Voxels occupied by the scene surface might be pruned even with high-accuracy depths due to voxel grid discreteness [S1].
*   **Confidence Threshold Selection:** The threshold $\tau$ is manually set; improper selection may retain noisy depths or discard valid ones [S1].

## Subtopics

### Voxel Dilation
A morphological operation applied to a pruned voxel grid to restore voxels that were incorrectly removed due to artifacts in the depth prior [S1]. It is executed immediately after the depth-guided pre-pruning step to alleviate overpruning resulting from inevitable artifacts existing in the depth prior [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The method is described as highly efficient, completing the necessary geometry initialization in a matter of seconds using parallel computing [S1].
*   **Memory Complexity:** Requires storage for the grid mask $M_g$ ($N_x N_y N_z$), ray sets, and depth maps [S1]. The pruning reduces the final memory footprint of the voxel grid $V$ [S1].
*   **Strengths:** Avoids the computational burden and imperfect data issues of explicit depth supervision [S1]. It reduces samples in free space and parameters to be optimized, accelerating training [S1]. It implicitly imposes geometric constraints without introducing extra computational cost during training [S1].
*   **Limitations:** Relies on the accuracy and density of the depth prior [S1]. Overpruning can occur if the depth prior is inaccurate or too large [S1]. Discreteness errors may lead to slight geometry loss at surfaces [S1].
*   **When to use:** When using explicit voxel radiance field representations (e.g., Plenoxels) and sparse multi-view inputs, particularly when training efficiency is critical and a 3D vision foundation model is available to provide dense depth priors [S1].
*   **When not to use:** When high-precision geometry is required without post-processing dilation, or when depth priors are known to be significantly inaccurate or sparse [S1].

## Background
Depth-guided voxel pre-pruning was proposed in the context of **F2Plenoxels**, a framework for fast few-shot view synthesis based on explicit voxel radiance field representations [S1]. The method addresses the challenges of incorporating depth supervision, which increases training overhead and relies on imperfect data from SfM, image matchers, or monocular depth estimators [S1]. Instead of explicit depth supervision, F2Plenoxels uses depth priors from the 3D vision foundation model **VGGT** (Wang et al. 2025) to prune voxels before training [S1]. VGGT was chosen because it is completely feedforward, supports a variable number of input views, and balances density and multi-view consistency [S1]. The method builds on the concept of empty space filtering but specifically targets voxel grids using dense depth priors to implicitly regularize geometry [S1].

## Sources
[S1] F2Plenoxels: fast voxel radiance fields without neural networks for few-shot view synthesis, https://doi.org/10.1007/s44443-026-00491-2, fetched 2026-10-26
