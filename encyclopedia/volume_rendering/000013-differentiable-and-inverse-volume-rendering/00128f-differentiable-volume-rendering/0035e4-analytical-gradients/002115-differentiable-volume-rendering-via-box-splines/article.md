# Differentiable Volume Rendering via Box Splines

Also called: Differentiable X-ray Transform with Ray-Trace Gradients, Differentiable X-ray transform, Differentiable XRT Forward Projection, xrt_ad_t (Geometry Gradient w.r.t Ray Anchor)

## Overview
This method computes the forward projection of a volume defined by box splines and its analytical gradients with respect to ray parameters (origin $\mathbf{t}$ and direction $\mathbf{n}$) and volume coefficients. It implements a differentiable X-ray transform operator that supports 2D and 3D uniform grids with interpolation orders 0, 1, and 2. Unlike standard differentiable renderers that rely on finite differences or implicit automatic differentiation through the entire ray march, this method explicitly computes ray-traced geometry gradients for anchors and directions using specific adjoint functions, while using the standard adjoint transform for volume coefficients. It is primarily used in geometry calibration in tomography.

## How it works
The method computes samples of the projection integral $\xrt[f](\mathbf{n}, \mathbf{t}) = \int_{\mathbb{R}} f(\mathbf{t} + \tau \mathbf{n}) d\tau$ for a volume defined on a uniform grid. The volume is represented as $f(\mathbf{x}) = \sum_{\mathbf{q} \in \text{discreteRange}(\mathbf{0}, \mathbf{Q}-1)} f_{\mathbf{q}} \psi(\mathbf{x} - \mathbf{x}_{\mathbf{q}})$, where $\psi$ is a box spline basis function determined by the interpolation order and direction vectors $\mathbf{E}$.

### Inputs and Outputs
**Inputs:**
- `ray_spec`: Tuple `(t_spec, n_spec)` where `t_spec` and `n_spec` are arrays of shape `(L,)` representing ray anchors and directions.
- `knot_spec`: `UniformSpec` object containing volume properties: start $\mathbf{x}_0$, step $\mathbf{\Delta}$, and dimensions $\mathbf{Q}$.
- `data`: Flattened volume weights of shape $(Q_1, \dots, Q_D)$ for $D$ dimensions.
- `order`: Integer specifying interpolation order (0, 1, or 2). For 3D projections, only order 0 is supported in the closed-form variant; orders 1 and 2 are supported for 2D.
- `mode`: String for automatic differentiation mode, typically `'symbolic'`.

**Outputs:**
- `proj`: Array of shape `(L,)` containing projection values.
- Gradients with respect to `data`, `t_spec`, and `n_spec` (computed during backward pass).

### Data Structures
- `ArrayNf`: Ray type (e.g., `drjit.Array3f`).
- `ArrayNi`: Integer type (`int32`).
- `ArrayNu`: Unsigned integer type (`uint32`).
- `Float`: Scalar value type.
- `Bool`: Mask type.
- `stride`: Linear indexing stride. For 2D: `ArrayNu(Q_y, 1)`. For 3D: `ArrayNu(Q_y * Q_z, Q_z, 1)`.

### Algorithm Steps

1. **Initialization**:
   - Compute bounding box: `bbox_ll = knot_start - (knot_step / 2)`, `bbox_ur = knot_start - (knot_step / 2) + (knot_num * knot_step)`.
   - Compute `stride` based on dimension $D$.
   - Initialize `buffer` of shape `(L,)` to zero.
   - Initialize state: For order 0, `state = (buffer,)`. For order 1 (2D) and order > 0 (3D), `state = (buffer, index_prev)` with `index_prev = ArrayNi(-1)`.

2. **Ray Bounding Box Intersection**:
   - Compute `active, t1, t2 = ray_bbox_intersect(bbox_ll, bbox_ur, ray_t, ray_n)`.
   - Compute `t_min = minimum(t1, t2)`.
   - Adjust ray origin: `ray_t = select(active & bbox_contains(bbox_ll, bbox_ur, ray_t), ray_t + (t_min - 1) * ray_n, ray_t)`.

3. **DDA Traversal**:
   - Execute DDA loop: `dda(ray_o=ray_t, ray_d=ray_n, ray_max=inf, grid_res=knot_num, grid_min=bbox_ll, grid_max=bbox_ur, func=project, state=state, active=active, mode=mode, max_iterations=-1)`.

