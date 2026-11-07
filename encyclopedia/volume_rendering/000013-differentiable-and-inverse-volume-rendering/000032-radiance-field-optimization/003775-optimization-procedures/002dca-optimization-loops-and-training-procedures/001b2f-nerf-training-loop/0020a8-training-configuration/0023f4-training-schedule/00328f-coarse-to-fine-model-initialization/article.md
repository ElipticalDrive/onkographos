# Coarse-to-Fine Model Initialization

Also called: Coarse-to-Fine Initialization, Progressive Scale Training

## Overview
Coarse-to-Fine Model Initialization accelerates the convergence of dynamic radiance field optimization by leveraging a pre-trained coarse model to initialize a higher-resolution fine model [S1]. It addresses the slow convergence and instability associated with training high-resolution deformation fields from scratch [S1]. Unlike [Hierarchical Spherical Harmonics Optimization](5444-hierarchical-spherical-harmonics-optimization.md), which refines angular resolution, this method refines spatial resolution and deformation complexity by transferring learned features from a low-resolution or simplified representation to a detailed one [S1, S2].

## How it works
The method consists of two distinct phases: a coarse pre-training phase and a fine-model initialization phase. It is primarily applied in dynamic view synthesis contexts where deformation fields are learned [S1].

### Inputs and Outputs
**Inputs:**
- Scene data (images, camera poses, time stamps).
- Coarse model parameters ($\theta_{coarse}, \phi$) optimized during pre-training.
- Target fine model architecture (identical MLP structure for deformation, higher resolution feature grid) [S1].

**Outputs:**
- Initialized fine model parameters ($\theta_{fine}, \phi_{fine}$), where $\phi_{fine}$ is interpolated from $\phi_{coarse}$ [S1].

### Data Structures
**Coarse Deformation Feature Grid:**
- A 3D grid storing deformation features.
- Resolution: Lower than the fine model (exact initial resolution not specified, but subject to progressive doubling) [S1].
- Field: Feature vectors (dimension 44 in the cited implementation) [S1].

**Fine Deformation Feature Grid:**
- A 3D grid storing deformation features.
- Resolution: Higher than the coarse model (initially double the coarse resolution, then doubling further) [S1].
- Field: Feature vectors (dimension 44) [S1].

**MLP Weights:**
- Deformation Network ($F_{\theta_d}$): 4 fully connected layers, width 64. Inputs: position (3), time (1), feature (44) [S1].
- Color Network ($F_{\theta_c}$): 3 fully connected layers, width 128. Inputs: position (3), view direction (27), feature (12) [S1].

### Steps
1. **Pre-train Coarse Model:**
   - Train coarse density network $\sigma_{coarse}$ and color network $c_{coarse}$ for the entire scene for approximately 100,000 iterations [S2].
   - During this phase, optimize the deformation parameters $\theta_{coarse}$ and the decomposition parameters $\phi$ separately [S2].
   - Minimize reconstruction loss $L_{radiance}$ for $\theta_{coarse}$ and uniformity loss $L_{uniform}$ for $\phi$ [S2].
   - **Pitfall:** Do not allow $L_{uniform}$ to affect the optimization of $\theta_{coarse}$, as it inhibits the density network's ability to learn scene structure and causes instability [S2].

2. **Initialize Fine Model:**
   - Copy the trained MLP weights from the coarse deformation module to the fine deformation module, as the architectures are identical [S1].
   - Initialize the fine model's deformation feature grid by interpolating the optimized feature grid from the coarse model [S1].

3. **Progressive Scale Training (Fine Model):**
   - Train the fine model using the initialized parameters.
   - Apply progressive scale training by doubling the voxel resolution of the feature grid after a set number of training iterations [S1].
   - For decomposed models (e.g., DeRF), once $\phi$ is pre-trained, fix $\phi$ and train per-decomposition networks $\sigma_{\theta_n}$ and $c_{\theta_n}$ using only $L_{radiance}$, as $L_{uniform}$ is no longer necessary [S2].

### Formulas
- **Loss Functions:**
  - Reconstruction Loss: $L_{radiance}$ [S2].
  - Uniformity Loss: $L_{uniform}$ [S2].
