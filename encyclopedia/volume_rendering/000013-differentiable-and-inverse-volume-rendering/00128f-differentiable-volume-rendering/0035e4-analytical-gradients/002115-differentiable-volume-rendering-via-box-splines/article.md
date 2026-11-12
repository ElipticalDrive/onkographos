# Differentiable Volume Rendering via Box Splines

Also called: Differentiable X-ray Transform with Ray-Trace Gradients, Differentiable X-ray transform, Differentiable XRT Forward Projection, xrt_ad_t (Geometry Gradient w.r.t Ray Anchor)

## Overview
This method computes the forward projection of a volume defined by box splines and its analytical gradients with respect to ray parameters (origin $\mathbf{t}$ and direction $\mathbf{n}$) and volume data coefficients. It implements the differentiable X-ray transform operator, supporting 2D and 3D uniform grids with interpolation orders 0, 1, and 2. Unlike standard differentiable renderers that rely on finite differences or implicit automatic differentiation through the entire ray march, this method explicitly computes ray-traced geometry gradients for anchors and directions using specific adjoint functions, while using the standard adjoint transform for volume coefficients. It is primarily used in geometry calibration in tomography.

## How it works
The method computes the projection integral $\int_{\mathbb{R}} f(\mathbf{t} + \tau \mathbf{n}) d\tau$ for a volume $f(\mathbf{x}) = \sum_{\mathbf{q}} f_{\mathbf{q}} \psi(\mathbf{x} - \mathbf{x}_{\mathbf{q}})$, where $\psi$ is a box spline basis function determined by the interpolation order. The volume is defined on a uniform grid with start $\mathbf{x}_0$, step $\mathbf{\Delta}$, and dimensions $\mathbf{Q}$.

### Inputs and Outputs
*   **Inputs**:
    *   `ray_spec`: Tuple `(t_spec, n_spec)` containing ray anchors $\mathbf{t}$ and directions $\mathbf{n}$, both arrays of shape `(L,)`.
    *   `knot_spec`: `UniformSpec` object containing volume grid properties: start $\mathbf{x}_0$, step $\mathbf{\Delta}$, and number of knots $\mathbf{Q}$.
    *   `data`: Flattened volume weights $f_{\mathbf{q}}$ of shape `(Q1, Q2)` for 2D or `(Q1, Q2, Q3)` for 3D.
    *   `order`: Integer specifying interpolation order (0, 1, or 2).
    *   `mode`: String for automatic differentiation mode (typically `'symbolic'`).
*   **Outputs**:
    *   Forward: Projection values $\hat{C} \in \mathbb{R}^L$.
    *   Backward: Gradients with respect to `data` (adjoint transform), `t` (ray anchors), and `n` (ray directions).

### Data Structures
*   `ArrayNf`: Ray type (e.g., `drjit.Array3f`).
*   `ArrayNi`: Integer type (`int32`).
*   `ArrayNu`: Unsigned integer type (`uint32`).
*   `Float`: Scalar value type.
*   `Bool`: Mask type.
*   `stride`: Linear indexing stride. For 2D: `(knot_num.y, 1)`. For 3D: `(knot_num.y * knot_num.z, knot_num.z, 1)`.

### Algorithm Steps

1.  **Initialization**:
    *   Compute bounding box: $\mathbf{bbox}_{ll} = \mathbf{x}_0 - (\mathbf{\Delta} / 2)$, $\mathbf{bbox}_{ur} = \mathbf{x}_0 - (\mathbf{\Delta} / 2) + (\mathbf{Q} \cdot \mathbf{\Delta})$.
    *   Compute stride based on dimension $D$ (2 or 3).
    *   Initialize accumulator buffer.

2.  **Ray Bounding Box Intersection**:
    *   Intersect ray with $\mathbf{bbox}_{ll}$ and $\mathbf{bbox}_{ur}$ to get entry/exit distances $t_1, t_2$.
    *   Compute $t_{min} = \min(t_1, t_2)$.
    *   Adjust ray origin if necessary: $\mathbf{t} \leftarrow \mathbf{t} + (t_{min} - 1) \mathbf{n}$ if active and inside.

3.  **DDA Traversal**:
    *   Execute Digital Differential Analyzer (DDA) traversal over the grid cells intersected by the ray.
    *   For each cell index $\mathbf{idx}$, entry point $\mathbf{p}_a$, and exit point $\mathbf{p}_b$ (in local grid coordinates $[0,1]^D$), call the `project` callback.

