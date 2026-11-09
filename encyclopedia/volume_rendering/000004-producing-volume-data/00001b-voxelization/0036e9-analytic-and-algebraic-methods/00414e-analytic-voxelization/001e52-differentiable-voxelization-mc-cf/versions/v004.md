# Differentiable Voxelization (MC/CF)

Also called: Closed-form Voxelization, Convolution Filter Voxelization, Differentiable 3D Voxelization (Forward/Backward Mode), VoxelizeFunc

## Overview
Differentiable Voxelization (MC/CF) is a PyTorch autograd function that converts 2D polygons or 3D triangle meshes into a grid of winding numbers (occupancy), supporting both Marching Cubes (MC) and Convolution Filter (CF) methods with differentiable backward passes [S1, unofficial]. It builds on [Analytic Voxelization](10493-analytic-voxelization.md) by providing closed-form gradients with respect to mesh vertex positions, enabling gradient-based optimization of surface representations based on volumetric properties [S1, unofficial]. Unlike [Monte Carlo 2D Voxelization](1734-monte-carlo-2d-voxelization.md) which relies on stochastic sampling, this method uses deterministic closed-form integration for the CF variant and hybrid MC/closed-form for the MC variant, yielding sparse near-surface derivative signals [S1, unofficial].

## How it works
**Inputs:**
- `vertices`: Array of shape $(N, D)$ containing triangle/polygon vertices, type Float ($N$ is number of vertices, $D \in \{2, 3\}$) [S1, unofficial].
- `simplices`: Array of shape $(F, D)$ containing vertex indices for each face, type `uint32_t` ($F$ is number of faces) [S1, unofficial].
- `grid_shape`: Tuple of integers defining the output voxel grid dimensions (e.g., $(H, W)$ for 2D or $(D, H, W)$ for 3D) [S1, unofficial].
- `method`: String, either `'mc'` (Monte Carlo) or `'cf'` (Convolution Filter) [S1, unofficial].
- `dtype`: Data type, either `float32` or `float64` [S1, unofficial].
- `num_samples_per_voxel`: Number of samples per voxel for MC forward pass. Default: 16 [S1, unofficial].
- `num_samples_per_simplex`: Number of samples per simplex for MC backward pass. Default: 64 [S1, unofficial].
- `filter_radius`: Radius of the filter. Default: half voxel size. For CF method, it must equal half voxel size, otherwise it has no effect [S1, unofficial].

**Outputs:**
- `occupancy`: Output tensor of shape `grid_shape` and matching `dtype`, containing the winding number/occupancy values [S1, unofficial].

**Data Structures:**
The implementation uses standard PyTorch tensors. The internal C++ extension (`dvx`) manages memory for intermediate polygon splitting structures (vectors of barycentric coordinates) during the voxelization loop [S1, unofficial].

**Algorithm:**
The method dispatches to specific C++ kernels based on dimension (2D/3D), method (MC/CF), and differentiation mode (Forward/Backward/Primal) [S1, unofficial].

1. **Validation:** Validate inputs using `validate_common_voxelize_arguments` [S1, unofficial].
2. **Dispatch:** Select the primal function (`voxelize_mc_f32`, `voxelize_mc_f64`, `voxelize_cf_f32`, or `voxelize_cf_f64`) based on `method` and `dtype` [S1, unofficial].
3. **Initialization:** Initialize the output `occupancy` tensor as a zero tensor of shape `grid_shape` and `dtype` [S1, unofficial].
4. **Primal Execution:** Execute the primal voxelization function with `vertices`, `simplices`, and the `occupancy` output tensor [S1, unofficial].
5. **Save for Backward:** Save `vertices`, `simplices`, and `occupancy` for the backward pass [S1, unofficial].

**Backward Pass:**
In the backward pass, the method computes gradients of the occupancy loss with respect to the vertex positions [S1, unofficial].

