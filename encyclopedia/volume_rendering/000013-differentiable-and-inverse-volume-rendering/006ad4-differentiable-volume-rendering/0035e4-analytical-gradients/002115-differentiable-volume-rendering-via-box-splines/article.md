# Differentiable Volume Rendering via Box Splines

Also called: Differentiable X-ray Transform with Ray-Trace Gradients, Differentiable X-ray transform, Differentiable XRT Forward Projection, xrt_ad_t (Geometry Gradient w.r.t Ray Anchor)

## Overview
This method computes the forward projection of a volume defined by box splines and its analytical gradients with respect to ray parameters (origin $\mathbf{t}$ and direction $\mathbf{n}$) and volume data coefficients [S1, unofficial]. It solves the differentiable X-ray transform (XRT) for uniform 2D and 3D grids with interpolation orders 0, 1, and 2 [S1, unofficial]. Unlike standard differentiable renderers that rely on finite differences or implicit automatic differentiation through the entire ray-marching loop, this method explicitly computes ray-traced geometry gradients using specific adjoint functions for ray anchors and directions, while using the standard adjoint transform for volume coefficients [S1, unofficial].

## How it works
The method implements the differentiable X-ray transform operator $\mathcal{X}[f](\mathbf{n}, \mathbf{t}) = \int_{\mathbb{R}} f(\mathbf{t} + \tau \mathbf{n}) d\tau$ for a volume $f(\mathbf{x}) = \sum_{\mathbf{q} \in \text{discreteRange}(\mathbf{0}, \mathbf{Q}-1)} f_{\mathbf{q}} \psi(\mathbf{x} - \mathbf{x}_{\mathbf{q}})$, where $\psi$ is a box spline basis function determined by the interpolation order [S1, unofficial]. The volume is defined on a uniform grid with start $\mathbf{x}_0$, step $\mathbf{\Delta}$, and dimensions $\mathbf{Q}$ [S1, unofficial].

**Inputs:**
- `ray_spec`: Tuple `(t_spec, n_spec)` of shape `(L,)` containing ray anchors $\mathbf{t}$ and directions $\mathbf{n}$ [S1, unofficial].
- `knot_spec`: `UniformSpec` object containing volume properties $(\mathbf{x}_0, \mathbf{\Delta}, \mathbf{Q})$ [S1, unofficial].
- `data`: Flattened volume weights of shape $(Q_1, \dots, Q_D)$ [S1, unofficial].
- `order`: Integer specifying interpolation order (0, 1, or 2) [S1, unofficial].
- `mode`: String for AD mode, typically `'symbolic'` [S1, unofficial].

**Outputs:**
- `proj`: Array of shape `(L,)` containing projection values [S1, unofficial].
- Gradients w.r.t. `data`, `t_spec`, and `n_spec` are computed during the backward pass [S1, unofficial].

**Data Structures:**
- `ArrayNf`: Ray type (e.g., `drjit.Array3f`) [S1, unofficial].
- `ArrayNi`: Integer type (`int32`) [S1, unofficial].
- `ArrayNu`: Unsigned integer type (`uint32`) [S1, unofficial].
- `Float`: Scalar value type [S1, unofficial].
- `Bool`: Mask type [S1, unofficial].
- `stride`: Linear indexing stride. For 2D: `(Q_y, 1)`. For 3D: `(Q_y Q_z, Q_z, 1)` [S1, unofficial].

**Algorithm Steps:**

1. **Initialization:**
   - Compute bounding box: $\mathbf{bbox}_{ll} = \mathbf{x}_0 - (\mathbf{\Delta} / 2)$, $\mathbf{bbox}_{ur} = \mathbf{x}_0 - (\mathbf{\Delta} / 2) + (\mathbf{Q} \cdot \mathbf{\Delta})$ [S1, unofficial].
   - Compute `stride` based on dimension $D$ and $\mathbf{Q}$ [S1, unofficial].
   - Initialize accumulator `buffer` of size $L$ to zeros [S1, unofficial].
   - Initialize state: For order 0, `state = (buffer,)`. For order 1 (2D) and order > 0 (3D), `state = (buffer, index_prev)` with `index_prev = ArrayNi(-1)` [S1, unofficial].

