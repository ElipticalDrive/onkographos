# CTDR Optimization Loop with Regularization

Also called: CTDR training loop, Shape from Projections optimization

## Overview
An iterative optimization process that minimizes a composite loss function—combining data fidelity, edge, Laplacian, and flatness regularization terms—to reconstruct a 3D triangular mesh directly from 2D tomographic projection data [S1, unofficial]. It distinguishes itself from standard differentiable rendering by explicitly enforcing topological constraints via collision rollback and utilizing robust loss functions (Student-t) to handle missing data or outliers in projection masks [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A differentiable forward projector model (derived from [Shape from Projections via Differentiable Forward Projector](13333-shape-from-projections-via-differentiable-forward-projector.md)), initial mesh parameters (vertex positions, material densities $\mu$), 2D projection images $p_{batch}$, angular indices $idx\_angles$, and a wedge mask $args.wedge$ defining valid pixel regions.
*   **Outputs:** Optimized mesh geometry and material values.

**Data Structures:**
*   `model`: Contains differentiable mesh representation. Attributes include `mus` (material densities, size $N$), and internal CUDA tensors for forward/backward passes (e.g., `points2d_bxfx6`, `grad_points2d_bxfx6`).
*   `optimizer`: Adam or SGD instance.
*   `args`: Configuration object containing weights `wedge`, `wlap`, `wflat`, and flags like `silhouette_mode`.

**Numbered Steps:**
1.  **Initialize Optimizer:** Create an optimizer using Adam with learning rate $lr$ and betas $(\beta_1=0.9, \beta_2=0.99)$, or SGD with momentum $0.9$ and `nesterov=True` [S1, unofficial].
2.  **Forward Projection:** Compute the predicted projections $\hat{p}$, validity mask, and regularization losses by calling the model:
    $$ \hat{p}, \text{mask\_valid}, L_{\text{edge}}, L_{\text{lap}}, L_{\text{flat}} = \text{model}(idx\_angles, args.wedge) $$
3.  **Compute Data Loss:** Calculate the fidelity term on valid pixels.
    *   **L2 Loss:**
        $$ L_{\text{data}} = \frac{1}{|\Omega|} \sum_{i \in \Omega} (p_i - \hat{p}_i)^2 $$
        where $\Omega$ is the set of indices where `mask_valid` is true [S1, unofficial].
    *   **Student-t Loss (Alternative):** If robust fitting is required, use:
        $$ L_{\text{data}} = \frac{1}{|\Omega|} \sum_{i \in \Omega} \log\left(1 + \left(\frac{p_i - \hat{p}_i}{\sigma}\right)^2\right) $$
        with $\sigma=1$ [S1, unofficial].
    *   **Background Loss (Optional):** If `silhouette_mode` is enabled, add a penalty for non-zero predictions in background regions:
        $$ L_{\text{data}} \leftarrow L_{\text{data}} + \frac{1}{|\Omega_{bg}|} \sum_{j \in \Omega_{bg}} \hat{p}_j^2 $$
        where $\Omega_{bg}$ are background indices [S1, unofficial].
4.  **Compute Total Loss:** Combine data fidelity with regularization terms:
    $$ L_{\text{total}} = L_{\text{data}} + w_{\text{edge}} L_{\text{edge}} + w_{\text{lap}} L_{\text{lap}} + w_{\text{flat}} L_{\text{flat}} $$
    where $w_{\text{edge}}, w_{\text{lap}}, w_{\text{flat}}$ are scalar weights from `args` [S1, unofficial].
5.  **Backpropagation:** Compute gradients via automatic differentiation:
    $$ \nabla L_{\text{total}} = \frac{\partial L_{\text{total}}}{\partial \theta} $$
    where $\theta$ represents mesh parameters (vertex positions, $\mu$) [S1, unofficial].
6.  **Optimization Step:** Update parameters:
    $$ \theta \leftarrow \theta - \eta \cdot \text{OptimizerStep}(\nabla L_{\text{total}}) $$
    [S1, unofficial].
7.  **Clamp Material Values:** Enforce non-negativity on material densities:
    $$ \mu \leftarrow \max(0, \mu) $$
    [S1, unofficial].
8.  **Topology Preservation (Optional):** If `use_collision` is enabled, perform a collision rollback to maintain mesh topology:
    *   Detect self-intersections or invalid topology.
    *   Revert displacements using `keep_topology(model, search_tree, displace_prev)` [S1, unofficial].
9.  **Convergence Check:** If the change in loss $|L_{\text{prev}} - L_{\text{curr}}| < 10^{-11}$ and $L_{\text{curr}} > 0$, terminate [S1, unofficial].
10. **Learning Rate Decay:** If `epoch + 100 == niter` and `niter > 400`, halve the learning rate [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
optimizer = Adam(model.parameters(), lr=lr, betas=(0.9, 0.99))
# or: optimizer = SGD(model.parameters(), lr=lr, momentum=0.9, nesterov=True)

for epoch in range(niter):
    # 1. Forward pass
    phat, mask_valid, edge_loss, lap_loss, flat_loss = model(idx_angles, args.wedge)
    
    # 2. Data loss calculation
    diff = p_batch - phat
    valid_diff = diff[mask_valid]
    
    # Choose loss type (L2 or Student-t)
    if use_student_t:
        sigma = 1.0
        data_loss = torch.log(1 + (valid_diff / sigma)**2).mean()
    else:
        data_loss = valid_diff.pow(2).mean()
        
    # Optional background loss
    if silhouette_mode:
        bg_loss = phat[idx_bg].pow(2).mean()
        data_loss += bg_loss
        
    # 3. Total loss
    loss = data_loss + args.wedge * edge_loss + args.wlap * lap_loss + args.wflat * flat_loss
    
    # 4. Backpropagation and step
    loss.backward()
    optimizer.step()
    optimizer.zero_grad() # Clear gradients for next iteration
    
    # 5. Post-processing
    loss_now = loss.item()
    
    # Clamp material densities to non-negative
    model.mus.data.clamp_(min=0.0)
    
    # Topology preservation
    if use_collision:
        keep_topology(model, search_tree, displace_prev)
        
    # 6. Convergence check
    if epoch > 0:
        dloss = abs(loss_prev - loss_now)
        if dloss < 1e-11 and dloss > 0:
            break
    loss_prev = loss_now
    
    # 7. Learning rate decay
    if niter > 400 and (epoch + 100) == niter:
        for param_group in optimizer.param_groups:
            param_group['lr'] /= 2.0
```

**Parameters:**
*   `lr`: Learning rate. Typical value not specified, but Adam betas are $(0.9, 0.99)$ [S1, unofficial].
*   `sigma`: Scale parameter for Student-t loss. Value: $1$ [S1, unofficial].
*   `wedge`, `wlap`, `wflat`: Weights for edge, Laplacian, and flatness regularization. Values not specified [S1, unofficial].
*   `niter`: Total iterations. Decay triggers at `niter > 400` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Convergence threshold is set to $10^{-11}$, requiring high-precision loss tracking [S1, unofficial].
*   **Boundaries:** Material values must be clamped to $\ge 0$ to prevent physical invalidity [S1, unofficial].
*   **Topology:** Without collision rollback, the mesh may self-intersect or invert, leading to invalid projections. The `keep_topology` function handles this but requires a search tree and previous displacement buffer [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the forward projection and its CUDA-based backward pass, which scales with the number of pixels, triangles, and rays. The optimization loop adds $O(N_{iter})$ overhead.
*   **Memory Complexity:** Requires storing gradients for all mesh parameters and intermediate CUDA tensors (e.g., `grad_points2d_bxfx6`, `grad_mus_n`) [S1, unofficial].
*   **Strengths:** Handles missing data via Student-t loss and masks; enforces smoothness and flatness via regularization; preserves topology during deformation.
*   **Limitations:** Computationally expensive due to differentiable ray-marching and backpropagation through CUDA kernels; sensitive to regularization weights; requires careful initialization to avoid local minima.
*   **When to use:** When reconstructing 3D shapes from sparse or noisy tomographic projections where mesh topology must be preserved.
*   **When not to use:** For real-time rendering or when volumetric (voxel) representation is sufficient and topology is not a concern.

## Background
The method originates from the "ShapeFromProjections" project by Jake Young, which implements differentiable rendering for tomographic reconstruction [S1, unofficial]. It builds upon differentiable forward projectors that compute gradients of projection data with respect to mesh vertices and material densities [S1, unofficial]. The approach integrates concepts from inverse rendering and geometric regularization to solve the ill-posed problem of shape recovery from 2D projections.

## Sources
[S1] jakeoung/ShapeFromProjections, https://github.com/jakeoung/ShapeFromProjections, fetched 2026-10-26 (unofficial)
