# Depth-guided Voxel Pre-pruning

Also called: Depth prior voxel pruning, Free space pruning via depth prior

## Overview
Depth-guided voxel pre-pruning is a preprocessing strategy that removes voxels located in free space using a dense depth prior before the optimization phase begins [S1]. It reduces the number of samples in free space and the parameters to be optimized, thereby accelerating training while implicitly regularizing scene geometry without requiring explicit depth supervision [S1]. Unlike methods that rely on explicit depth supervision using sparse Structure-from-Motion (SfM) or imperfect monocular depth estimators, this method leverages dense, multi-view consistent depth priors from 3D vision foundation models to prune the voxel grid directly [S1].

## How it works
**Inputs:**
*   Sparse multi-view images $I = \{I_n\}_{n=1}^S$.
*   Camera intrinsics $K = \{K_n\}_{n=1}^S$ and extrinsics $P = \{P_n\}_{n=1}^S$.
*   Voxel grid $V$ (either density grid $V_D$ or spherical harmonics grid $V_{SH}$) with resolution $[N_x, N_y, N_z]$.
*   Scene bounding box defined by corners $(x_a, y_a, z_a)$ and $(x_b, y_b, z_b)$.
*   Ray marching step size $s$.
*   Threshold $\tau$ for confidence filtering.

**Outputs:**
*   Pruned voxel grid $V$.

**Data Structures:**
*   **Grid Mask ($M_g$):** A boolean array of shape $[N_x, N_y, N_z]$. `True` indicates the voxel is active (to be kept/optimized); `False` indicates the voxel is pruned (free space). Initialized to all `True`.
*   **Ray Sets:**
    *   World frame: Origin set $O_w$, Direction set $Q_w$.
    *   Grid frame: Origin set $O_g$, Direction set $Q_g$.
    *   Distance parameter $T$: Scalar values representing distance along the ray.
*   **Depth Data:**
    *   Depth maps $D = \{D_n\}_{n=1}^S$.
    *   Confidence maps $E = \{E_n\}_{n=1}^S$.
    *   Binary masks $M = \{M_n\}_{n=1}^S$.

**Algorithm Steps:**

1.  **Depth Acquisition and Filtering:**
    Input images $I$ into a 3D vision foundation model (e.g., VGGT) to obtain depth maps $D$ and confidence maps $E$ [S1]. Filter low-confidence depths using threshold $\tau$ to generate binary masks $M = \{M_n\}_{n=1}^S$ [S1]:
    $$
    m = \begin{cases} \text{True} & e \geq \tau \\ \text{False} & e < \tau \end{cases}, \quad \text{where } m \in M_n, e \in E_n \quad (1)
    $$
    *Note: Depths predicted by the foundation model are aligned with sparse depths obtained by triangulating corresponding pixels using least squares, or by aligning predicted poses with known poses [S1].*

2.  **Ray Generation:**
    Generate a set of rays $R_w = \{R_{w_n}\}_{n=1}^S$ in the world frame using camera intrinsics $K$ and extrinsics $P$. This consists of origin set $O_w$ and direction set $Q_w$ [S1].

3.  **Masking and Aggregation:**
    Filter the depth maps and rays using the binary masks $M$ [S1]:
    $$
    D \leftarrow \bigcup_{n=1}^S D_n[M_n], \quad R_w \leftarrow \bigcup_{n=1}^S R_{w_n}[M_n] \quad (2)
    $$
    Here, $D_n[M_n]$ denotes the depth values where the mask is true.

4.  **Coordinate Transformation:**
    Transform the filtered depth $D$ and rays $R_w$ into the grid frame to obtain $D_g$ and $R_g$ (origin $O_g$, direction $Q_g$) based on the grid resolution and scene bounding box [S1].

5.  **Ray-Grid Intersection:**
    Compute the near-bound set $T_n$ and far-bound set $T_f$ of rays $R_g$ intersected with the voxel grid $V$ [S1].
    Filter out rays that do not intersect the grid (where $T_n > T_f$) [S1]:
    $$
    R_g \leftarrow R_g[T_n \leq T_f], \quad D_g \leftarrow D_g[T_n \leq T_f]
    $$
    $$
    T_n \leftarrow T_n[T_n \leq T_f], \quad T_f \leftarrow T_f[T_n \leq T_f] \quad (3)
    $$

