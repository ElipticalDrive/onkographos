# FlexiCubes

Also called: FlexiCubes Surface Extraction with Gradient-Based Optimization, Flexible Isosurface Extraction for Gradient-Based Mesh Optimization

## Overview
FlexiCubes is a differentiable isosurface extraction method that converts a Signed Distance Field (SDF) defined on a voxel grid into a triangle mesh, specifically designed for gradient-based optimization of geometric, visual, or physical objectives [S2, unofficial]. Unlike static extraction methods, it introduces learnable per-cube weights (beta, alpha, gamma) and vertex deformations to resolve topological ambiguities and improve mesh quality during training [S2, unofficial]. It builds on [Neural and Learning-based Extraction](6371-neural-and-learning-based-extraction.md) by providing a fully differentiable pipeline where the mesh geometry itself becomes an optimization variable [S2, unofficial].

## How it works
**Inputs:**
- Input mesh (OBJ format) or ground truth SDF data [S1, unofficial][S2, unofficial].
- Voxel grid resolution $R$ (integer, $4 \le R \le 512$) [S1, unofficial].
- Optimization parameters: learning rate, maximum iterations, convergence threshold, clamping values [S1, unofficial].

**Outputs:**
- Extracted triangle mesh (vertices and faces) [S1, unofficial][S2, unofficial].
- Optional optimized weights and deformed grid [S2, unofficial].

**Data Structures:**
1.  **Voxel Grid:**
    -   `x_nx3`: Tensor of shape $(N_v, 3)$ containing the 3D coordinates of grid vertices.
    -   `cube_fx8`: Tensor of shape $(N_c, 8)$ containing the indices of the 8 vertices defining each cube cell.
2.  **Learnable Parameters:**
    -   `sdf`: Tensor of shape $(N_v,)$ representing the SDF values at grid vertices [S2, unofficial].
    -   `deform`: Tensor of shape $(N_v, 3)$ representing vertex displacement vectors [S2, unofficial].
    -   `weight`: Tensor of shape $(N_c, 21)$ containing per-cube weights [S2, unofficial].
        -   `beta` (indices 0–11): Shape $(N_c, 12)$, controls edge weights [S1, unofficial].
        -   `alpha` (indices 12–19): Shape $(N_c, 8)$, controls corner weights [S1, unofficial].
        -   `gamma` (index 20): Scalar per cube, controls internal topology [S1, unofficial].