- **Optimization Objective (Coarse Phase):**
  - $\theta_{coarse} \leftarrow \arg \min L_{radiance}$ [S2].
  - $\phi \leftarrow \arg \min L_{uniform}$ [S2].

### Parameters
- **Pre-training Iterations:** Approximately 100,000 iterations for the coarse network and decomposition pre-training phase [S2].
- **MLP Widths:** Deformation network width = 64; Color network width = 128 [S1].
- **Feature Dimensions:** Deformation feature input dimension = 44; Color feature input dimension = 12 [S1].

### Edge Cases and Pitfalls
- **Instability from Joint Optimization:** Allowing the uniformity loss $L_{uniform}$ to influence the density network parameters $\theta_{coarse}$ during pre-training inhibits the learning of scene structure and leads to unstable training [S2].
- **Architecture Mismatch:** The method relies on the coarse and fine deformation modules having identical MLP architectures to allow direct weight transfer [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def coarse_to_fine_initialization(scene_data, coarse_iterations=100000):
    """
    Implements coarse-to-fine initialization for dynamic radiance fields.
    """
    # Step 1: Pre-train Coarse Model
    # Initialize coarse deformation network F_theta_d_coarse and color network F_theta_c_coarse
    # Initialize coarse deformation feature grid Grid_coarse
    # Initialize decomposition parameters phi
    
    theta_coarse = initialize_parameters()
    phi = initialize_parameters()
    
    for iteration in range(coarse_iterations):
        # Sample points and render
        rendered_color = render_coarse(scene_data, theta_coarse, phi)
        
        # Calculate losses
        loss_radiance = calculate_reconstruction_loss(rendered_color, ground_truth)
        loss_uniform = calculate_uniformity_loss(phi)
        
        # Update parameters separately to avoid instability
        # Update theta_coarse using only reconstruction loss
        theta_coarse = update_parameters(theta_coarse, loss_radiance)
        
        # Update phi using only uniformity loss
        phi = update_parameters(phi, loss_uniform)
        
    # Step 2: Initialize Fine Model
    # Create fine deformation network F_theta_d_fine with same architecture as F_theta_d_coarse
    # Copy weights from coarse to fine
    F_theta_d_fine.weights = F_theta_d_coarse.weights
    
    # Interpolate coarse feature grid to fine resolution
    Grid_fine = interpolate_grid(Grid_coarse, target_resolution=2 * Grid_coarse.resolution)
    
    # Step 3: Progressive Scale Training
    current_resolution = Grid_fine.resolution
    while training_not_converged():
        # Render using fine model
        rendered_color_fine = render_fine(scene_data, F_theta_d_fine, Grid_fine, phi)
        
        # Calculate reconstruction loss
        loss_radiance_fine = calculate_reconstruction_loss(rendered_color_fine, ground_truth)
        
        # Update fine model parameters
        F_theta_d_fine = update_parameters(F_theta_d_fine, loss_radiance_fine)
        Grid_fine = update_grid(Grid_fine, loss_radiance_fine)
        
        # Progressive scale: double resolution after some iterations
        if iteration % doubling_interval == 0:
            Grid_fine = interpolate_grid(Grid_fine, target_resolution=2 * Grid_fine.resolution)
            current_resolution = Grid_fine.resolution
            
    return F_theta_d_fine, Grid_fine, phi
```

## Operating characteristics
- **Convergence Speed:** Significantly shortens training time compared to training the fine model from scratch [S1].
- **Performance:** Results in higher final performance (PSNR/SSIM) due to better initialization [S1].
- **Memory:** Requires storing both coarse and fine models during the transition, but the coarse model can be discarded after initialization.
- **Scalability:** The progressive doubling of voxel resolution allows handling of high-resolution scenes incrementally [S1].
- **Limitations:** Requires identical MLP architectures for coarse and fine deformation modules to enable weight transfer [S1].

## Background
This method is used in dynamic view synthesis, specifically in frameworks like Neural Deformable Voxel Grid [S1] and DeRF (Decomposed Radiance Fields) [S2]. It addresses the challenges of optimizing high-dimensional radiance fields by breaking the problem into manageable stages [S1, S2].

## Sources
[S1] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
[S2] DeRF: Decomposed Radiance Fields, https://doi.org/10.1109/cvpr46437.2021.01393, fetched 2026-10-26