6.  **Initialization:**
    Initialize the grid mask $M_g$ with resolution $[N_x, N_y, N_z]$ where all values are `True` [S1].
    Initialize a ray mask $M_r$ with the same shape as $R_g$ where all values are `True` [S1].
    Initialize the marching distance parameter $T \leftarrow T_n$ [S1].

7.  **Iterative Pruning:**
    While there exist elements in $M_r$ that are `True`:
    a. Compute positions $P$ where all active rays reach during marching forward:
       $$
       P = O_g + T \times Q_g \quad (4)
       $$
    b. Determine the voxel indices $I, J, K$ occupied by positions $P$ [S1].
    c. Mark these voxels as occupied (pruned) in the grid mask:
       $$
       M_g[I, J, K] = \text{False} \quad (5)
       $$
    d. Advance the marching distance:
       $$
       T \leftarrow T + s \quad (6)
       $$
    e. Mark rays that have reached or passed the depth prior as inactive:
       $$
       M_r[T \geq D_g] = \text{False} \quad (7)
       $$
    f. Filter active rays and their associated data using $M_r$:
       $$
       O_g \leftarrow O_g[M_r], \quad Q_g \leftarrow Q_g[M_r]
       $$
       $$
       T \leftarrow T[M_r], \quad D_g \leftarrow D_g[M_r], \quad M_r \leftarrow M_r[M_r] \quad (8)
       $$

8.  **Finalization:**
    Apply the grid mask to the voxel grid:
    $$
    V \leftarrow V[M_g] \quad (9)
    $$
    Return the pruned voxel grid $V$ [S1].

**Pitfalls and Edge Cases:**
*   **Overpruning:** If depth priors are larger than the ground truth, scene voxels may be incorrectly removed [S1]. This is mitigated by applying voxel dilation after pruning to restore some voxels potentially containing scene context [S1].
*   **Discreteness Error:** Even with high-accuracy depths, voxels occupied by the scene surface might be pruned due to the discreteness of the voxel grid representation [S1].
*   **Confidence Threshold:** The threshold $\tau$ is manually set. Improper selection may retain noisy depths or discard valid ones [S1].

## Operating characteristics
*   **Time Complexity:** The pruning process is highly efficient, completing in a matter of seconds using parallel computing [S1]. The iterative marching step depends on the number of active rays and the depth of the scene relative to the step size $s$.
*   **Memory Complexity:** Requires storage for the grid mask $M_g$ ($N_x N_y N_z$), ray sets, and depth maps. The pruning reduces the final memory footprint of the voxel grid $V$.
*   **Strengths:** Avoids the computational burden and imperfect data issues of explicit depth supervision [S1]. It reduces samples in free space and parameters to be optimized, accelerating training [S1]. It implicitly imposes geometric constraints without extra computational cost during training [S1].
*   **Limitations:** Relies on the accuracy and density of the depth prior. Overpruning can occur if the depth prior is inaccurate or too large [S1]. Discreteness errors may lead to slight geometry loss at surfaces [S1].
*   **When to use:** When using explicit voxel radiance field representations (e.g., Plenoxels) and sparse multi-view inputs, particularly when training efficiency is critical and a 3D vision foundation model is available to provide dense depth priors [S1].
*   **When not to use:** When high-precision geometry is required without post-processing dilation, or when depth priors are known to be significantly inaccurate or sparse [S1].

## Background
Depth-guided voxel pre-pruning was proposed in the context of **F2Plenoxels**, a framework for fast few-shot view synthesis based on explicit voxel radiance field representations [S1]. The method addresses the challenges of incorporating depth supervision, which increases training overhead and relies on imperfect data from SfM, image matchers, or monocular depth estimators [S1]. Instead of explicit depth supervision, F2Plenoxels uses depth priors from the 3D vision foundation model **VGGT** (Wang et al. 2025) to prune voxels before training [S1]. VGGT was chosen because it is completely feedforward, supports a variable number of input views, and balances density and multi-view consistency [S1]. The method builds on the concept of empty space filtering but specifically targets voxel grids using dense depth priors to implicitly regularize geometry [S1].

## Sources
[S1] F2Plenoxels: fast voxel radiance fields without neural networks for few-shot view synthesis, https://doi.org/10.1007/s44443-026-00491-2, fetched 2026-10-26
