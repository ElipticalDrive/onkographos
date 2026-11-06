# FlexiCubes

Also called: FlexiCubes Surface Extraction with Gradient-Based Optimization, Flexible Isosurface Extraction for Gradient-Based Mesh Optimization

## Overview
FlexiCubes is a differentiable isosurface extraction method that converts a Signed Distance Field (SDF) defined on a voxel grid into a triangle mesh, specifically designed for gradient-based optimization of geometric, visual, or physical objectives [S2, unofficial]. Unlike static extraction methods, it introduces learnable per-cube weights (beta, alpha, gamma) and vertex deformations to resolve topological ambiguities and improve mesh quality during training [S2, unofficial]. It builds on [Neural and Learning-based Extraction](6371-neural-and-learning-based-extraction.md) by providing a fully differentiable pipeline where the mesh geometry itself becomes an optimization variable [S2, unofficial].

## How it works
**Inputs:**
- Input mesh (OBJ format) or ground truth SDF data.
- Voxel grid resolution $R$ (integer, $4 \le R \le 512$) [S1, unofficial].
- Optimization parameters: learning rate, max iterations, convergence threshold, clamping values [S1, unofficial].

**Outputs:**
- Extracted triangle mesh (vertices and faces).
- Optional optimized weights and deformed grid.

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
    -   Call the differentiable extraction function `fc.extract_surface_with_grads(grid_verts, sdf, cube_fx8, R, beta, alpha, gamma, training=True)`.
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
    -   Compute gradients of $L_{total}$ with respect to `sdf`, `deform`, and `weight` via backpropagation through the extraction function.
    -   Update parameters using an optimizer (e.g., Adam with lr 0.01) or gradient descent [S1, unofficial][S2, unofficial].
    -   If `opt_clamp` > 0, clamp `beta`, `alpha`, and `gamma` to $[-\text{opt\_clamp}, \text{opt\_clamp}]$ [S1, unofficial].
    -   Check for convergence: if $|\text{loss}_{prev} - \text{loss}_{curr}| < \text{opt\_convergence}$ and iteration > 5, terminate [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def flexicubes_optimize(input_mesh_path, resolution, opt_iterations, opt_lr, opt_clamp, opt_convergence, sdf_weight=0.2, opt_l_dev_weight=0.0):
    # 1. Load and Normalize
    mesh = load_obj(input_mesh_path)
    scale = mesh.normalize_to_unit_cube()
    
    # 2. Construct Voxel Grid
    grid = construct_voxel_grid(resolution)
    x_nx3 = grid.vertices # (N_v, 3)
    cube_fx8 = grid.cubes # (N_c, 8)
    
    # 3. Build TMD and Compute Initial SDF
    tmd = TriangleMeshDistance(mesh.vertices, mesh.faces)
    sdf = torch.zeros(x_nx3.shape[0])
    for i in range(x_nx3.shape[0]):
        sdf[i] = tmd.signed_distance(x_nx3[i]).distance
        
    # 4. Initialize Learnable Parameters
    deform = torch.zeros_like(x_nx3, requires_grad=True)
    weight = torch.zeros((cube_fx8.shape[0], 21), requires_grad=True)
    
    # 5. Optimization Loop
    prev_loss = float('inf')
    for iter in range(opt_iterations):
        # Compute deformed grid vertices
        # Factor (2 - 1e-8) / (2 * resolution) scales deformation to cube edge length
        grid_verts = x_nx3 + (2 - 1e-8) / (2 * resolution) * torch.tanh(deform)
        
        # Extract surface and get gradients
        # training=True ensures 4 triangles per quad for better gradient flow
        vertices, faces, L_dev = fc.extract_surface_with_grads(
            grid_verts, sdf, cube_fx8, resolution,
            beta=weight[:, :12],
            alpha=weight[:, 12:20],
            gamma=weight[:, 20],
            training=True
        )
        
        # Compute Losses
        # SDF Regularization (e.g., smoothness)
        reg_loss = sdf_reg_loss(sdf, grid_edges).mean() * sdf_weight
        
        # Deviation Loss from extraction
        dev_loss = L_dev.mean() * opt_l_dev_weight
        
        # Weight L1 Regularization
        weight_reg = weight[:, :20].abs().mean() * 0.1
        
        total_loss = reg_loss + dev_loss + weight_reg
        
        # Backpropagation
        total_loss.backward()
        
        # Update weights (using SGD example; Adam is also used in practice)
        with torch.no_grad():
            weight -= opt_lr * weight.grad
            deform -= opt_lr * deform.grad
            sdf -= opt_lr * sdf.grad
            
            # Zero gradients
            weight.grad.zero_()
            deform.grad.zero_()
            sdf.grad.zero_()
            
            # Clamp weights if specified
            if opt_clamp > 0.0:
                weight[:, :12] = torch.clamp(weight[:, :12], -opt_clamp, opt_clamp)
                weight[:, 12:20] = torch.clamp(weight[:, 12:20], -opt_clamp, opt_clamp)
                weight[:, 20] = torch.clamp(weight[:, 20], -opt_clamp, opt_clamp)
                
        # Convergence Check
        loss_change = abs(prev_loss - total_loss.item())
        if loss_change < opt_convergence and iter > 5:
            break
        prev_loss = total_loss.item()
        
    return vertices, faces
```

## Operating characteristics
-   **Time Complexity:** Extraction is $O(N_c)$ per iteration, where $N_c$ is the number of cubes. Optimization adds gradient computation overhead proportional to the mesh size and network depth if the SDF is learned [S2, unofficial].
-   **Memory Complexity:** Dominated by the voxel grid and learnable parameters. Memory scales with $O(R^3)$ for the grid and $O(R^3)$ for weights/deformations [S2, unofficial].
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