3.  **TriangleMeshDistance (TMD):** A query structure built from the input mesh vertices and faces to compute exact SDF values [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    -   Load input mesh and normalize it to a unit cube, storing the scale factor [S1, unofficial].
    -   Construct the voxel grid at resolution $R$, obtaining `x_nx3` and `cube_fx8` [S1, unofficial].
    -   Build the TMD structure from the normalized mesh vertices and faces [S1, unofficial].
    -   Initialize `sdf` by querying the TMD structure for the signed distance at each grid vertex `x_nx3` [S1, unofficial].
    -   Initialize `weight` (all zeros) and `deform` (all zeros) [S2, unofficial].
    -   Initialize `beta`, `alpha`, `gamma` to zero if optimization is enabled [S1, unofficial].

2.  **Deformation Calculation:**
    -   Compute the effective grid vertex positions `grid_verts` using the deformation tensor:
        $$ \text{grid\_verts} = \mathbf{x} + \frac{2 - 10^{-8}}{2R} \tanh(\text{deform}) $$
        where $\mathbf{x}$ are the original grid vertex positions, $R$ is the resolution, and the factor $(2 - 10^{-8})$ prevents numerical boundary issues [S2, unofficial].

3.  **Extraction and Loss Computation:**
    -   Call the differentiable extraction function `fc.extract_surface_with_grads(grid_verts, sdf, cube_fx8, R, beta, alpha, gamma, training=True)` [S2, unofficial].
    -   This function returns the extracted mesh vertices, faces, and the deviation loss $L_{dev}$ [S2, unofficial].
    -   Compute SDF regularization loss:
        $$ L_{sdf} = \text{mean}(\text{sdf\_reg\_loss}(sdf, \text{grid\_edges})) \times w_{sdf} $$
        where $w_{sdf}$ is the SDF regularization weight (default 0.2) [S2, unofficial].
    -   Compute total loss:
        $$ L_{total} = L_{sdf} + L_{dev} \times w_{dev} + L_{weight\_L1} \times w_{w1} $$
        where $L_{dev}$ is the mean deviation loss from extraction, $w_{dev}$ is the Laplacian deviation weight, and $L_{weight\_L1}$ is the L1 norm of `weight[:,:20]` with weight 0.1 [S2, unofficial].
    -   Optionally, add SDF MSE loss against ground truth if available:
        $$ L_{sdf\_mse} = \text{mean}((\text{SDF}_{pred} - \text{SDF}_{gt})^2) \times 2000 $$
        [S2, unofficial].
    -   Optionally add developability regularization if enabled [S2, unofficial].

4.  **Optimization Loop:**
    -   Compute gradients of $L_{total}$ with respect to `sdf`, `deform`, and `weight` via backpropagation through the extraction function [S2, unofficial].
    -   Update parameters using gradient descent [S1, unofficial] or Adam optimizer with default learning rate 0.01 [S2, unofficial].
    -   If `opt_clamp` > 0, clamp `beta`, `alpha`, and `gamma` to $[-\text{opt\_clamp}, \text{opt\_clamp}]$ [S1, unofficial].
    -   Check for convergence: if $|\text{loss}_{prev} - \text{loss}_{curr}| < \text{opt\_convergence}$ and iteration > 5, terminate [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def flexicubes_optimize(input_mesh_path, resolution, opt_iterations, opt_lr, opt_clamp, opt_convergence, sdf_weight=0.2, opt_l_dev_weight=0.0):
    # 1. Load and normalize mesh to unit cube
    mesh = parse_obj_file(input_mesh_path)
    scale = normalize_to_unit_cube(mesh)

    # 2. Construct voxel grid
    grid = create_regular_grid(resolution)
    x = grid.vertices  # (N_v, 3)
    cube_fx8 = grid.cubes  # (N_c, 8)

    # 3. Build nearest-point query structure and compute initial SDF
    tmd = build_nearest_point_query(mesh.vertices, mesh.faces)
    sdf = compute_signed_distances(tmd, x)  # (N_v,)

    # 4. Initialize learnable parameters
    deform = zeros_like(x)  # (N_v, 3)
    weight = zeros((cube_fx8.shape[0], 21))  # (N_c, 21)
    beta = weight[:, :12]
    alpha = weight[:, 12:20]
    gamma = weight[:, 20]

    prev_loss = float('inf')

    for iteration in range(opt_iterations):
        # 5. Compute deformed grid vertices
        # Factor (2 - 1e-8) prevents numerical boundary issues
        grid_verts = x + (2.0 - 1e-8) / (2.0 * resolution) * tanh(deform)

        # 6. Extract surface with gradients
        # training=True produces 4 triangles per quad for better mesh quality
        vertices, faces, L_dev = flexicubes_extract_surface(
            grid_verts, sdf, cube_fx8, resolution,
            beta, alpha, gamma, training=True
        )

        # 7. Compute losses
        # SDF regularization (e.g., smoothness on grid edges)
        grid_edges = compute_grid_adjacency_edges(x, resolution)
        reg_loss = mean(sdf_smoothness_loss(sdf, grid_edges)) * sdf_weight

        # Deviation loss from extraction
        reg_loss += mean(L_dev) * 0.5

        # Weight L1 regularization
        reg_loss += mean(abs(weight[:, :20])) * 0.1

        # 8. Backpropagation and parameter update
        gradients = compute_gradients(reg_loss, [sdf, deform, weight])
        sdf -= opt_lr * gradients.sdf
        deform -= opt_lr * gradients.deform
        beta -= opt_lr * gradients.beta
        alpha -= opt_lr * gradients.alpha
        gamma -= opt_lr * gradients.gamma

        # 9. Clamp weights if opt_clamp > 0
        if opt_clamp > 0.0:
            beta = clip(beta, -opt_clamp, opt_clamp)
            alpha = clip(alpha, -opt_clamp, opt_clamp)
            gamma = clip(gamma, -opt_clamp, opt_clamp)

        # 10. Convergence check
        loss_change = abs(prev_loss - reg_loss)
        if loss_change < opt_convergence and iteration > 5:
            break
        prev_loss = reg_loss

    return vertices, faces, weight, deform
```

**Parameters:**
- `resolution`: Voxel grid resolution, must be between 4 and 512 [S1, unofficial]. Default is 64 [S2, unofficial].
- `opt_iterations`: Maximum number of optimization iterations [S1, unofficial].
- `opt_lr`: Learning rate for gradient descent optimization [S1, unofficial]. Default is 0.01 [S2, unofficial].
- `opt_convergence`: Threshold for loss change to declare convergence [S1, unofficial].
- `opt_clamp`: Maximum absolute value for weight clamping; if > 0.0, weights are clamped to $[-\text{opt\_clamp}, \text{opt\_clamp}]$ [S1, unofficial].
- `opt_l_dev_weight`: Weight for the Laplacian deviation term in the loss function [S1, unofficial].
- `training`: Boolean flag set to true to use training mode, which produces 4 triangles per quad instead of 2 for better mesh quality [S2, unofficial].

**Edge Cases and Pitfalls:**
- Precision at boundaries is handled by the $(2 - 10^{-8})$ scaling factor to avoid numerical issues [S2, unofficial].
- Resolution must be strictly between 4 and 512; out-of-bounds values cause validation errors [S1, unofficial].
- Strong regularization weights can hinder convergence; low weights are recommended initially [S2, unofficial].
- Switching from tetrahedral grids (e.g., DMTet) to voxel grids requires a 4:5 resolution ratio to align output triangle counts, as voxel grids contain $(R+1)^3$ vertices versus $(R/2+1)^3$ for tetrahedra [S2, unofficial].

## Operating characteristics
-   **Time Complexity:** Extraction is $O(N_c)$ per iteration, where $N_c$ is the number of cubes [S2, unofficial]. Optimization adds gradient computation overhead proportional to the mesh size and network depth if the SDF is learned [S2, unofficial].
-   **Memory Complexity:** Dominated by the voxel grid and learnable parameters [S2, unofficial]. Memory scales with $O(R^3)$ for the grid and $O(R^3)$ for weights/deformations [S2, unofficial].
-   **Performance:** A $64^3$ FlexiCubes grid generates approximately the same number of triangles as an $80^3$ DMTet grid, but with higher computational cost due to the denser vertex count $(R+1)^3$ vs $(R/2+1)^3$ [S2, unofficial].
-   **Strengths:** High-quality mesh topology, differentiable, supports complex regularizers (e.g., developability), robust to ambiguous SDF values via learnable weights [S2, unofficial].
-   **Limitations:** Requires optimization iterations to converge; sensitive to regularization weights; higher memory usage than static extraction [S2, unofficial].
-   **When to use:** When optimizing a mesh against visual or geometric losses where gradient flow through the surface is required [S2, unofficial].
-   **When not to use:** For static, one-time extraction from a fixed SDF without optimization needs; for real-time inference where optimization latency is prohibitive [S2, unofficial].

## Background
FlexiCubes was introduced by Tianchang Shen et al. in 2023 as a successor to DMTet, addressing limitations in tetrahedral-based extraction [S2, unofficial]. It was published in ACM Transactions on Graphics (SIGGRAPH Asia 2023) [S2, unofficial]. The method is widely used in NVIDIA's 3D generative pipelines such as GET3D and nvdiffrec [S2, unofficial].

## Sources
[S1] ddiakopoulos/flexicubes_cpp, https://github.com/ddiakopoulos/flexicubes_cpp, fetched 2026-10-27 (unofficial)
[S2] GitHub - nv-tlabs/FlexiCubes, https://github.com/nv-tlabs/FlexiCubes, fetched 2026-10-27 (unofficial)