4.  **Projection Callback (`project`)**:
    The implementation of `project` depends on the dimension and order.

    **Case 1: 3D, Order 0 (Voxel)**
    *   Compute length factor: $L_{fw} = \frac{\|\mathbf{p}_b - \mathbf{p}_a\| \cdot \|\mathbf{\Delta}\|}{\prod \mathbf{\Delta}}$.
    *   For each dimension $idx \in \{0, 1, 2\}$:
        *   Compute sigma (exit indicator):
            $$ \sigma = \mathbb{I}(p_a[idx] < \epsilon) + \mathbb{I}(p_b[idx] < \epsilon) - \mathbb{I}(p_a[idx] > 1 - \epsilon) - \mathbb{I}(p_b[idx] > 1 - \epsilon) $$
        *   Compute $n_{abs} = |n[idx]|$.
        *   Compute gradient component $g_s$:
            $$ g_s = \begin{cases} \sigma / n_{abs} & \text{if } n_{abs} > \epsilon \\ 0 & \text{otherwise} \end{cases} $$
        *   Accumulate gradient w.r.t. anchor: $g[idx] += f_{\mathbf{idx}} \cdot g_s$.
    *   Accumulate forward projection: $accum += f_{\mathbf{idx}} \cdot L_{fw}$.

    **Case 2: 2D or 3D Non-Voxel (Order > 0)**
    *   Determine movement direction: $\mathbf{direction} = \mathbf{n} / \mathbf{\Delta}$.
    *   Determine dominant axis to define shifts.
        *   For 2D: $\mathbf{mv\_dir} = (1, 0)$ if $|direction.x| \ge |direction.y|$, else $(0, 1)$.
        *   For 3D: Compute dominant axis, then perpendiculars $\mathbf{n}_{perp\_x}, \mathbf{n}_{perp\_y}$.
    *   Define shifts relative to current cell index $\mathbf{idx}$:
        *   2D: $\mathbf{shift}_l, \mathbf{shift}_m, \mathbf{shift}_r$.
        *   3D: $\mathbf{shift}_m, \pm \mathbf{shift}_x, \pm \mathbf{shift}_y, \pm \mathbf{shift}_x \pm \mathbf{shift}_y$.
    *   For each shift $\mathbf{s}$:
        *   Compute $\mathbf{idx}_s = \mathbf{idx} + \mathbf{s}$.
        *   Check bounds. If valid, gather $f_q$ from `data` at offset $\mathbf{idx}_s \cdot \mathbf{stride}$.
        *   Compute cell center $\mathbf{c} = (0.5, 0.5, 0.5) + \mathbf{s}$.
        *   Compute local coordinates $\mathbf{x}, \mathbf{y}$ in the perpendicular plane.
        *   Compute spline weight $L = \psi(\mathbf{x}, \mathbf{y}, \mathbf{n})$.
            *   For 2D: Use `box_spline_1d_dr`.
            *   For 3D Order > 0: Use neural network `spline_3d_dr` or closed-form if available.
    *   **Accumulation Logic**:
        *   Compute displacement $\mathbf{d} = \mathbf{idx} - \mathbf{idx}_{prev}$.
        *   Use conditional logic (`if_stmt`) to select which shifts contributed to the current cell transition (left, middle, right or up/down).
        *   Accumulate forward projection: $accum += \sum (f_q \cdot L)$.
        *   **Gradient Computation**:
            *   Create local copies of ray parameters: $\mathbf{n}_{local} = \mathbf{n}, \mathbf{p}_{local} = \mathbf{p}_a$.
            *   Enable gradients on local copies.
            *   Compute contribution function $C(\mathbf{p}_a, \mathbf{n})$ using the same spline evaluation logic.
            *   Run backward pass: $\nabla C$.
            *   Compute gradient w.r.t. anchor: $\mathbf{g}_p = \nabla_{\mathbf{p}} C$.
            *   Compute gradient w.r.t. direction: $\mathbf{g}_n = \nabla_{\mathbf{n}} C$.
            *   Combine: $\mathbf{g} = \mathbf{g}_p + \frac{\mathbf{p}_a - \mathbf{t}_{init}}{\|\mathbf{n}\|^2} \cdot \mathbf{g}_n$ (simplified form; exact implementation uses `fac_t`).

