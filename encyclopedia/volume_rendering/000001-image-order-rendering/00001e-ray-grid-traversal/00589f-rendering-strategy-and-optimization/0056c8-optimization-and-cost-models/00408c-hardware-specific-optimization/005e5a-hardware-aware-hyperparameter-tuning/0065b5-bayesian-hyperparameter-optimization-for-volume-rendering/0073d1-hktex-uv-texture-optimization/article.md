# HKTex UV-texture Optimization

Also called: Optimise existing UV-textures, UV-texture Optimization

## Overview
HKTex UV-texture Optimization is a method for compressing existing ground-truth UV textures by fitting Heat Kernel Texture (HKTex) parameters to minimize photometric error. It solves the problem of reducing storage footprint for 3D mesh textures while maintaining high perceptual quality. It differs from neural baselines (MLPs, InstantNGP) and explicit 2D baselines (downsampled images, ImageGS) by using intrinsic geometric kernels, achieving superior structural preservation (SSIM, MS-SSIM) and lower storage costs for high-vertex-count meshes where UV-coordinate overhead dominates other methods [S1].

## How it works
**Inputs:**
- A mesh $M$ with a ground-truth albedo UV texture [S1].
- The mesh geometry (vertices, faces) to compute geodesic distances and heat kernels [S1].
- Hyperparameters for the HKTex model (kernel locations, pruning/densification thresholds) obtained via Bayesian optimization [S1].

**Outputs:**
- Optimized HKTex parameters (kernel locations and values) that approximate the ground-truth UV texture [S1].

**Data Structures:**
- HKTex Model: Defined by a set of kernels on the mesh surface. Each kernel is characterized by its location and associated heat kernel weights [S1].
- Ground-Truth Texture: A 2D image mapped to the mesh via UV coordinates [S1].

**Steps:**
1. **Initialization:** Initialize HKTex parameters. Kernel locations are optimized using a geodesic Riemannian-SGD algorithm on the GPU [S1].
2. **Sampling:** At each optimization step, uniformly sample a dense set of points $p$ directly on the faces of mesh $M$ [S1].
3. **Evaluation:** Evaluate the HKTex color $C(p)$ at these sampled points using the current HKTex model [S1].
4. **Ground-Truth Query:** Query the ground-truth UV texture color $C_{GT}(p)$ at the same spatial locations $p$ [S1].
5. **Loss Calculation:** Compute the photometric loss:
   $$L = \|C(p) - C_{GT}(p)\|_2^2$$
   where $C(p)$ is the HKTex color and $C_{GT}(p)$ is the ground-truth color [S1].
6. **Optimization:** Update HKTex parameters to minimize $L$.
   - Use an Adam optimizer with a step-function learning rate scheduler [S1].
   - For kernel locations, use geodesic Riemannian-SGD [S1].
   - Apply importance sampling during fitting [S1].