4. **Project Callback (Forward Pass)**:
   The `project` callback is invoked for each cell intersection. It depends on dimension $D$ and `order`.

   **Case 1: 3D Order 0 (Voxel)**
   - Compute offset: `offset = dot(index, stride)`.
   - Gather weight: `fq = gather(Float, data, offset, active)`.
   - Compute length: `L = norm((p_b - p_a) * knot_step) * rcp(prod(knot_step))`.
   - Accumulate: `accum += fq * L`.

   **Case 2: 2D or 3D Non-Voxel (Box Spline)**
   - Determine movement direction: `direction = ray_n * rcp(knot_step)`.
   - Determine look direction: `look_lr = abs(direction.x) >= abs(direction.y)`.
   - Set move direction: `mv_dir = select(look_lr, (+1, 0), (0, +1))`.
   - Define shifts: `shift_l = reverse(-mv_dir)`, `shift_m = (0, 0)`, `shift_r = reverse(+mv_dir)`.
   - For each shift `s` in `{shift_l, shift_m, shift_r}`:
     - Compute `index_s = index + shift`.
     - Check bounds: `active_s = all((0 <= index_s) & (index_s < knot_num))`.
     - Compute `offset = dot(index_s, stride)`.
     - Gather `fq = gather(Float, data, offset, active_s)`.
     - Compute `cell_center = (0.5, 0.5) + shift`.
     - Enable gradients on local copies: `n_local = ArrayNf(ray_n)`, `p_local = ArrayNf(p_a)`. `enable_grad(n_local, p_local)`.
     - Define contribution function:
       ```python
       def contribution(p_a_, ray_n_):
           n_perp = normalize((-ray_n_.y, ray_n_.x))
           E, E_mask = box_spline_1d_E(order, knot_step, n_perp)
           x = dot(n_perp, knot_step * (cell_center - p_a_))
           return box_spline_1d_dr(E, E_mask, x)
       ```
     - Run backward pass: `backward(contribution(p_local, n_local), flags=ADFlag.Default | ADFlag.AllowNoGrad)`.
     - Compute gradient `g = n_local.grad[idx] + fac_t * p_local.grad[idx]`. (Note: `fac_t` is computed as described in gradient section).
     - Compute length `L = box_spline_1d_dr(E, E_mask, x)`.
     - Handle displacement logic to select `fq_lmr` and `L_lmr` based on `dot(displacement, mv_dir)`.
     - Accumulate: `accum += dot(fq_lmr, (L_l, L_m, L_r))`.
     - Update `index_prev = index`.

   **Case 3: 3D Order > 0 (Neural Network Spline)**
   - Normalize ray direction: `n = normalize(ray_n)`.
   - Compute direction: `direction = abs(ray_n * rcp(knot_step))`.
   - Compute dominant axis: `dominant_axis = floor(direction / max(direction) + 1e-6)`.
   - Compute shifts: `shift_x`, `shift_y`, `shift_m = (0,0,0)`.
   - Compute perpendiculars: `n_perp_x = normalize(shift_x - dot(shift_x, n) * n)`, `n_perp_y = cross(n, n_perp_x)`.
   - For each shift in `{shift_m, shift_x, -shift_x, shift_y, -shift_y, shift_x + shift_y, -shift_x - shift_y, shift_x - shift_y, -shift_x + shift_y}`:
     - Compute `index_s`, gather `fq`.
     - Compute `cell_center = (0.5, 0.5, 0.5) + shift`.
     - Compute `x = dot(n_perp_y, knot_step * (cell_center - p_a))`, `y = dot(n_perp_x, knot_step * (cell_center - p_a))`.
     - Compute `L = spline_3d_dr(net, x, y, n)`.
   - Accumulate using complex logic involving `displacement` to handle corner weights and avoid double counting.

5. **Gradient Computation (Backward Pass)**:
   - Retrieve output gradient `g = self.grad_out()`.
   - Determine dimension `D`.
   - Initialize gradient accumulators for `t` and `n` as zero arrays.
   - **Gradient w.r.t. volume (`x`)**: If `want_x`, compute adjoint transform using `xrt_adjoint(rs, knot_spec, order, g)` and set as gradient for `x`.
   - **Gradient w.r.t. ray anchors (`t`)**: If `want_t`, compute partial derivatives.
     - For 2D: `gt[0] = g * xrt_ad_t_x(...)`, `gt[1] = g * xrt_ad_t_y(...)`.
     - For 3D: Additionally `gt[2] = g * xrt_ad_t_z(...)`.
     - Set `gt` as gradient for `t_spec`.
   - **Gradient w.r.t. ray directions (`n`)**: If `want_n`, compute partial derivatives.
     - For 2D: `gn[0] = g * xrt_ad_n_x(...)`, `gn[1] = g * xrt_ad_n_y(...)`.
     - For 3D: Additionally `gn[2] = g * xrt_ad_n_z(...)`.
     - Set `gn` as gradient for `n_spec`.

### Formulas and Parameters

**Box Spline Direction Vectors $\mathbf{E}$:**
- Order 0: $\mathbf{E} = [\mathbf{\Delta}]$.
- Order 1 (2D): $\mathbf{E} = [\mathbf{\Delta}_1, 0, \mathbf{\Delta}_1, 0, \mathbf{\Delta}_2, \mathbf{\Delta}_2]$.
- Order 2 (2D): $\mathbf{E} = [\mathbf{\Delta}_1, 0, \mathbf{\Delta}_1, \mathbf{\Delta}_1, 0, \mathbf{\Delta}_2, \mathbf{\Delta}_2, -\mathbf{\Delta}_2]$.