1. **Dispatch:** Select the appropriate backward function (`voxelize_backward_mc_*` or `voxelize_backward_cf_*`) based on `method` and `dtype` [S1, unofficial].
2. **Initialization:** Initialize gradient tensor `d_vertices` as zeros with same shape and `dtype` as `vertices` [S1, unofficial].
3. **Iterate Faces:** For each face in `simplices`:
    a. Extract vertices $v_0, v_1, v_2$ from `vertices` [S1, unofficial].
    b. Compute edge vectors $e_1 = v_1 - v_0$, $e_2 = v_2 - v_0$ [S1, unofficial].
    c. Compute normal $n = \text{normalize}(\text{cross}(e_1, e_2))$ and area $A = 0.5 \times \|\text{cross}(e_1, e_2)\|$ [S1, unofficial].
    d. **Monte Carlo Integration (Backward):** Generate uniform samples $u, v$. If $u + v > 1$, reflect: $u = 1 - u; v = 1 - v$ [S1, unofficial].
    e. Compute normal velocities for backward pass:
       $$d_{xbn\_d\_v0} = n \times (1 - u - v)$$
       $$d_{xbn\_d\_v1} = n \times u$$
       $$d_{xbn\_d\_v2} = n \times v$$
    f. **Forward Mode Gradient Accumulation:** If in forward mode, compute $d_{xbn}$ by dotting the incoming vertex gradients with the normal velocities:
       $$d_{xbn} = \text{dot}(d_{v0}, d_{xbn\_d\_v0}) + \text{dot}(d_{v1}, d_{xbn\_d\_v1}) + \text{dot}(d_{v2}, d_{xbn\_d\_v2})$$
       [S1, unofficial].
    g. **Sample Point:** Compute sample point on triangle: $s = v_0 + u \cdot e_1 + v \cdot e_2$ [S1, unofficial].
    h. **Grid Mapping:** Convert $s$ to grid coordinates and determine the support box (bounding box of affected voxels) [S1, unofficial].
    i. **Voxel Iteration:** Iterate over neighbor voxels $(nx, ny, nz)$ in the support box:
       - Compute neighbor center $c = (nx + 0.5) \times \text{voxel\_size} - 1$ [S1, unofficial].
       - Compute filter weight $w$ based on distance from $s$ to $c$ and `filter_radius` [S1, unofficial].
       - If $w \neq 0$:
         - **Forward:** Update occupancy gradient:
           $$d_{\text{occupancy}}[idx] += d_{xbn} \times w \times A / \text{num\_samples\_per\_simplex}$$
           [S1, unofficial].
         - **Backward:** Compute differential weight:
           $$dw = d_{\text{occupancy}}[idx] \times w \times A / \text{num\_samples\_per\_simplex}$$
           Add to vertex gradients:
           $$d_{\text{vertices}}[3 \cdot v_0 + d] += d_{xbn\_d\_v0}[d] \times dw$$
           $$d_{\text{vertices}}[3 \cdot v_1 + d] += d_{xbn\_d\_v1}[d] \times dw$$
           $$d_{\text{vertices}}[3 \cdot v_2 + d] += d_{xbn\_d\_v2}[d] \times dw$$
           for all dimensions $d \in \{0, 1, 2\}$ [S1, unofficial].

**Pitfalls:**
- The CF method uses a different coordinate convention (width, height vs height, width), requiring a transpose of the output voxels (`voxels.transpose(0, -1)`), which regresses performance [S1, unofficial].
- The implementation is CPU-only; GPU tensors are automatically moved to CPU for processing and moved back, issuing a warning [S1, unofficial].
- For the CF method, `filter_radius` must equal half voxel size; otherwise, it has no effect [S1, unofficial].
- The 'auto' method selects 'cf' if `filter_radius` equals half voxel size, otherwise 'mc' [S1, unofficial].

## Subtopics

### [Differentiable Voxelization via Smoothed Winding Numbers](2325-differentiable-voxelization-via-smoothed-winding-numbers.md)
Differentiable Voxelization via Smoothed Winding Numbers (DVX) is a method for voxelizing triangle meshes into smoothed winding number voxel grids that provides closed-form gradients with respect to mesh vertex positions, enabling gradient-based optimization of surface representations based on volumetric properties. Unlike [Differentiable Voxelization (MC/CF)](1742-differentiable-voxelization-mc-cf.md) which samples a non-smoothed winding number and uses automatic differentiation (yielding sparse near-surface derivative signals), this method yields a broader gradient pattern matching finite-difference references.

