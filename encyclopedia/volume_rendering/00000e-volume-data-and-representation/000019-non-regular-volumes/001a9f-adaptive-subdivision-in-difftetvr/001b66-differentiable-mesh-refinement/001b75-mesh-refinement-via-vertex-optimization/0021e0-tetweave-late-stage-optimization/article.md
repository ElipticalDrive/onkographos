# TetWeave Late-Stage Optimization


## Overview
TetWeave Late-Stage Optimization is a differentiable refinement phase that adjusts the Signed Distance Field (SDF) values, point positions, and spherical harmonics (SH) coefficients of an existing tetrahedral mesh without modifying its topological structure [S1, unofficial]. It distinguishes itself from earlier stages by decoupling geometry optimization from grid updates, relying instead on multi-view rendering losses (mask, depth, normal) and SDF regularization to polish the reconstruction [S1, unofficial]. This method builds on [Mesh Refinement via Vertex Optimization](989-mesh-refinement-via-vertex-optimization.md) by operating on the final mesh representation derived from the optimized points and SDF [S1, unofficial].

## How it works
**Inputs:**
- Target multi-view data: masks ($M$), depth maps ($D$), and normal maps ($N$) [S1, unofficial].
- Current state: vertex positions ($V$), face connectivity ($F$), tetrahedra ($T$), SDF values ($S$) at vertices, and spherical harmonics coefficients ($C_{SH}$) [S1, unofficial].
- Configuration parameters: loss weights, optimization steps, and SH degree [S1, unofficial].

