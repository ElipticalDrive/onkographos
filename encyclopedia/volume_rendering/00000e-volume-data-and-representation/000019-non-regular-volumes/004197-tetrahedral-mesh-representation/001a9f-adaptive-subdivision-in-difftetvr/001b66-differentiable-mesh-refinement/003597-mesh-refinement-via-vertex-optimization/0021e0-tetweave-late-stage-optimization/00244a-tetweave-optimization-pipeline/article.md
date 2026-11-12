# TetWeave Optimization Pipeline

Also called: TetWeave implicit surface reconstruction, TetWeave training

## Overview
The TetWeave Optimization Pipeline is a differentiable optimization method that reconstructs a target 3D mesh from 2D multi-view images (mask, depth, normals) by optimizing a set of 3D points, their Signed Distance Field (SDF) values, and spherical harmonic (SH) coefficients [S1, unofficial]. It builds on [TetWeave Late-Stage Optimization](2583-tetweave-late-stage-optimization.md) by including the initial phase where point positions and tetrahedral topology are also updated [S1, unofficial]. It is distinguished from sibling methods like [Implicit Surface Optimization](#implicit-surface-optimization) by its explicit use of tetrahedralization and marching tetrahedra for mesh extraction, rather than direct implicit surface parameterization [S3].

## How it works
The pipeline optimizes three parameter sets: point positions, SDF values, and SH coefficients. It operates in two stages: a main stage where topology and geometry are updated, and a late stage where only the scalar field and features are refined [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- Target multi-view images: mask, depth, and normal maps [S1, unofficial].
- Camera parameters for the target views [S1, unofficial].
- Configuration parameters: `accumulation_delaunay`, `spherical_harmonics.degree`, `resampling_dict`, `sampling_method`, and learning rates [S1, unofficial].

**Outputs:**
- Optimized point positions `x_nx3` [S1, unofficial].
- Optimized SDF values `sdf` [S1, unofficial].
- Optimized SH coefficients `sh_coefficients` [S1, unofficial].
- Final mesh defined by vertices and faces, extracted via marching tetrahedra [S1, unofficial].

### Data Structures
The parameters are stored as PyTorch `nn.Parameter` tensors [S1, unofficial]:
- `x_nx3`: Shape `(N, 3)`, stores 3D coordinates of $N$ points. `requires_grad=True` [S1, unofficial].
- `sdf`: Shape `(N,)`, stores scalar field values at each point. `requires_grad=True` [S1, unofficial].
- `sh_coefficients`: Shape `(N, num_coeff)`, stores SH coefficients per point. `num_coeff = (degree + 1)**2`. `requires_grad=True` [S1, unofficial].
- `tetrahedra`: Tensor of shape `(T, 4)`, where $T$ is the number of tetrahedra, defining the mesh topology [S1, unofficial].

### Parameters
- `accumulation_delaunay`: Integer. Frequency (in iterations) to re-tetrahedralize points. Controls how often the mesh topology is updated [S1, unofficial].
- `spherical_harmonics.degree`: Integer. Degree of SH used for color/feature encoding. Must be between 0 and 4 [S1, unofficial].
- `resampling_dict`: Dictionary mapping iteration numbers to target point counts. Triggers resampling when `iter` matches a key [S1, unofficial].
- `sampling_method`: String. Resampling strategy. Options: `"functional_density"` or `"error_density_gt"` [S1, unofficial].
- `lr_sdf`, `lr_x_nx3`, `lr_sh`: Learning rates for AdamW optimizers for SDF, points, and SH coefficients respectively [S1, unofficial].

### Steps
1. **Initialization**: Initialize `x_nx3` with $N$ points in a ball of radius $\sqrt{3}$. Initialize `sdf` with random values. Initialize `sh_coefficients` with shape `(N, (degree + 1)**2)` [S1, unofficial]. Create AdamW optimizers for each parameter set with specified learning rates [S1, unofficial].
2. **Main Stage Loop**: Run for `conf.optimization.main_stage.iter` iterations [S1, unofficial].
    a. Fetch target mask, depth, and normals for the current batch [S1, unofficial].
    b. If `iter % accumulation_delaunay == 0`, call `tetrahedralize(x_nx3)` to compute new tetrahedra from the current point positions [S1, unofficial].
    c. Otherwise, use the existing `tetrahedra` [S1, unofficial].
    d. Call `marching_tetrahedra(x_nx3, tetrahedra, sdf, sh_coefficients, sh_deg)` to extract the mesh vertices and faces [S1, unofficial].
    e. Render the mesh using `render_mesh` to obtain buffers: mask, depth, and normals [S1, unofficial].
    f. Compute losses (see Formulas below) [S1, unofficial].
    g. Perform backward pass to update SDF, SH, and point positions. Note: Point positions are updated every iteration, but topology is only recomputed every `accumulation_delaunay` iterations [S1, unofficial].
    h. If `iter` is in `resampling_dict`, resample points to increase count using the specified `sampling_method` [S1, unofficial].
3. **Late Stage Loop**: Run for remaining iterations [S1, unofficial].
    a. Fix point positions `x_nx3` and tetrahedral grid `tetrahedra` [S1, unofficial].
    b. Only optimize `sdf` and `sh_coefficients` [S1, unofficial].
    c. Compute mesh via `marching_tetrahedra` and render for loss computation [S1, unofficial].

### Formulas
Let $t_{iter} = \frac{iter}{total\_iter}$ [S1, unofficial].

**Mask Loss:**
$$ L_{mask} = \text{mean}(|\text{buffers['mask']} - \text{target\_mask}|) \times \text{mask\_reg} $$

**Depth Loss:**
$$ L_{depth} = \text{mean}\left(\sqrt{\sum_{c} ((\text{buffers['depth']} - \text{target\_depth}) \times \text{target\_mask})^2 + 10^{-8}}\right) \times \text{depth\_reg} $$
where the sum is over channels $c$ [S1, unofficial].

**Normal Loss:**
$$ L_{normal} = \text{mean}\left(\sqrt{\sum_{c} ((\text{buffers['normals\_face']} - \text{target\_normals}) \times \text{target\_mask})^2 + 10^{-8}}\right) \times \text{normal\_reg} $$
where the sum is over channels $c$ [S1, unofficial].

**SDF Regularization Weight:**
$$ w_{sdf} = \text{sdf\_reg} - \left(\text{sdf\_reg} - \frac{\text{sdf\_reg}}{20}\right) \times \min(1.0, 4.0 \times t_{iter}) $$

**SDF Regularization Loss:**
$$ L_{sdf\_reg} = \text{mean}(\text{loss.sdf\_reg\_loss}(self.sdf, \text{edges.reshape}(-1, 2))) \times w_{sdf} $$
where `edges` are derived from the tetrahedra [S1, unofficial].

**ODT Loss:**
$$ L_{ODT} = \text{loss.E\_ODT}(self.x\_nx3, \text{tetrahedra}) \times \text{ODT\_reg} $$

**Triangle Angle Fairness Loss:**
$$ L_{angle} = \text{loss.triangle\_angle\_fairness}(\text{vertices}, \text{faces}) \times \text{triangle\_angle\_fairness\_reg} $$

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def tetweave_optimization_pipeline(conf, targets, target_cameras):
    """
    Main optimization loop for TetWeave.
    """
    # 1. Initialization
    N = conf.initialization.num_points
    degree = conf.spherical_harmonics.degree
    num_coeff = (degree + 1)**2
    
    # Initialize points in a ball of radius sqrt(3)
    x_nx3 = nn.Parameter(torch.randn(N, 3) * (3**0.5))
    sdf = nn.Parameter(torch.randn(N))
    sh_coefficients = nn.Parameter(torch.randn(N, num_coeff))
    
    # Create optimizers
    optimizer_x = optim.AdamW([x_nx3], lr=conf.optimization.lr_x_nx3)
    optimizer_sdf = optim.AdamW([sdf], lr=conf.optimization.lr_sdf)
    optimizer_sh = optim.AdamW([sh_coefficients], lr=conf.optimization.lr_sh)
    
    # Initialize tetrahedra (e.g., via initial Delaunay)
    tetrahedra = compute_initial_tetrahedra(x_nx3)
    
    total_main_iters = conf.optimization.main_stage.iter
    accumulation_delaunay = conf.optimization.main_stage.accumulation_delaunay
    
    for iter in range(total_main_iters):
        # Fetch targets
        target_mask = targets["mask"]
        target_depth = targets["depth"]
        target_normals = targets["normals_face"]
        
        # Topology update check
        if iter % accumulation_delaunay == 0:
            # Re-tetrahedralize
            tetrahedra = tetrahedralize(x_nx3)
        
        # Marching Tetrahedra
        vertices, faces = marching_tetrahedra(x_nx3, tetrahedra, sdf, sh_coefficients, degree)
        
        # Render
        buffers = render_mesh(vertices, faces, target_cameras)
        
        # Compute Losses
        # Mask Loss
        mask_loss = torch.abs(buffers['mask'] - target_mask).mean() * conf.loss.mask_reg
        
        # Depth Loss
        depth_diff = (buffers["depth"] - target_depth) * target_mask
        depth_loss = torch.sqrt((depth_diff**2).sum(dim=-1) + 1e-8).mean() * conf.loss.depth_reg
        
        # Normal Loss
        normal_diff = (buffers['normals_face'] - target_normals) * target_mask
        normal_loss = torch.sqrt((normal_diff**2).sum(dim=-1) + 1e-8).mean() * conf.loss.normal_reg
        
        # SDF Regularization
        t_iter = iter / total_main_iters
        sdf_weight = conf.loss.sdf_reg - (conf.loss.sdf_reg - conf.loss.sdf_reg/20) * min(1.0, 4.0 * t_iter)
        edges = get_edges_from_tetrahedra(tetrahedra)
        reg_sdf_loss = loss.sdf_reg_loss(sdf, edges.reshape(-1, 2)).mean() * sdf_weight
        
        # ODT Loss
        odt_loss = loss.E_ODT(x_nx3, tetrahedra) * conf.loss.ODT_reg
        
        # Triangle Angle Fairness
        triangle_angle_fairness_loss = loss.triangle_angle_fairness(vertices, faces) * conf.loss.triangle_angle_fairness_reg
        
        total_loss = mask_loss + depth_loss + normal_loss + reg_sdf_loss + odt_loss + triangle_angle_fairness_loss
        
        # Backward Pass
        optimizer_x.zero_grad()
        optimizer_sdf.zero_grad()
        optimizer_sh.zero_grad()
        total_loss.backward()
        
        optimizer_x.step()
        optimizer_sdf.step()
        optimizer_sh.step()
        
        # Resampling
        if iter in conf.optimization.resampling_dict:
            target_count = conf.optimization.resampling_dict[iter]
            sampling_method = conf.optimization.sampling_method
            x_nx3, sdf, sh_coefficients = resample_points(x_nx3, sdf, sh_coefficients, tetrahedra, target_count, sampling_method)
            # Reassign optimizer param groups if necessary to handle new tensors
            # [G] Implementation detail: Optimizers need to be updated with new parameter tensors if resampling creates new tensors.
            
    # Late Stage
    late_stage_iters = conf.optimization.total_iter - total_main_iters
    for iter in range(late_stage_iters):
        # Fix x_nx3 and tetrahedra
        x_nx3.requires_grad = False
        
        # Marching Tetrahedra
        vertices, faces = marching_tetrahedra(x_nx3, tetrahedra, sdf, sh_coefficients, degree)
        
        # Render
        buffers = render_mesh(vertices, faces, target_cameras)
        
        # Compute Losses (same as above, but without ODT/Angle fairness if they depend on topology/positions, 
        # though SDF reg and rendering losses remain)
        # [G] Assuming ODT and Angle fairness are not computed in late stage as per "Fix point positions and tetrahedral grid"
        # and "Only optimize SDF and SH coefficients".
        
        mask_loss = torch.abs(buffers['mask'] - target_mask).mean() * conf.loss.mask_reg
        depth_diff = (buffers["depth"] - target_depth) * target_mask
        depth_loss = torch.sqrt((depth_diff**2).sum(dim=-1) + 1e-8).mean() * conf.loss.depth_reg
        normal_diff = (buffers['normals_face'] - target_normals) * target_mask
        normal_loss = torch.sqrt((normal_diff**2).sum(dim=-1) + 1e-8).mean() * conf.loss.normal_reg
        
        t_iter = iter / late_stage_iters # Or relative to total? Source says t_iter = iter / total_iter in main stage.
        # [G] Assuming t_iter continues to be calculated relative to total iterations for consistency, 
        # or reset. Source formula uses t_iter = iter / total_iter.
        t_iter_total = (total_main_iters + iter) / conf.optimization.total_iter
        sdf_weight = conf.loss.sdf_reg - (conf.loss.sdf_reg - conf.loss.sdf_reg/20) * min(1.0, 4.0 * t_iter_total)
        edges = get_edges_from_tetrahedra(tetrahedra)
        reg_sdf_loss = loss.sdf_reg_loss(sdf, edges.reshape(-1, 2)).mean() * sdf_weight
        
        total_loss = mask_loss + depth_loss + normal_loss + reg_sdf_loss
        
        optimizer_sdf.zero_grad()
        optimizer_sh.zero_grad()
        total_loss.backward()
        
        optimizer_sdf.step()
        optimizer_sh.step()
        
        x_nx3.requires_grad = True # Reset for next potential run or cleanup
        
    return x_nx3, sdf, sh_coefficients, vertices, faces
```

### Edge Cases and Pitfalls
- **NaN/Inf in Losses**: Depth and normal losses add $10^{-8}$ before the square root to prevent NaN/inf from zero denominators or negative values [S1, unofficial].
- **Resampling**: Resampling dynamically increases point count by concatenating new tensors. Optimizer param groups must be updated to avoid stale gradients [S1, unofficial].
- **Symmetry**: If initialized in a symmetric configuration (e.g., center of a torus), a rotated camera view may be necessary to break symmetry and avoid local minima [S3].

## Subtopics

### Voromesh Co-planarity Prevention
Applies a very small Centroidal Voronoi Tessellation (CVT) regularization term during mesh extraction to prevent co-planarity of sites [S2]. This is necessary because unconverged SDF representations frequently produce co-planar configurations in the final site distribution, which are not supported by gDel3D [S2]. This increases runtime because Delaunay triangulation must be recomputed at every iteration [S2]. CGAL can be used as an alternative to gDel3D for mesh extraction to handle co-planar configurations [S2].

### Implicit Surface Optimization
Fits an image of a torus using an implicit surface parameterized by spheres, demonstrating handling of topology changes [S3]. Two parameterizations are shown: Swept Sphere (ring radius $r_2$, sphere radius $r_1$) and Random Spheres (union of 200 spheres with individual radii and 2-D positions) [S3]. The loss is mean-absolute error between rendering and target, optimized with Adam for 400 iterations [S3]. Unlike MeshSDF, this method does not rely on a deep neural network to compute gradients, allowing any implicit surface parameterization that can be evaluated on a grid [S3].

## Operating characteristics
- **Time Complexity**: The main stage involves iterative tetrahedralization and marching tetrahedra, which are computationally expensive. Re-tetrahedralization every `accumulation_delaunay` iterations adds significant overhead [S1, unofficial]. Voromesh co-planarity prevention further increases runtime by requiring Delaunay triangulation at every iteration [S2].
- **Memory Complexity**: Scales with the number of points $N$ and tetrahedra $T$. Resampling increases $N$ dynamically [S1, unofficial].
- **Strengths**: Handles topology changes effectively [S3]. Uses differentiable rendering for direct image-space supervision [S1, unofficial].
- **Limitations**: Computationally intensive due to repeated mesh extraction and tetrahedralization [S1, unofficial][S2]. Sensitive to initialization symmetry [S3].
- **When to use**: When reconstructing complex 3D meshes from multi-view images with varying topology [S1, unofficial][S3].
- **When not to use**: When real-time performance is critical due to high computational cost [S1, unofficial][S2].

## Background
The TetWeave Optimization Pipeline is part of the TetWeave framework, which uses tetrahedralization and differentiable rendering for 3D reconstruction [S1, unofficial]. The method was developed by Alexandre Binninger [S1, unofficial]. Implicit surface optimization techniques using differentiable rendering have been explored in earlier work, such as the 2021 paper on differentiable surface rendering via non-differentiable sampling [S3].

## Sources
[S1] AlexandreBinninger/TetWeave, https://github.com/AlexandreBinninger/TetWeave, fetched 2026-10-27 (unofficial)
[S2] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
[S3] Differentiable Surface Rendering via Non-Differentiable Sampling, https://doi.org/10.1109/iccv48922.2021.00603, fetched 2026-10-26