7. **Hyperparameter Tuning:** Use Optuna with a Tree-Structured Parzen Estimator (TPE) to tune hyperparameters (e.g., pruning/densification thresholds) on a validation set [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_hktex_uv_texture(mesh, gt_uv_texture, hyperparameters):
    """
    Optimizes HKTex parameters to fit a ground-truth UV texture.
    
    Args:
        mesh: The 3D mesh object with geometry and UV map.
        gt_uv_texture: The ground-truth albedo UV texture image.
        hyperparameters: Dict containing HKTex model settings and optimizer configs.
        
    Returns:
        optimized_hktex_params: The optimized HKTex parameters.
    """
    # Initialize HKTex model with hyperparameters
    hktex_model = initialize_hktex(mesh, hyperparameters)
    
    # Setup optimizer
    # Note: Geodesic Riemannian-SGD is used for kernel locations [S1]
    # Adam is used for other parameters [S1]
    optimizer = setup_optimizer(hktex_model, hyperparameters['optimizer_config'])
    
    # Bayesian Hyperparameter Tuning (Outer Loop)
    # Uses Optuna with TPE [S1]
    best_params = bayesian_tuning(
        objective_function=lambda params: evaluate_loss(mesh, gt_uv_texture, params),
        search_space=hyperparameters['search_space'],
        n_trials=hyperparameters['n_trials']
    )
    
    # Final Optimization with best hyperparameters
    hktex_model.set_parameters(best_params)
    
    for step in range(hyperparameters['num_steps']):
        # 1. Sample points on mesh faces [S1]
        points = uniform_sample_points_on_mesh_faces(mesh, hyperparameters['num_samples'])
        
        # 2. Evaluate HKTex color at points [S1]
        c_p = evaluate_hktex_color(hktex_model, points)
        
        # 3. Query ground-truth color at same locations [S1]
        # Requires mapping 3D points to UV coordinates
        uv_coords = project_points_to_uv(mesh, points)
        c_gt_p = sample_uv_texture(gt_uv_texture, uv_coords)
        
        # 4. Compute Loss [S1]
        loss = mean_squared_error(c_p, c_gt_p)
        
        # 5. Backpropagate and Update [S1]
        # Gradients backpropagated from loss to HKTex parameters [S1]
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # Apply importance sampling if configured [S1]
        if hyperparameters['use_importance_sampling']:
            apply_importance_sampling(points, loss, hktex_model)
            
    return hktex_model.get_parameters()
```

**Parameters:**
- **Optimizer:** Adam with step-function learning rate scheduler [S1].
- **Kernel Location Optimizer:** Geodesic Riemannian-SGD on GPU [S1].
- **Bayesian Optimization:** Optuna with Tree-Structured Parzen Estimator (TPE) [S1].
- **Importance Sampling:** Used during fitting [S1].
- **Rendering/Differentiation:** Mitsuba3 with Dr.Jit, path-replay backpropagation (PRB) integrator, ray depth 3 [S1].

**Edge Cases and Pitfalls:**
- **High Vertex Count:** For meshes with high vertex counts, the fixed memory overhead of per-vertex UV coordinates can dominate the storage budget, making it difficult for explicit 2D baselines to match HKTex's efficiency [S1].
- **Storage Budget Matching:** Neural baselines require grid search over network width, depth, and hash-grid size to exactly match HKTex's storage cost for fair comparison [S1].
- **Degenerate Input:** The method assumes a valid UV map is available. If the UV map is missing, this specific optimization path is not applicable (though multi-view optimization exists) [S1].

## Subtopics

### Fitting UV-textures Baselines
This section details the comparative baselines used to evaluate HKTex UV-texture optimization [S1].
- **Neural Baselines:**
  - **MLP with Fourier Positional Encoding:** Standard MLP using traditional Fourier encoding .
  - **InstantNGP Adaptation:** Uses hash-grid encoding .
  - **Intrinsic Neural Fields:** Barycentrically interpolates eigenvectors of the Laplace-Beltrami Operator (LBO) .
  - *Note:* Grid search is performed over network width, depth, and hash-grid size to exactly match HKTex's storage cost [S1].
- **Explicit 2D Baselines:**
  - **Downsampled Ground-Truth Images:** Lower resolution versions of the original UV texture [S1].
  - **ImageGS:** A 2D Gaussian Splatting approach .
  - *Note:* Both rely on UV-mapping. For high-vertex-count meshes, the UV-coordinate overhead dominates, leading to catastrophic performance penalties if storage is constrained [S1].
- **Vertex Color Baselines:**
  - **GT VTex:** Vertex colors sampled directly from GT UV-textures at native mesh resolution [S1].
  - **HR GT VTex:** Iteratively subdivided mesh to match HKTex's storage budget [S1].
- **Performance:** HKTex achieves the lowest storage footprint after GT VTex, with competitive PSNR, best SSIM/MS-SSIM/MSE, and second-best LPIPS [S1]. Neural baselines and HR GT VTex require significantly larger storage for comparable accuracy [S1].

## Operating characteristics
- **Storage Efficiency:** HKTex achieves significantly reduced storage footprint compared to neural and explicit 2D baselines while maintaining competitive reconstruction quality [S1]. It yields the best SSIM, MS-SSIM, and MSE among compressed methods [S1].
- **Computational Cost:** Training involves differentiable ray tracing via Mitsuba3/Dr.Jit, which is computationally intensive. However, inference is fast as the KNN database and heat kernels are pre-computed once [S1].
- **Scalability:** Performance degrades for explicit 2D baselines on high-vertex-count meshes due to UV-coordinate overhead, whereas HKTex scales better by leveraging intrinsic geometry [S1].
- **Strengths:** High perceptual quality (LPIPS), structural preservation (SSIM), and low storage cost. Effective for meshes with available UV maps [S1].
- **Limitations:** Requires a ground-truth UV texture. Computationally expensive training due to differentiable rendering and geodesic optimization. Not applicable if UV map is missing (use multi-view optimization instead) [S1].

## Background
- **Origin:** Heat Kernel Textures (HKTex) was introduced by Foti et al. in "Heat Kernel Textures the Geodesic Gaussians That Do Not Splat" (2025) [S1].
- **Development:** The method builds on heat kernel theory to represent textures intrinsically on meshes. The UV-texture optimization variant was developed to compress existing UV textures by fitting HKTex parameters to ground-truth albedo [S1].
- **Usage:** Used in 3D computer graphics for texture compression and representation, particularly on datasets like Objaverse [S1].

## Sources
[S1] Heat Kernel Textures the Geodesic Gaussians That Do Not Splat, https://doi.org/10.1007/978-3-032-37595-7_17, fetched 2026-10-29