2. **Ray Bounding Box Intersection:**
   - Compute intersection: `active, t1, t2 = ray_bbox_intersect(bbox_ll, bbox_ur, ray_t, ray_n)` [S1, unofficial].
   - Compute `t_min = minimum(t1, t2)` [S1, unofficial].
   - Adjust ray origin: `ray_t = select(active & bbox_contains(bbox_ll, bbox_ur, ray_t), ray_t + (t_min - 1) * ray_n, ray_t)` [S1, unofficial].

3. **DDA Traversal:**
   - Execute DDA loop: `dda(ray_o=ray_t, ray_d=ray_n, ray_max=inf, grid_res=Q, grid_min=bbox_ll, grid_max=bbox_ur, func=project, state=state, active=active, mode=mode, max_iterations=-1)` [S1, unofficial].
   - The `project` callback is invoked for each cell $(index, p_a, p_b, active)$ where $p_a, p_b$ are entry/exit fractional positions [S1, unofficial].

4. **Projection Callback (`project`):**
   The implementation of `project` depends on the dimension $D$ and order:

   **Case 1: 3D, Order 0 (Voxel)**
   - Compute closed-form gradient for each dimension $idx \in \{0, 1, 2\}$ [S1, unofficial]:
     - $\sigma = \text{select}(p_a[idx] < \epsilon, 1, 0) + \text{select}(p_b[idx] < \epsilon, 1, 0) - \text{select}(p_a[idx] > 1.0 - \epsilon, 1, 0) - \text{select}(p_b[idx] > 1.0 - \epsilon, 1, 0)$ [S1, unofficial].
     - $n_{abs} = |n[idx]|$ [S1, unofficial].
     - $g = \text{select}(n_{abs} > \epsilon, \sigma / n_{abs}, 0)$ [S1, unofficial].
   - Accumulate: `accum += fq * g` where `fq` is gathered from `data` at `offset = dot(index, stride)` [S1, unofficial].

   **Case 2: 2D (Any Order) or 3D Order > 0 (Using AD)**
   - **Setup:**
     - Compute perpendicular vector: $\mathbf{n}_{perp} = \text{normalize}((-n_y, n_x))$ [S1, unofficial].
     - Compute box spline parameters: $E, E_{mask} = \text{box\_spline\_1d\_E}(order, \mathbf{\Delta}, \mathbf{n}_{perp})$ [S1, unofficial].
     - Determine movement direction: $\text{direction} = \mathbf{n} / \mathbf{\Delta}$, $\text{look\_lr} = |\text{direction}.x| \ge |\text{direction}.y|$, $\text{mv\_dir} = \text{select}(\text{look\_lr}, (1, 0), (0, 1))$ [S1, unofficial].
     - Define shifts: $\text{shift}_l = \text{reverse}(-\text{mv\_dir})$, $\text{shift}_m = (0, 0)$, $\text{shift}_r = \text{reverse}(\text{mv\_dir})$ [S1, unofficial].
   - **Processing Shifts:**
     - For each shift $s \in \{\text{shift}_l, \text{shift}_m, \text{shift}_r\}$:
       - Compute `index_s = index + shift` [S1, unofficial].
       - Check bounds: `active_s = all((0 <= index_s) & (index_s < Q))` [S1, unofficial].
       - Gather `fq = gather(data, dot(index_s, stride), active_s)` [S1, unofficial].
       - Compute cell center: $\mathbf{c} = (0.5, 0.5) + \text{shift}$ [S1, unofficial].
       - Enable gradients on local copies: $\mathbf{p}_{local} = \mathbf{p}_a$, $\mathbf{n}_{local} = \mathbf{n}$, `enable_grad(p_local, n_local)` [S1, unofficial].
       - Define contribution function:
         $$ \text{contribution}(\mathbf{p}_a, \mathbf{n}) = \text{box\_spline\_1d\_dr}(E, E_{mask}, \mathbf{n}_{perp} \cdot \mathbf{\Delta} \cdot (\mathbf{c} - \mathbf{p}_a)) $$
         where $\mathbf{n}_{perp} = \text{normalize}((-n_y, n_x))$ [S1, unofficial].
       - Run backward pass: `backward(contribution(p_local, n_local))` [S1, unofficial].
       - Compute gradient: $g = \mathbf{n}_{local}.grad[idx] + \text{fac\_t} \cdot \mathbf{p}_{local}.grad[idx]$ [S1, unofficial].
         - Note: $\text{fac\_t}$ is computed as $\text{select}(n_{norm\_sq} > \epsilon^2, \mathbf{p}_{phys} \cdot \mathbf{n} / n_{norm\_sq}, 0)$ where $\mathbf{p}_{phys} = \mathbf{bbox}_{ll} + (\text{index} + \mathbf{p}_a) \cdot \mathbf{\Delta}$ and $n_{norm\_sq} = |\mathbf{n}|^2$ [S1, unofficial].
       - Return `fq` and `g` (length $L$) [S1, unofficial].
   - **Accumulation Logic:**
     - Compute `displacement = index - index_prev` [S1, unofficial].
     - Select `fq_lmr` based on `dot(displacement, mv_dir) != 0` [S1, unofficial].
     - Accumulate: `accum += dot(fq_lmr, (L_l, L_m, L_r))` [S1, unofficial].
     - Update `index_prev = index` [S1, unofficial].

   **Case 3: 3D Order > 0 (Using Neural Network for Spline)**
   - **Setup:**
     - Normalize $\mathbf{n} = \text{normalize}(\mathbf{n})$ [S1, unofficial].
     - Compute direction: $\text{direction} = |\mathbf{n} / \mathbf{\Delta}|$ [S1, unofficial].
     - Compute dominant axis: $\text{dominant\_axis} = \text{floor}(\text{direction} / \max(\text{direction}) + 10^{-6})$ [S1, unofficial].
     - Compute shifts: $\text{shift}_x = (d_y, d_z, d_x)$, $\text{shift}_y = \text{cross}(\text{dominant\_axis}, \text{shift}_x)$ [S1, unofficial].
     - Compute perpendiculars: $\mathbf{n}_{perp\_x} = \text{normalize}(\text{shift}_x - \text{dot}(\text{shift}_x, \mathbf{n}) \mathbf{n})$, $\mathbf{n}_{perp\_y} = \text{cross}(\mathbf{n}, \mathbf{n}_{perp\_x})$ [S1, unofficial].
     - Define shifts: $\text{shift}_m, \text{shift}_x, -\text{shift}_x, \text{shift}_y, -\text{shift}_y, \text{shift}_x+\text{shift}_y, -\text{shift}_x-\text{shift}_y, \text{shift}_x-\text{shift}_y, -\text{shift}_x+\text{shift}_y$ [S1, unofficial].
   - **Processing Shifts:**
     - For each shift, compute `index_s`, gather `fq`, compute $\mathbf{c} = (0.5, 0.5, 0.5) + \text{shift}$, compute $x = \mathbf{n}_{perp\_y} \cdot \mathbf{\Delta} \cdot (\mathbf{c} - \mathbf{p}_a)$, $y = \mathbf{n}_{perp\_x} \cdot \mathbf{\Delta} \cdot (\mathbf{c} - \mathbf{p}_a)$, and $L = \text{spline\_3d\_dr}(net, x, y, \mathbf{n})$ [S1, unofficial].
   - **Accumulation Logic:**
     - Compute `displacement = index - index_prev` [S1, unofficial].
     - Use `if_stmt` to select `fq_lmr` and `fq_dmu` based on `dot(displacement, shift_x)` and `dot(displacement, shift_y)` [S1, unofficial].
     - Zero out corner weights if displacement conditions are not met [S1, unofficial].
     - Accumulate: `accum += dot(fq_dmu, L_dmu) + dot(fq_lmr, L_lmr) + fq_ll * L_ll + fq_rr * L_rr + fq_lr * L_lr + fq_rl * L_rl` [S1, unofficial].