5.  **Backward Pass (Global)**:
    *   Retrieve output gradient $\mathbf{g}_{out}$.
    *   **Gradient w.r.t. Volume (`x`)**: Compute adjoint transform using `xrt_adjoint`. This distributes $\mathbf{g}_{out}$ back to volume coefficients based on the forward projection weights.
    *   **Gradient w.r.t. Ray Anchors (`t`)**: Compute partial derivatives using specific adjoint functions (`xrt_ad_t_x`, `xrt_ad_t_y`, `xrt_ad_t_z`).
        *   $g_t[idx] = g_{out} \cdot \frac{\partial P}{\partial t_{idx}}$.
    *   **Gradient w.r.t. Ray Directions (`n`)**: Compute partial derivatives using specific adjoint functions (`xrt_ad_n_x`, `xrt_ad_n_y`, `xrt_ad_n_z`).
        *   $g_n[idx] = g_{out} \cdot \frac{\partial P}{\partial n_{idx}}$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def xrt_apply(ray_spec, knot_spec, order, data, buffer=None, mode='symbolic'):
    ray_t, ray_n = ray_spec
    ArrayNf = type(ray_t)
    ArrayNi = dr.int32_array_t(ArrayNf)
    ArrayNu = dr.uint32_array_t(ArrayNf)
    Float = dr.value_t(ArrayNf)
    Bool = dr.mask_t(Float)
    
    D = dr.size_v(ArrayNf)
    assert D in (2, 3)
    assert order in (0, 1, 2)
    
    L = max(ray_t.shape[1], ray_n.shape[1])
    if buffer is None:
        buffer = dr.zeros(Float, L)
        
    knot_start = ArrayNf(*knot_spec.start)
    knot_step = ArrayNf(*knot_spec.step)
    knot_num = ArrayNu(*knot_spec.num)
    
    bbox_ll = knot_start - (knot_step / 2)
    bbox_ur = knot_start - (knot_step / 2) + (knot_num * knot_step)
    
    if D == 2:
        stride = ArrayNu(knot_num.y, 1)
    else:
        stride = ArrayNu(knot_num.y * knot_num.z, knot_num.z, 1)

    # Ray Bounding Box Intersection
    active, t1, t2 = ray_bbox_intersect(bbox_ll, bbox_ur, ray_t, ray_n)
    t_min = dr.minimum(t1, t2)
    ray_t = dr.select(
        active & bbox_contains(bbox_ll, bbox_ur, ray_t),
        ray_t + (t_min - 1) * ray_n,
        ray_t
    )

    # State Initialization
    if order == 0 and D == 3:
        state = (buffer,)
    else:
        index_prev = ArrayNi(-1)
        state = (buffer, index_prev)

    # DDA Traversal
    dda(
        ray_o=ray_t, ray_d=ray_n, ray_max=Float(dr.inf),
        grid_res=knot_num, grid_min=bbox_ll, grid_max=bbox_ur,
        func=project, state=state, active=active, mode=mode, max_iterations=-1
    )
    return buffer

def project(state, index, p_a, p_b, active):
    # Implementation varies by order and dimension as described in steps
    # ...
    return state, Bool(True)
```

## Subtopics

### Neural Network Spline Evaluation
A neural network used to evaluate the 3D box spline projection for orders > 0. It takes inputs related to position and direction and outputs the spline value. The network architecture is `dnn.Sequential` with layers: `dnn.Linear(4, 16)`, `dnn.ReLU()`, `dnn.Linear(16, 16)`, `dnn.ReLU()`, `dnn.Linear(16, 16)`, `dnn.ReLU()`, `dnn.Linear(16, 16)`, `dnn.ReLU()`, `dnn.Linear(16, 1)`. Weights are stored in `TensorXf16` format and loaded from `spline_weights.npz`. It is called as `spline_3d_dr(net, x, y, n)` where `x` and `y` are projected coordinates and `n` is the normalized ray direction.

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of rays and the number of cells intersected by each ray (DDA traversal). The spline evaluation is constant time per cell (or constant time neural network inference).
*   **Memory Complexity**: Linear with respect to the number of rays for the buffer and gradients. The volume data is stored once.
*   **Performance**: The method is efficient because it avoids storing the full computational graph. Gradients are computed in a single backward pass or alongside the forward pass with constant memory overhead relative to the number of parameters.
*   **Strengths**: Analytical gradients for ray parameters and volume data. Supports 2D and 3D. Handles different interpolation orders.
*   **Limitations**: 3D order 0 uses a closed-form gradient, while higher orders use AD or NN. Forward-mode differentiation is not implemented. Only reverse-mode (backward) differentiation is supported.
*   **When to use**: Geometry calibration in tomography, differentiable ray-based models where analytical gradients are required.
*   **When not to use**: When forward-mode differentiation is needed, or when the scene is not defined on a uniform grid with box spline basis.

## Background
The method originates from the paper "Differentiable X-ray Transform with Ray-Trace Gradients" (implied by the source [S1, unofficial]). It is implemented as a vector-Jacobian product within Dr.Jit's automatic differentiation framework. The method handles 2D and 3D uniform grids, supporting interpolation orders 0, 1, and 2. It is used in geometry calibration in tomography.

## Sources
[S1] HaouchatY/differentiable_xrt, https://github.com/HaouchatY/differentiable_xrt, fetched 2026-10-27 (unofficial)