**Gradient w.r.t. Ray Anchor (3D Order 0 Closed-Form):**
For dimension `idx`:
$$ \sigma = \mathbb{I}(p_a[idx] < \epsilon) + \mathbb{I}(p_b[idx] < \epsilon) - \mathbb{I}(p_a[idx] > 1.0 - \epsilon) - \mathbb{I}(p_b[idx] > 1.0 - \epsilon) $$
$$ n_{abs} = |n[idx]| $$
$$ g = \begin{cases} \sigma / n_{abs} & \text{if } n_{abs} > \epsilon \\ 0 & \text{otherwise} \end{cases} $$
The gradient contribution is `fq * g`.

**Gradient w.r.t. Ray Anchor (General):**
$$ g_s = \begin{cases} \sigma / n_{abs} & \text{if } n_{abs} > \epsilon \\ 0 & \text{otherwise} \end{cases} $$
$$ \sigma_{out} = \mathbb{I}(p_b[idx] < \epsilon) - \mathbb{I}(p_b[idx] > 1.0 - \epsilon) $$
$$ L_{fw} = \frac{\|(p_b - p_a) \mathbf{\Delta}\|}{\prod \mathbf{\Delta}} $$
$$ local\_g_d = \begin{cases} L_{fw} \cdot \sigma_{out} / n_{abs} & \text{if } n_{abs} > \epsilon \\ 0 & \text{otherwise} \end{cases} $$
$$ p_{a\_phys} = \mathbf{x}_0 + (\text{index} + p_a) \mathbf{\Delta} $$
$$ fac_t = \begin{cases} \frac{\dot(p_{a\_phys} - \mathbf{t}_{init}, \mathbf{n})}{\|\mathbf{n}\|^2} & \text{if } \|\mathbf{n}\|^2 > \epsilon^2 \\ 0 & \text{otherwise} \end{cases} $$
$$ g = local\_g_d + fac_t \cdot g_s $$

**Parameters:**
- `order`: 0, 1, or 2.
- `eps`: Small constant for numerical stability (e.g., $10^{-6}$).

**Edge Cases and Pitfalls:**
- When $n[idx]=0$, the ray never crosses `idx`-faces; $\sigma=0$ too, so return 0 to avoid division by zero.
- Enabling grad on loop-external `ray_n` breaks symbolic tracing; must use local copies `n_local` and `p_local`.
- Forward-mode differentiation is not implemented. Only reverse-mode (backward) differentiation is supported.

## Subtopics

### Neural Network Spline Evaluation
A neural network used to evaluate the 3D box spline projection for orders > 0. It takes inputs related to position and direction and outputs the spline value. Weights are loaded from `spline_weights.npz` file. The network architecture is `dnn.Sequential` with layers: `dnn.Linear(4, 16)`, `dnn.ReLU()`, `dnn.Linear(16, 16)`, `dnn.ReLU()`, `dnn.Linear(16, 16)`, `dnn.ReLU()`, `dnn.Linear(16, 16)`, `dnn.ReLU()`, `dnn.Linear(16, 1)`. Weights are stored in `TensorXf16` format. It is called as `spline_3d_dr(net, x, y, n)` where `x` and `y` are projected coordinates and `n` is the normalized ray direction.

## Operating characteristics
- **Time Complexity**: Linear in the number of rays and the number of cells intersected by each ray. The DDA traversal is efficient for uniform grids.
- **Memory Complexity**: Linear in the number of rays and the volume size.
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales well with the number of rays due to parallelization in Dr.Jit.
- **Strengths**: Analytical gradients for ray parameters allow for efficient optimization in tomography and geometry calibration. Supports multiple interpolation orders.
- **Limitations**: 3D projections only support order 0 in the closed-form variant. Orders 1 and 2 for 3D require neural network evaluation, which may be slower and less precise than analytical methods. Forward-mode differentiation is not supported.
- **When to use**: Geometry calibration in tomography, differentiable ray-based models.
- **When not to use**: When forward-mode differentiation is required, or when high-order 3D analytical gradients are needed without neural network approximation.

## Background
The method originates from the paper "Differentiable X-ray Transform with Ray-Trace Gradients" by Haouchat et al. [S1, unofficial]. It is implemented as part of the `differentiable_xrt` library using Dr.Jit's automatic differentiation framework. The method explicitly differentiates the projection integral with respect to ray parameters, providing a robust alternative to finite differences or implicit autodiff through the entire ray march.

## Sources
[S1] HaouchatY/differentiable_xrt, https://github.com/HaouchatY/differentiable_xrt, fetched 2026-10-27 (unofficial)