**Outputs:**
- Refined vertex positions ($V'$), SDF values ($S'$), and SH coefficients ($C_{SH}'$) [S1, unofficial].
- The tetrahedral topology ($T$) remains unchanged [S1, unofficial].

**Data Structures:**
- `vertices`: Tensor of shape $(V, 3)$, float32.
- `faces`: Tensor of shape $(F, 3)$, int64, defining the surface mesh.
- `tetrahedra`: Tensor of shape $(T, 4)$, int64, defining the volume mesh.
- `sdf`: Tensor of shape $(V,)$, float32, SDF values at vertices.
- `sh_coefficients`: Tensor of shape $(V, K)$, float32, where $K$ is the number of SH coefficients based on the degree.
- `buffers`: Dictionary containing rendered `mask`, `depth`, and `normals_face` tensors.

**Steps:**

1.  **Initialization and Mesh Creation:**
    Construct a surface mesh object from the current vertices and faces [S1, unofficial]. Extract unique edges from the tetrahedra for SDF regularization [S1, unofficial].
    ```python
    def get_edges(tetrahedra):
        edges = []
        for i in range(4):
            for j in range(i + 1, 4):
                edges.append(tetrahedra[:, [i, j]])
        return torch.unique(torch.cat(edges, dim=0), dim=0)

    kal_mesh = kal.rep.SurfaceMesh(vertices, faces)
    edges = get_edges(tetrahedra)
    ```

2.  **Rendering:**
    Render the surface mesh to the target camera views to obtain buffers [S1, unofficial].
    ```python
    # [G] Actual differentiable rasterization implementation returns requested buffers
    buffers = render.render_mesh(
        kal_mesh, 
        target_cameras, 
        self.conf.optimization.train_res, 
        return_types=self.return_types, 
        white_bg=False
    )
    ```

3.  **Loss Calculation:**
    Compute individual loss components weighted by configuration constants [S1, unofficial].
    - **Mask Loss:** Mean absolute difference between rendered and target masks [S1, unofficial].
      $$ L_{mask} = \left( \frac{1}{N} \sum |M_{rendered} - M_{target}| \right) \cdot w_{mask} $$
      Where $w_{mask}$ is `optimization.late_stage.loss.mask_reg` [S1, unofficial].
    - **Depth Loss:** Root mean square error between rendered and target depth, masked by the target mask [S1, unofficial]. An epsilon $\epsilon = 10^{-8}$ is added for numerical stability [S1, unofficial].
      $$ L_{depth} = \text{mean}\left(\sqrt{\text{sum}_{c}\left((D_{rendered} - D_{target}) \cdot M_{target}\right)^2 + \epsilon}\right) \cdot w_{depth} $$
      Where $w_{depth}$ is `optimization.late_stage.loss.depth_reg` [S1, unofficial].
    - **Normal Loss:** Root mean square error between rendered face normals and target normals, masked by the target mask [S1, unofficial].
      $$ L_{normal} = \text{mean}\left(\sqrt{\text{sum}_{c}\left((N_{rendered} - N_{target}) \cdot M_{target}\right)^2 + \epsilon}\right) \cdot w_{normal} $$
      Where $w_{normal}$ is `optimization.late_stage.loss.normal_reg` [S1, unofficial].
    - **SDF Regularization Loss:** Computes the SDF regularization loss on the edges (pairs of vertices) [S1, unofficial]. The weight is dynamic, decreasing linearly with iteration progress to prioritize geometric alignment early and smoothness later [S1, unofficial].
      Let $t_{iter} = \frac{iter}{total\_iter}$ [S1, unofficial].
      The dynamic weight $w_{sdf}$ is calculated as:
      $$ w_{sdf} = w_{sdf\_base} - \left( w_{sdf\_base} - \frac{w_{sdf\_base}}{20} \right) \cdot \min(1.0, 4.0 \cdot t_{iter}) $$
      Where $w_{sdf\_base}$ is `optimization.late_stage.loss.sdf_reg` [S1, unofficial].
      The loss is:
      $$ L_{sdf} = \text{mean}( \text{sdf\_reg\_loss}(S, edges) ) \cdot w_{sdf} $$
      Where `sdf_reg_loss` computes a regularization term over edge pairs [S1, unofficial].

4.  **Total Loss and Optimization Step:**
    Sum the losses and perform the backward pass [S1, unofficial].
    $$ L_{total} = L_{mask} + L_{depth} + L_{normal} + L_{sdf} $$
    Update the optimizers and schedulers for both SDF and SH coefficients [S1, unofficial].
    ```python
    total_loss = mask_loss + depth_loss + normal_loss + reg_sdf_loss
    total_loss.backward()
    optimizer_sdf.step()
    scheduler_sdf.step()
    optimizer_sh.step()
    scheduler_sh.step()
    ```

**Parameters:**
- `optimization.late_stage.loss.mask_reg`: Weight for mask loss [S1, unofficial].
- `optimization.late_stage.loss.depth_reg`: Weight for depth loss [S1, unofficial].
- `optimization.late_stage.loss.normal_reg`: Weight for normal loss [S1, unofficial].
- `optimization.late_stage.loss.sdf_reg`: Base weight for SDF regularization [S1, unofficial].
- `spherical_harmonics.degree`: Degree of the spherical harmonics used for appearance [S1, unofficial].
- `optimization.train_res`: Resolution of the rendered buffers [S1, unofficial].
- `accumulation_delaunay`: Number of steps before two updates of the Tetrahedral grid. Default value is 5. Setting to 1 updates Delaunay triangulation every step, slowing optimization [S1, unofficial].
- `resampling.aggregation`: Method for computing SDF and SH coefficients of newly sampled points. Options: `interpolation` (barycentric interpolation) or `zero` (initialize to 0) [S1, unofficial].
- `resampling.sampling_method`: Method to compute density for resampling. Options: `error_density_gt` (uses error between target's rendering and current reconstruction's rendering) or `functional_density` (uses explicit function, default constant) [S1, unofficial].
- `resampling.voxel_resolution`: Resolution of the voxel grid used to estimate the density function during resampling [S1, unofficial].
- `resampling.steps`: Iterations at which resampling occurs and the number of points for each step. Number of points cannot decrease during optimization [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Topology Freeze:** The tetrahedral grid topology is not updated during this phase [S1, unofficial]. ODT (Oriented Delaunay Tetrahedralization) loss and fairness loss are not used because the grid is not updated [S1, unofficial].
- **Numerical Stability:** Epsilon ($10^{-8}$) is added to depth and normal losses to prevent division by zero or NaN gradients during the square root operation [S1, unofficial].
- **Convergence:** The dynamic SDF weight ensures that SDF regularization does not dominate the early stages of refinement where geometric alignment (mask/depth/normal) is critical [S1, unofficial].

## Subtopics

## Operating characteristics
Not given in the sources.

## Background
TetWeave Late-Stage Optimization is a late-stage optimization phase for TetWeave that refines the Signed Distance Field (SDF), point positions, and spherical harmonics coefficients without updating the tetrahedral grid topology, using mask, depth, normal, and SDF regularization losses [S1, unofficial]. It was introduced as part of the broader TetWeave framework to address the need for precise surface alignment after the coarse geometry and topology have been established [S1, unofficial]. The method leverages the differentiability of the rendering process to backpropagate errors from 2D observations to 3D vertex positions, SDF values, and appearance coefficients [S1, unofficial].

## Sources
[S1] AlexandreBinninger/TetWeave, https://github.com/AlexandreBinninger/TetWeave, fetched 2026-10-27 (unofficial)
