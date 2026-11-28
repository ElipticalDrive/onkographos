# Gradient-Based Mesh Optimization via Spherical Harmonics

Also called: Geometric texture generation, Paparazzi-style optimization

## Overview
This method optimizes high-frequency surface details of a mesh by backpropagating image-space gradients into spherical harmonic (SH) coefficients within a TetWeave representation [S1]. It solves the problem of generating geometric textures or refining mesh details without altering the coarse topology or significantly deviating from a reference shape [S1]. Unlike direct vertex displacement, it adjusts normals to tilt surface details, making it suitable for applications like photogrammetry refinement or artistic texture generation [S1].

## How it works
**Inputs:**
- A reference mesh or coarse shape $M_{ref}$ [S1].
- A TetWeave representation initialized with a point cloud, SDF values, and SH coefficients [S1].
- An image filter $F$ (e.g., Guided filter, L0-Smoothing, or K-means quantization) [S1].
- A loss function $L$ combining mask, depth, and normal map differences against a target [S1].

**Outputs:**
- An optimized TetWeave representation with updated SH coefficients [S1].

**Data Structures:**
- **TetWeave Parameters:**
  - Point positions $\mathbf{p}_i$ (fixed after initial fit) [S1].
  - SDF values $s_i$ (fixed after initial fit) [S1].
  - Spherical Harmonic coefficients $\mathbf{c}_{l,m}$ for each point or local region [S1].
- **SH Degree:**
  - Degree $L=2$ for initial fitting [S1].
  - Degree $L=1$ for refinement stages [S1].

**Steps:**
1. **Initialization:** Fit the TetWeave representation to the reference shape $M_{ref}$ using SH of degree 2. Optimize point positions and SDF values to minimize reconstruction error [S1].
2. **Freeze Geometry:** Once the fit is complete, fix the point positions $\mathbf{p}_i$ and SDF values $s_i$. Only SH coefficients $\mathbf{c}_{l,m}$ remain trainable [S1].
3. **Reference Update:** Periodically update the reference shape with the current mesh during optimization to avoid being constrained too much by the initial reference [S1].
4. **Rendering:** Render the normal map $\mathbf{N}_{render}$ of the current mesh $\tilde{M}$ from a randomly sampled camera view [S1].
5. **Filtering:** Apply the chosen image filter $F$ to the rendered normal map to obtain $\mathbf{N}_{filtered} = F(\mathbf{N}_{render})$ [S1].
6. **Gradient Computation:** Compute the gradient as the difference between the filtered and rendered normal maps: $\nabla = \mathbf{N}_{filtered} - \mathbf{N}_{render}$ [S1].
7. **Backpropagation:** Backpropagate the gradient through the differentiable renderer and TetWeave representation to update only the SH coefficients $\mathbf{c}_{l,m}$ [S1].
8. **Loss Calculation:** Compute a loss $L$ on the mask, depth map, and normal map compared to the reference mesh to ensure the generated shape remains close to the reference and does not degenerate [S1].
9. **Iteration:** Repeat steps 4-8 until convergence or a specified number of iterations [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_geometric_texture(tetweave_model, reference_mesh, image_filter, num_iterations):
    # Step 1: Initial fit with degree 2 SH
    tetweave_model.fit_to_reference(reference_mesh, sh_degree=2)
    
    # Step 2: Fix point positions and SDF values
    tetweave_model.freeze_geometry()
    
    # Step 3: Set SH degree to 1 for refinement
    tetweave_model.set_sh_degree(1)
    
    for iteration in range(num_iterations):
        # Step 3: Periodically update reference
        if iteration % update_frequency == 0:
            reference_mesh = tetweave_model.get_current_mesh()
        
        # Step 4: Render normal map from random camera
        camera = sample_random_camera()
        normal_map = tetweave_model.render_normal_map(camera)
        
        # Step 5: Apply image filter
        filtered_normal_map = image_filter(normal_map)
        
        # Step 6: Compute gradient
        gradient = filtered_normal_map - normal_map
        
        # Step 7: Backpropagate to update SH coefficients
        loss = compute_loss(normal_map, gradient, reference_mesh)
        loss.backward()
        tetweave_model.optimizer.step()
        tetweave_model.optimizer.zero_grad()
        
    return tetweave_model
```

**Parameters:**
- **SH Degree:** 2 for initial fitting, 1 for refinement [S1].
- **Image Filters:** Guided filter [He et al. 2013], L0-Smoothing [Xu et al. 2011], or K-means image quantization [MacQueen 1967] [S1].

**Pitfalls:**
- Adjusting SH coefficients primarily tilts normals to affect high-frequency details without deviating significantly from the reference shape [S1].
- Ensure the reference shape is updated periodically to avoid stagnation [S1].

## Operating characteristics
- **Time Complexity:** Depends on the number of iterations, camera samples, and filter complexity. Differentiable rendering and backpropagation are computationally expensive [S1].
- **Memory Complexity:** Low, as point positions and SDF values are fixed after initialization. Only SH coefficients are stored and updated [S1].
- **Strengths:** Efficiently generates high-frequency geometric textures without altering coarse geometry. Supports user-defined energies (e.g., CLIP loss, developability energy) via automatic differentiation [S1].
- **Limitations:** May not recover large-scale geometric changes. Performance depends on the quality of the initial fit and the choice of image filter [S1].
- **When to Use:** Geometric texture generation, photogrammetry refinement, artistic mesh editing [S1].
- **When Not to Use:** When significant topological or large-scale geometric changes are required [S1].

## Background
This method is part of the TetWeave pipeline, which uses on-the-fly Delaunay tetrahedral grids for gradient-based mesh optimization [S1]. It builds on the Paparazzi method for geometric texture generation [Liu et al. 2018] and integrates with differentiable rendering frameworks like NVDiffRec [Munkberg et al. 2022] for photogrammetry [S1]. The approach leverages spherical harmonics to represent high-frequency details, allowing for efficient optimization via backpropagation [S1].

## Sources
[S1] TetWeave: Isosurface Extraction using On-The-Fly Delaunay Tetrahedral Grids for Gradient-Based Mesh Optimization, https://arxiv.org/abs/2505.04590v2, fetched 2026-10-26
