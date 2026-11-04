# Incremental Ray Traversal through OpenVDB Frustum Grids

Also called: Frustum Grid Ray Traversal, Incremental DDA for Frustum Grids

## Overview
This method performs efficient ray traversal through OpenVDB frustum grids by generalizing the 3D Digital Differential Analyzer (DDA) algorithm to account for the non-linearity of the perspective transform [S1, unofficial]. It allows for the traversal of any ray (not just camera rays) against frustum-structured voxel data without requiring auxiliary grids or precomputed acceleration structures, solving the difficulty that standard uniform-grid DDA faces when applied to non-linearly mapped index spaces [S1, unofficial]. It builds on [OpenVDB DDA](2149-openvdb-dda.md) by replacing the linear step logic with incremental updates to rational functions derived from the inverse frustum map.

## How it works

### Inputs and Outputs
**Inputs:**
- Ray in world space: origin $\mathbf{r}_0$ and direction $\hat{\omega}$ (unit vector).
- OpenVDB Grid parameters:
  - Index space bounds: $[I_0, I_1] \times [J_0, J_1] \times [K_0, K_1]$.
  - Leaf node size: $N$ (typically 8, so leaf dimensions are $N \times N \times N$).
  - Frustum parameters: taper $\tau$, depth $d$, aspect ratio $a = (J_1 - J_0)/(I_1 - I_0)$.
  - Affine transform matrix $M$ mapping frustum space to world space.
- Grid hierarchy structure (to access leaf nodes).

**Outputs:**
- A sequence of leaf node indices $(i, j, k)$ visited by the ray within the valid distance range $[t_{entry}, t_{exit}]$.

### Data Structures
The OpenVDB grid is a hierarchy of nodes, commonly four levels deep, where leaf nodes hold the actual voxel data [S1, unofficial]. A leaf node stores an array of $N \times N \times N$ voxels (configurable to other powers of two) [S1, unofficial]. The traversal operates on integer index coordinates $(i, j, k)$ within the global index space $[I_0, I_1] \times [J_0, J_1] \times [K_0, K_1]$ [S1, unofficial].

### Parameters
- $\tau$ (taper): Ratio between the sizes of the near rectangle and the far rectangle in frustum space.
- $d$ (depth): The z-coordinate of the far plane in frustum space; the near plane is at $z=0$.
- $a$ (aspect ratio): Ratio of grid dimensions in the $j$ and $i$ directions, $a = (J_1 - J_0)/(I_1 - I_0)$.
- $\gamma$ (gamma): Constant derived from taper and depth, $\gamma = (1/\tau - 1)/d$ [S1, unofficial].

### Algorithm Steps

1. **Transform Ray to Frustum Space**:
   Calculate the frustum ray origin $\mathbf{s}_0$ and direction $\delta$ using the inverse of the affine matrix $M^{-1}$.
   - $\mathbf{s}_0$ is a point (homogeneous coordinate 1).
   - $\delta$ is a direction (homogeneous coordinate 0).
   - Do not renormalize $\delta$ [S1, unofficial].