5. **Backward Pass (Gradients):**
   - Retrieve output gradient `g = self.grad_out()` [S1, unofficial].
   - **Gradient w.r.t. Volume (`x`):** If `want_x`, compute adjoint transform using `xrt_adjoint(rs, knot_spec, order, g)` [S1, unofficial].
   - **Gradient w.r.t. Ray Anchors (`t`):** If `want_t`, compute partial derivatives. For 2D: `gt[0] = g * xrt_ad_t_x(...)`, `gt[1] = g * xrt_ad_t_y(...)`. For 3D: additionally `gt[2] = g * xrt_ad_t_z(...)` [S1, unofficial].
   - **Gradient w.r.t. Ray Directions (`n`):** If `want_n`, compute partial derivatives. For 2D: `gn[0] = g * xrt_ad_n_x(...)`, `gn[1] = g * xrt_ad_n_y(...)`. For 3D: additionally `gn[2] = g * xrt_ad_n_z(...)` [S1, unofficial].

**Parameters:**
- `order`: 0, 1, or 2. Order 0 uses $\mathbf{E} = \mathbf{\Delta}$. Order 1 (2D) uses $\mathbf{E} = [\mathbf{\Delta}_1, 0, \mathbf{\Delta}_1, 0, \mathbf{\Delta}_2, \mathbf{\Delta}_2]$. Order 2 (2D) uses $\mathbf{E} = [\mathbf{\Delta}_1, 0, \mathbf{\Delta}_1, \mathbf{\Delta}_1, 0, \mathbf{\Delta}_2, \mathbf{\Delta}_2, -\mathbf{\Delta}_2]$ [S1, unofficial].
- `eps`: Small constant for numerical stability (e.g., $10^{-6}$) [S1, unofficial].

