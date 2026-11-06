# Differentiable Volume Rendering via Box Splines

Also called: Differentiable X-ray Transform with Ray-Trace Gradients, Differentiable X-ray transform, Differentiable XRT Forward Projection, xrt_ad_t (Geometry Gradient w.r.t. Ray Anchor)

## Overview
This method computes the differentiable X-ray transform (forward projection) of a volume defined on a uniform grid using box spline interpolation, providing exact gradients with respect to ray parameters (origin $\mathbf{t}$ and direction $\mathbf{n}$) and volume coefficients [S1, unofficial]. It solves the problem of back-propagating gradients through the ray-tracing integral in tomography and geometry calibration without relying on finite differences or implicit automatic differentiation through the entire ray march [S1, unofficial]. Unlike standard differentiable renderers that may use finite differences or implicit autodiff, this method explicitly computes ray-traced geometry gradients for anchors and directions using specific adjoint functions, while using the standard adjoint transform for volume coefficients [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `ray_spec`: A tuple `(t_spec, n_spec)` containing ray anchors $\mathbf{t} \in \mathbb{R}^L$ and directions $\mathbf{n} \in \mathbb{R}^L$, where $L$ is the number of rays. Both are arrays of shape `(L,)`.
    *   `knot_spec`: A `UniformSpec` object containing the volume grid properties: start $\mathbf{x}_0$, step $\mathbf{\Delta}$, and dimensions $\mathbf{Q}$ (num).
    *   `data`: Flattened volume weights $f_{\mathbf{q}}$ of shape $(Q_1 \times \dots \times Q_D)$, where $D \in \{2, 3\}$.
    *   `order`: Integer specifying interpolation order (0, 1, or 2). For 3D projections, only order 0 is supported [S1, unofficial].
    *   `mode`: String for automatic differentiation mode, typically `'symbolic'` [S1, unofficial].
*   **Outputs:**
    *   `proj`: Array of shape `(L,)` containing the projection values $\mathcal{R}[f](\mathbf{n}, \mathbf{t})$.
    *   Gradients w.r.t. `data`, `t_spec`, and `n_spec` are computed during the backward pass.

**Data Structures:**
*   `ArrayNf`: Ray type (e.g., `drjit.Array3f`).
*   `ArrayNi`: Integer type (`int32`).
*   `ArrayNu`: Unsigned integer type (`uint32`).
*   `Float`: Scalar value type.
*   `Bool`: Mask type.
*   `stride`: Linear indexing stride. For 2D: `ArrayNu(knot_num.y, 1)`. For 3D: `ArrayNu(knot_num.y * knot_num.z, knot_num.z, 1)`.

**Algorithm Steps:**

1.  **Initialization:**
    *   Determine dimension $D$ from `ray_t` (2 or 3).
    *   Compute bounding box:
        $$ \mathbf{bbox}_{ll} = \mathbf{x}_0 - \frac{\mathbf{\Delta}}{2} $$
        $$ \mathbf{bbox}_{ur} = \mathbf{x}_0 - \frac{\mathbf{\Delta}}{2} + (\mathbf{Q} \odot \mathbf{\Delta}) $$
    *   Initialize state:
        *   For order 0: `state = (buffer,)`.
        *   For order 1 (2D) and order > 0 (3D): `state = (buffer, index_prev)` with `index_prev = ArrayNi(-1)`.

2.  **Ray Bounding Box Intersection:**
    *   Compute intersection: `active, t1, t2 = ray_bbox_intersect(bbox_ll, bbox_ur, ray_t, ray_n)`.
    *   Compute `t_min = minimum(t1, t2)`.
    *   Adjust ray origin:
        ```python
        ray_t = select(active & bbox_contains(bbox_ll, bbox_ur, ray_t),
                       ray_t + (t_min - 1) * ray_n,
                       ray_t)
        ```

3.  **DDA Traversal:**
    *   Execute Digital Differential Analyzer (DDA) loop:
        ```python
        dda(ray_o=ray_t, ray_d=ray_n, ray_max=inf,
            grid_res=knot_num, grid_min=bbox_ll, grid_max=bbox_ur,
            func=project, state=state, active=active, mode=mode,
            max_iterations=-1)
        ```
    *   The `project` callback is invoked for each cell intersected by the ray. It receives `index` (cell index), `p_a` (entry point in [0,1]^D), `p_b` (exit point in [0,1]^D), and `active` mask.

4.  **Projection Weight Calculation (`project` callback):**
    The calculation depends on dimension and order.

    *   **3D Order 0 (Voxel Case):**
        *   Compute offset: `offset = dot(index, stride)`.
        *   Gather weight: `fq = gather(Float, data, offset, active)`.
        *   Compute length: $L = \|\mathbf{p}_b - \mathbf{p}_a\| \cdot \|\mathbf{\Delta}\| \cdot \frac{1}{\prod \mathbf{\Delta}}$.
        *   Accumulate: `accum += fq * L`.

    *   **2D Order 0:**
        *   Similar to 3D Order 0, but using 2D stride and grid resolution.

    *   **2D Order 1 or 2:**
        *   Compute perpendicular vector: $\mathbf{n}_{\perp} = \text{normalize}(-n_y, n_x)$.
        *   Compute box spline parameters: `E, E_mask = box_spline_1d_E(order, knot_step, n_perp)`.
        *   Determine movement direction:
            ```python
            direction = ray_n * rcp(knot_step)
            look_lr = abs(direction.x) >= abs(direction.y)
            mv_dir = select(look_lr, ArrayNi(+1, 0), ArrayNi(0, +1))
            shift_l = reverse(-mv_dir)
            shift_m = ArrayNi(0, 0)
            shift_r = reverse(+mv_dir)
            ```
        *   For each shift $s \in \{\text{shift}_l, \text{shift}_m, \text{shift}_r\}$:
            *   Compute `index_s = index + shift`.
            *   Check bounds: `active_s = all((0 <= index_s) & (index_s < knot_num))`.
            *   Gather `fq`.
            *   Compute cell center: $\mathbf{c} = (0.5, 0.5) + \text{shift}$.
            *   Compute position: $x = \mathbf{n}_{\perp} \cdot (\mathbf{\Delta} \odot (\mathbf{c} - \mathbf{p}_a))$.
            *   Compute length: $L = \text{box\_spline\_1d\_dr}(E, E\_mask, x)$.
        *   Handle displacement logic to avoid double-counting:
            ```python
            displacement = index - index_prev
            fq_lmr = if_stmt((fq_l, fq_m, fq_r),
                             dot(displacement, mv_dir) != 0,
                             lambda l,m,r: (l,m,r),
                             lambda l,m,r: select(dot(displacement, reverse(mv_dir)) == -1, (l,0,0), (0,0,r)))
            accum += dot(fq_lmr, (L_l, L_m, L_r))
            ```
        *   Update `index_prev = index`.

    *   **3D Order > 0 (Neural Network Spline Evaluation):**
        *   Normalize ray direction: $\mathbf{n} = \text{normalize}(\mathbf{ray\_n})$.
        *   Compute direction magnitudes: $\mathbf{d} = |\mathbf{ray\_n} \odot \mathbf{\Delta}^{-1}|$.
        *   Compute dominant axis: $\mathbf{a} = \lfloor \mathbf{d} / \max(\mathbf{d}) + 10^{-6} \rfloor$.
        *   Compute shifts:
            ```python
            shift_x = ArrayNf(a.y, a.z, a.x)
            shift_y = cross(a, shift_x)
            shift_m = ArrayNi(0, 0, 0)
            ```
        *   Compute perpendiculars:
            ```python
            n_perp_x = normalize(shift_x - dot(shift_x, n) * n)
            n_perp_y = cross(n, n_perp_x)
            ```
        *   Define shifts: $\text{shift}_m$, $\pm \text{shift}_x$, $\pm \text{shift}_y$, $\pm \text{shift}_x \pm \text{shift}_y$.
        *   For each shift, compute `index_s`, gather `fq`, compute cell center $\mathbf{c} = (0.5, 0.5, 0.5) + \text{shift}$, and coordinates:
            $$ x = \mathbf{n}_{\perp_y} \cdot (\mathbf{\Delta} \odot (\mathbf{c} - \mathbf{p}_a)) $$
            $$ y = \mathbf{n}_{\perp_x} \cdot (\mathbf{\Delta} \odot (\mathbf{c} - \mathbf{p}_a)) $$
        *   Compute length using neural network: $L = \text{spline\_3d\_dr}(\text{net}, x, y, \mathbf{n})$.
        *   Accumulate with complex displacement logic to handle corner weights and avoid double counting (see [S1, unofficial] for full logic involving `fq_lmr`, `fq_dmu`, and corner weights `fq_ll`, etc.).

5.  **Backward Pass (Gradients):**
    *   Retrieve output gradient `g = self.grad_out()`.
    *   **Gradient w.r.t. Volume (`x`):**
        *   Compute adjoint transform: `rx.xrt_adjoint(rs, knot_spec, order, g)`.
        *   Set as gradient for `x`.
    *   **Gradient w.r.t. Ray Anchors (`t`):**
        *   For 2D: `gt[0] = g * rx.xrt_ad_t_x(...)`, `gt[1] = g * rx.xrt_ad_t_y(...)`.
        *   For 3D: additionally `gt[2] = g * rx.xrt_ad_t_z(...)`.
        *   Set `gt` as gradient for `t_spec`.
    *   **Gradient w.r.t. Ray Directions (`n`):**
        *   For 2D: `gn[0] = g * rx.xrt_ad_n_x(...)`, `gn[1] = g * rx.xrt_ad_n_y(...)`.
        *   For 3D: additionally `gn[2] = g * rx.xrt_ad_n_z(...)`.
        *   Set `gn` as gradient for `n_spec`.

**Pseudocode for 2D Order 1 Gradient w.r.t. Anchor:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient_anchor_2d(order, knot_step, ray_n, p_a, p_b, active, eps=1e-6):
    # p_a, p_b are fractional positions in [0,1]^2
    # ray_n is direction vector
    # Returns gradient of projection weight P w.r.t. p_a components
    
    # Compute sigma (exit indicator) for each dimension
    sigma_x = select(p_a.x < eps, 1, 0) + select(p_b.x < eps, 1, 0) - select(p_a.x > 1.0 - eps, 1, 0) - select(p_b.x > 1.0 - eps, 1, 0)
    sigma_y = select(p_a.y < eps, 1, 0) + select(p_b.y < eps, 1, 0) - select(p_a.y > 1.0 - eps, 1, 0) - select(p_b.y > 1.0 - eps, 1, 0)
    
    # Compute sigma_out (exit face indicator)
    sigma_out_x = select(p_b.x < eps, 1, 0) - select(p_b.x > 1.0 - eps, 1, 0)
    sigma_out_y = select(p_b.y < eps, 1, 0) - select(p_b.y > 1.0 - eps, 1, 0)
    
    n_abs_x = abs(ray_n.x)
    n_abs_y = abs(ray_n.y)
    
    # g_s: gradient component from sigma
    g_s_x = select(n_abs_x > eps, sigma_x / n_abs_x, 0)
    g_s_y = select(n_abs_y > eps, sigma_y / n_abs_y, 0)
    
    # Compute length factor L_fw
    L_fw = norm((p_b - p_a) * knot_step) * rcp(prod(knot_step))
    
    # local_g_d: gradient component from length
    local_g_d_x = select(n_abs_x > eps, L_fw * sigma_out_x / n_abs_x, 0)
    local_g_d_y = select(n_abs_y > eps, L_fw * sigma_out_y / n_abs_y, 0)
    
    # Compute fac_t
    n_norm_sq = squared_norm(ray_n)
    # p_a_phys is physical position, but fac_t uses fractional p_a relative to ray_t_init
    # Assuming ray_t_init is the original ray origin before adjustment
    # fac_t = dot(p_a_phys - ray_t_init, ray_n) / n_norm_sq
    # In the source, p_a is fractional, so p_a_phys = bbox_ll + (index + p_a) * knot_step
    # However, for gradient w.r.t. t_idx, we use the chain rule through t
    # The source computes fac_t based on p_a_phys and ray_t_init
    # For implementation, we need ray_t_init. If not available, this step is marked [G]
    # Assuming ray_t_init is passed or available in context
    p_a_phys = bbox_ll + (index + p_a) * knot_step
    fac_t = select(n_norm_sq > eps*eps, dot(p_a_phys - ray_t_init, ray_n) / n_norm_sq, 0)
    
    # Final gradient
    g_x = local_g_d_x + fac_t * g_s_x
    g_y = local_g_d_y + fac_t * g_s_y
    
    return ArrayNf(g_x, g_y)
```

## Subtopics

### Neural Network Spline Evaluation
A neural network used to evaluate the 3D box spline projection for orders > 0. It takes inputs related to position and direction and outputs the spline value. Weights are loaded from `spline_weights.npz` file. The network architecture is `dnn.Sequential` with layers: `dnn.Linear(4, 16)`, `dnn.ReLU()`, `dnn.Linear(16, 16)`, `dnn.ReLU()`, `dnn.Linear(16, 16)`, `dnn.ReLU()`, `dnn.Linear(16, 16)`, `dnn.ReLU()`, `dnn.Linear(16, 1)`. Weights are stored in `TensorXf16` format. It is called as `spline_3d_dr(net, x, y, n)` where `x` and `y` are projected coordinates and `n` is the normalized ray direction [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear in the number of rays and the number of cells intersected by each ray. The DDA traversal is efficient for uniform grids.
*   **Memory Complexity:** Linear in the number of rays and the volume size.
*   **Performance:** Measured performance depends on hardware and resolution. The use of explicit adjoint functions for geometry gradients avoids the overhead of full automatic differentiation through the ray march, making it efficient for geometry calibration tasks [S1, unofficial].
*   **Strengths:** Exact gradients for ray parameters; supports 2D and 3D grids; handles multiple interpolation orders.
*   **Limitations:** 3D projections only support order 0 (closed-form) or require a neural network for higher orders; forward-mode differentiation is not implemented [S1, unofficial].
*   **When to use:** Geometry calibration in tomography, differentiable ray-based models where gradients w.r.t. ray parameters are needed [S1, unofficial].
*   **When not to use:** When forward-mode differentiation is required; when 3D higher-order interpolation is needed without a pre-trained neural network.

## Background
Originated in the `differentiable_xrt` library by HaouchatY, implementing Algorithms 1-2 of the reference paper as a vector-Jacobian product within Dr.Jit's automatic differentiation framework [S1, unofficial]. It is used in geometry calibration in tomography, specifically for differentiable ray-based models [S1, unofficial].

## Sources
[S1] HaouchatY/differentiable_xrt, https://github.com/HaouchatY/differentiable_xrt, fetched 2026-10-27 (unofficial)