### [Differentiable Voxelization from Radiative Gaussians](2380-differentiable-voxelization-from-radiative-gaussians.md)
This method provides a differentiable voxelizer $\mathcal{V}$ that efficiently queries a 3D density volume $V \in \mathbb{R}^{X \times Y \times Z}$ from a set of radiative Gaussians $G_3$. It solves the computational bottleneck of mapping continuous Gaussian splats to a discrete grid, enabling 3D regularization priors for tomographic reconstruction. Unlike [Differentiable Voxelization (MC/CF)](1742-differentiable-voxelization-mc-cf.md) which operates on polygonal meshes, this method operates directly on parametric Gaussian distributions, leveraging spatial tiling for acceleration.

### Backward Differentiable Closed-form Voxelization
Computes backward-mode gradients of voxel occupancy with respect to vertex positions using closed-form integration [S1, unofficial].
- **Inputs:** `d_occupancy`: input array of shape $(H, W)$ or $(D, H, W)$ containing occupancy gradients, type Float [S1, unofficial].
- **Outputs:** `d_vertices`: output array of shape $(N, D)$ containing vertex gradients, type Float [S1, unofficial].
- **Steps:** Validate inputs using `validate_common_differential_voxelize_arguments`. Dispatch to 2D or 3D backward-mode closed-form implementation [S1, unofficial].

### Forward Differentiable Closed-form Voxelization
- **Inputs:** `d_vertices`: input array of shape $(N, D)$ containing vertex gradients, type Float [S1, unofficial].
- **Outputs:** `d_occupancy`: output array of shape $(H, W)$ or $(D, H, W)$ containing occupancy gradients, type Float [S1, unofficial].
- **Steps:** Validate inputs using `validate_common_differential_voxelize_arguments`. Dispatch to 2D or 3D forward-mode closed-form implementation [S1, unofficial].

### 2D Polygon Rasterization to Grid Moments
Rasterizes a 2D polygon to a destination grid by computing polynomial density moments over the covered cells [S2, unofficial].
- **Inputs:** `poly`: The polygon to be rasterized. `ibox`: Minimum and maximum indices of the polygon, found with `r2d_get_ibox()`. Indices are from a virtual grid starting at the origin. `d`: The cell spacing of the grid. `polyorder`: Order of the polynomial density field. 0 for constant (1 moment), 1 for linear (3 moments), 2 for quadratic (6 moments) [S2, unofficial].
- **Outputs:** `dest_grid`: The rasterization buffer. A row-major grid patch starting at `d*ibox[0]` and ending at `d*ibox[1]`. Must have size at least $(ibox[1].i-ibox[0].i) \times (ibox[1].j-ibox[0].j) \times (ibox[1].k-ibox[0].k) \times R2D\_NUM\_MOMENTS(polyorder)$ [S2, unofficial].
- **Layout:** `r2d_dest_grid` struct: `r2d_real* moments[6]` (row-major C order, size $n.x \times n.y$), `r2d_int polyorder`, `r2d_orientation* orient` (size $(n.x+1) \times (n.y+1)$, unused if compiled with `-DUSE_TREE`), `r2d_long bufsz`, `r2d_dvec2 n` (grid dimensions), `r2d_rvec2 d` (cell size) [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Provides closed-form gradients for mesh voxelization, enabling gradient-based optimization. Supports both MC and CF methods.
- **Limitations:** CPU-only implementation (GPU tensors are moved to CPU). CF method has coordinate convention issues requiring transpose. Sparse derivative signals for MC variant.
- **When to use:** When differentiable voxelization of triangle meshes is required for gradient-based optimization.
- **When not to use:** When GPU-only execution is required or when operating on Gaussian splats instead of meshes.

## Background
Origin: The method is implemented in the `differentiable-voxelization` repository by M. Worchel [S1, unofficial]. It is used for differentiable rendering and optimization of surface representations.

## Sources
[S1] Differentiable Voxelization of Surface Representations, https://github.com/mworchel/differentiable-voxelization, fetched 2026-10-29 (unofficial)
[S2] devonmpowell/r3d, https://github.com/devonmpowell/r3d, fetched 2026-10-27 (unofficial)