**Edge Cases and Pitfalls:**
- When $n[idx] = 0$, the ray never crosses $idx$-faces; $\sigma=0$ too, so return 0 to avoid division by zero [S1, unofficial].
- Enabling grad on loop-external `ray_n` breaks symbolic tracing; must use local copies `n_local` and `p_local` [S1, unofficial].
- Forward-mode differentiation is not implemented; only reverse-mode (backward) differentiation is supported [S1, unofficial].
- 3D order 0 uses a closed-form gradient calculation (Algorithm vox2) instead of the general box-spline projection weight differentiation used for other orders/dimensions [S1, unofficial].

## Subtopics

### Neural Network Spline Evaluation
A neural network used to evaluate the 3D box spline projection for orders > 0. It takes inputs related to position and direction and outputs the spline value. Weights are loaded from `spline_weights.npz` file. The network architecture is `dnn.Sequential` with layers: `dnn.Linear(4, 16)`, `dnn.ReLU()`, `dnn.Linear(16, 16)`, `dnn.ReLU()`, `dnn.Linear(16, 16)`, `dnn.ReLU()`, `dnn.Linear(16, 16)`, `dnn.ReLU()`, `dnn.Linear(16, 1)`. Weights are stored in `TensorXf16` format. It is called as `spline_3d_dr(net, x, y, n)` where `x` and `y` are projected coordinates and `n` is the normalized ray direction [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels intersected by rays, determined by the DDA traversal. The backward pass is also $O(N)$ due to the explicit gradient computation [S1, unofficial].
- **Memory Complexity:** $O(N)$ for the output size, enabling single-pass or tightly coupled backward passes [G]. The state maintains `index_prev` for tracking cell transitions [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of rays and intersected voxels. The use of symbolic AD allows efficient gradient computation without storing the full computational graph [G].
- **Strengths:** Explicit computation of ray-traced geometry gradients for anchors and directions using specific adjoint functions (`xrt_ad_t_*`, `xrt_ad_n_*`), while using the standard adjoint transform for volume coefficients. Supports 2D and 3D uniform grids with interpolation orders 0, 1, and 2 [S1, unofficial].
- **Limitations:** Forward-mode differentiation is not implemented. 3D projections only support order 0 in the general case, with higher orders requiring a neural network approximation.
- **When to use:** Geometry calibration in tomography, specifically for differentiable ray-based models [S1, unofficial].
- **When not to use:** When forward-mode differentiation is required or when non-uniform grids are used.

## Background
The method is based on the differentiable X-ray transform operator, implemented as a vector-Jacobian product within Dr.Jit's automatic differentiation framework [S1, unofficial]. It implements Algorithms 1-2 of the reference paper [S1, unofficial]. The approach explicitly expands the chain rule across the continuous stages of the rendering pipeline, bypassing the computational graph storage required by reverse-mode AD [G]. For volumetric methods, the rendering integral is differentiated term-by-term, yielding a direct term for local emission and an indirect term for downstream transmittance or visibility attenuation [G]. The implementation is part of the `differentiable_xrt` library [S1, unofficial].

## Sources
[S1] HaouchatY/differentiable_xrt, https://github.com/HaouchatY/differentiable_xrt, fetched 2026-10-27 (unofficial)