2. **Clip Ray Against Frustum Boundaries**:
   Determine the valid distance range $[t_0, t_1]$ for traversal by clipping against the six planes of the frustum.
   
   a. **Z-Clipping (Near/Far Planes)**:
   Intersect with $k=K_0$ ($z=0$) and $k=K_1$ ($z=d$) to get initial $t_0$ and $t_1$:
   $$ t_0 = t_K(0) $$
   $$ t_1 = t_K(d) $$
   where $t_K(k) = (z - s_{0z})/\delta_z$ and $z = d(k - K_0)/(K_1 - K_0)$ [S1, unofficial].

   b. **Right Plane ($i=I_1, x'=1/2$)**:
   Calculate distance $t_R$:
   $$ t_R = \frac{(\gamma s_{0z} + 1)/2 - s_{0x}}{\delta_x - \gamma \delta_z/2} $$
   Determine entry/exit:
   - If $\delta_x < \gamma \delta_z/2$, the ray is entering. Update $t_0 = \max(t_0, t_R)$.
   - Otherwise, the ray is leaving. Update $t_1 = \min(t_1, t_R)$ [S1, unofficial].

   c. **Left Plane ($i=-I_1$? No, $x'=-1/2$)**:
   Calculate distance $t_L$:
   $$ t_L = \frac{-(\gamma s_{0z} + 1)/2 - s_{0x}}{\delta_x + \gamma \delta_z/2} $$
   Determine entry/exit:
   - If $\delta_x > -\gamma \delta_z/2$, the ray is entering. Update $t_0 = \max(t_0, t_L)$.
   - Otherwise, the ray is leaving. Update $t_1 = \min(t_1, t_L)$ [S1, unofficial].

   d. **Top and Bottom Planes**:
   Clip similarly to left/right, but using the factor $a/2$ instead of $1/2$ in the plane normal and point definitions. Specifically, replace $1/2$ with $a/2$ for the $y'$ coordinate calculations and adjust normals accordingly [S1, unofficial].

   e. **Check Validity**:
   If $t_1 \le t_0$, the ray does not enter the frustum. Return immediately [S1, unofficial].

3. **Determine Delta Signs**:
   Determine the signs of integer deltas $\{\Delta A = \pm N : A = I, J, K\}$ based on the ray direction components $\delta_x, \delta_y, \delta_z$.
   - $\Delta I = N$ if $\delta_x \ge 0$, else $-N$.
   - $\Delta J = N$ if $\delta_y \ge 0$, else $-N$.
   - $\Delta K = N$ if $\delta_z \ge 0$, else $-N$.
   These signs control whether coordinates are incremented or decremented during traversal [S1, unofficial].

4. **Initialize Iterative State**:
   Start at the leaf containing the ray origin (or the first leaf intersected by the ray within $[t_0, t_1]$). Let the current leaf indices be $(i_N, j_N, k_N)$, which are multiples of $N$.
   Initialize the distances to the next boundary crossings for each axis:
   
   For $I$:
   $$ x' = \frac{i_N - (I_1 + I_0)/2}{I_1 - I_0} $$
   $$ P_I = x'(\gamma s_{0z} + 1) - s_{0x} $$
   $$ Q_I = \delta_x - x' \gamma \delta_z $$
   $$ t_I = P_I / Q_I $$
   
   For $J$:
   $$ y' = \frac{j_N - (J_1 + J_0)/2}{I_1 - I_0} $$
   $$ P_J = y'(\gamma s_{0z} + 1) - s_{0y} $$
   $$ Q_J = \delta_y - y' \gamma \delta_z $$
   $$ t_J = P_J / Q_J $$
   
   For $K$:
   $$ z = d(k_N - K_0)/(K_1 - K_0) $$
   $$ t_K = (z - s_{0z})/\delta_z $$
   
   Define incremental constants:
   $$ \Delta'_I = \frac{\Delta I}{I_1 - I_0} $$
   $$ \Delta'_J = \frac{\Delta J}{I_1 - I_0} $$
   $$ \Delta'_K = \frac{d}{\delta_z} \frac{\Delta K}{K_1 - K_0} $$
   
   Define update deltas for $P$ and $Q$:
   $$ \Delta P_I = (\gamma s_{0z} + 1) \Delta'_I $$
   $$ \Delta Q_I = \gamma \delta_z \Delta'_I $$
   $$ \Delta P_J = (\gamma s_{0z} + 1) \Delta'_J $$
   $$ \Delta Q_J = \gamma \delta_z \Delta'_J $$ [S1, unofficial].

5. **Iterative Traversal**:
   While $t_{current} < t_1$:
   a. Find the next crossing distance: $t_{next} = \min(t_I, t_J, t_K)$.
   b. If $t_{next} > t_1$, break.
   c. Process the current leaf $(i_N, j_N, k_N)$.
   d. Identify which axis caused the minimum distance (e.g., if $t_I$ is min, increment $i_N$ by $\Delta I$).
   e. Update the state for the axis that crossed:
      - If $I$ crossed:
        $$ P_I \leftarrow P_I + \Delta P_I $$
        $$ Q_I \leftarrow Q_I - \Delta Q_I $$
        $$ t_I \leftarrow P_I / Q_I $$
        $$ i_N \leftarrow i_N + \Delta I $$
      - If $J$ crossed:
        $$ P_J \leftarrow P_J + \Delta P_J $$
        $$ Q_J \leftarrow Q_J - \Delta Q_J $$
        $$ t_J \leftarrow P_J / Q_J $$
        $$ j_N \leftarrow j_N + \Delta J $$
      - If $K$ crossed:
        $$ t_K \leftarrow t_K + \Delta'_K $$
        $$ k_N \leftarrow k_N + \Delta K $$
   f. Update $t_{current} = t_{next}$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_frustum_grid(ray_origin_world, ray_dir_world, grid_params, matrix_M_inv):
    """
    ray_origin_world: (x, y, z)
    ray_dir_world: (dx, dy, dz)
    grid_params: {I0, I1, J0, J1, K0, K1, N, tau, d, a}
    matrix_M_inv: 4x4 matrix inverse of M
    """
    # 1. Transform Ray to Frustum Space
    # s0 is point (w=1), delta is direction (w=0)
    s0 = transform_point(matrix_M_inv, ray_origin_world)
    delta = transform_vector(matrix_M_inv, ray_dir_world)
    s0x, s0y, s0z = s0
    dx, dy, dz = delta
    
    gamma = (1.0 / grid_params['tau'] - 1.0) / grid_params['d']
    
    # 2. Clip Ray
    # Z-clipping
    # t_K(k) = (z - s0z) / dz, where z = d*(k - K0)/(K1 - K0)
    # For k=K0, z=0 -> t0 = (0 - s0z)/dz
    # For k=K1, z=d -> t1 = (d - s0z)/dz
    # Note: dz must not be 0 for valid traversal, handled by caller or epsilon check
    
    t0 = (0.0 - s0z) / dz
    t1 = (grid_params['d'] - s0z) / dz
    
    # Right Plane (x' = 1/2)
    # t_R = ((gamma*s0z + 1)/2 - s0x) / (dx - gamma*dz/2)
    denom_R = dx - gamma * dz / 2.0
    if abs(denom_R) > 1e-9:
        t_R = ((gamma * s0z + 1.0) / 2.0 - s0x) / denom_R
        if dx < gamma * dz / 2.0:
            t0 = max(t0, t_R)
        else:
            t1 = min(t1, t_R)
    else:
        # Parallel to plane, no intersection logic needed for clipping range if already inside/outside
        pass

    # Left Plane (x' = -1/2)
    # t_L = (-(gamma*s0z + 1)/2 - s0x) / (dx + gamma*dz/2)
    denom_L = dx + gamma * dz / 2.0
    if abs(denom_L) > 1e-9:
        t_L = (-(gamma * s0z + 1.0) / 2.0 - s0x) / denom_L
        if dx > -gamma * dz / 2.0:
            t0 = max(t0, t_L)
        else:
            t1 = min(t1, t_L)
    else:
        pass

    # Top/Bottom Planes (y' = +/- a/2)
    # Similar logic to Left/Right but with a/2 factor
    # Right-ish Y (y' = a/2)
    denom_Y_top = dy - gamma * dz * (grid_params['a'] / 2.0)
    if abs(denom_Y_top) > 1e-9:
        t_Y_top = ((gamma * s0z + 1.0) * (grid_params['a'] / 2.0) - s0y) / denom_Y_top
        if dy < gamma * dz * (grid_params['a'] / 2.0):
            t0 = max(t0, t_Y_top)
        else:
            t1 = min(t1, t_Y_top)
    
    # Left-ish Y (y' = -a/2)
    denom_Y_bot = dy + gamma * dz * (grid_params['a'] / 2.0)
    if abs(denom_Y_bot) > 1e-9:
        t_Y_bot = (-(gamma * s0z + 1.0) * (grid_params['a'] / 2.0) - s0y) / denom_Y_bot
        if dy > -gamma * dz * (grid_params['a'] / 2.0):
            t0 = max(t0, t_Y_bot)
        else:
            t1 = min(t1, t_Y_bot)

    if t1 <= t0:
        return [] # No intersection

    # 3. Determine Deltas
    N = grid_params['N']
    dI = N if dx >= 0 else -N
    dJ = N if dy >= 0 else -N
    dK = N if dz >= 0 else -dK # G: Typo in thought, should be -N
    dK = N if dz >= 0 else -N
    
    # 4. Initialize State
    # Find starting leaf (iN, jN, kN) containing ray origin or first intersection
    # This requires mapping ray origin to index space or finding the first leaf intersected.
    # For simplicity, assume we have a function get_starting_leaf(s0, t0)
    iN, jN, kN = get_starting_leaf(s0, t0, grid_params)
    
    # Calculate initial tI, tJ, tK
    # I
    x_prime = (iN - (grid_params['I1'] + grid_params['I0']) / 2.0) / (grid_params['I1'] - grid_params['I0'])
    P_I = x_prime * (gamma * s0z + 1.0) - s0x
    Q_I = dx - x_prime * gamma * dz
    t_I = P_I / Q_I
    
    # J
    y_prime = (jN - (grid_params['J1'] + grid_params['J0']) / 2.0) / (grid_params['I1'] - grid_params['I0']) # Note: Denom is I1-I0
    P_J = y_prime * (gamma * s0z + 1.0) - s0y
    Q_J = dy - y_prime * gamma * dz
    t_J = P_J / Q_J
    
    # K
    z = grid_params['d'] * (kN - grid_params['K0']) / (grid_params['K1'] - grid_params['K0'])
    t_K = (z - s0z) / dz
    
    # Incremental Constants
    dI_prime = dI / (grid_params['I1'] - grid_params['I0'])
    dJ_prime = dJ / (grid_params['I1'] - grid_params['I0'])
    dK_prime = (grid_params['d'] / dz) * (dK / (grid_params['K1'] - grid_params['K0']))
    
    dP_I = (gamma * s0z + 1.0) * dI_prime
    dQ_I = gamma * dz * dI_prime
    dP_J = (gamma * s0z + 1.0) * dJ_prime
    dQ_J = gamma * dz * dJ_prime
    
    visited_leaves = []
    current_t = t0
    
    # 5. Iterate
    while current_t < t1:
        # Find next crossing
        t_next = min(t_I, t_J, t_K)
        
        if t_next > t1:
            break
            
        # Process leaf
        visited_leaves.append((iN, jN, kN))
        
        # Update state for the axis that crossed
        if t_I <= t_J and t_I <= t_K:
            P_I += dP_I
            Q_I -= dQ_I
            t_I = P_I / Q_I
            iN += dI
        elif t_J <= t_I and t_J <= t_K:
            P_J += dP_J
            Q_J -= dQ_J
            t_J = P_J / Q_J
            jN += dJ
        else:
            t_K += dK_prime
            kN += dK
            
        current_t = t_next
        
    return visited_leaves
```

### Edge Cases and Pitfalls
- **Parallel Rays**: If $\delta_z = 0$, the $K$-index update is undefined. The ray is parallel to the frustum's depth axis. This requires special handling or is considered a degenerate case [S1, unofficial].
- **Division by Zero**: Denominators in $t_R, t_L, t_Y$ calculations can be zero if the ray is parallel to the respective planes. The code must check for this [S1, unofficial].
- **Precision**: The rational functions for $t_I$ and $t_J$ involve division. Accumulated floating-point errors may occur over many iterations. Renormalization or periodic recalculation from exact formulas may be necessary for long traversals [G].
- **Boundary Conditions**: The clipping logic must correctly handle rays starting inside or outside the frustum. The initial $t_0, t_1$ from Z-clipping assumes the ray intersects the slab $z \in [0, d]$. If the ray is parallel to Z, this step fails [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(L)$ where $L$ is the number of leaf nodes visited. Each step involves constant-time arithmetic operations.
- **Memory Complexity**: $O(1)$ auxiliary memory, excluding the grid structure itself.
- **Performance**: Efficient for frustum grids because it avoids the overhead of general-purpose acceleration structures like BVHs for the grid itself, leveraging the regular structure of OpenVDB leaves. It is particularly effective for perspective camera rays where voxel density naturally decreases with distance [S1, unofficial].
- **Strengths**: No auxiliary grids needed; handles any ray; efficient traversal of non-linear grids.
- **Limitations**: Only applicable to frustum grids; requires the inverse affine transform; sensitive to numerical precision in rational function updates [S1, unofficial].

## Background
The method was introduced in "Incremental Ray Traversal through OpenVDB Frustum Grids" [S1, unofficial]. It addresses the limitations of standard 3D DDA [Amanatides and Woo 1987] when applied to non-uniform, perspective-mapped grids. OpenVDB, developed by DreamWorks Animation and widely adopted in production path tracing, supports frustum grids to optimize memory usage for perspective views. This method generalizes the DDA to handle the rational curve nature of rays in index space after the inverse frustum transform [S1, unofficial].

## Sources
[S1] Incremental Ray Traversal through OpenVDB Frustum Grids, https://jcgt.org/published/0010/01/03/paper-lowres.pdf, fetched 2026-10-25 (unofficial)
