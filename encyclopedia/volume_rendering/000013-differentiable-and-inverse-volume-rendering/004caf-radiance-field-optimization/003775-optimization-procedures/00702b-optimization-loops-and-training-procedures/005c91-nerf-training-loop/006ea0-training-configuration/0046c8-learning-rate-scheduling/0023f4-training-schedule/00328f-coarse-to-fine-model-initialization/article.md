# Coarse-to-Fine Model Initialization

Also called: Coarse-to-Fine Initialization, Progressive Scale Training

## Overview
Coarse-to-Fine Model Initialization is a training protocol that initializes a high-resolution neural radiance field (or deformable voxel grid) using the optimized parameters from a lower-resolution counterpart. It solves the problem of unstable convergence and local minima in complex, high-dimensional loss landscapes by establishing a coherent global structure before fitting high-frequency details [S1, S2]. It differs from standard full-resolution training by deferring the fitting of fine features until the global geometry is established , and differs from simple two-stage pipelines by potentially using continuous or multi-stage resolution scaling rather than a single abrupt switch [S1].

## How it works
**Inputs:**
1.  A trained coarse model consisting of:
    *   A deformation feature grid (voxel grid) optimized at resolution $R_{coarse}$.
    *   A deformation MLP with weights $W_{deform}^{coarse}$.
2.  Target fine model architecture:
    *   A deformation feature grid at resolution $R_{fine}$ (where $R_{fine} > R_{coarse}$).
    *   A deformation MLP with identical architecture to the coarse model.

**Outputs:**
1.  Initialized fine model parameters ready for high-resolution optimization.

**Data Structures:**
*   **Deformation Feature Grid:** A 3D tensor of shape $(D, H, W, C)$, where $D, H, W$ are spatial dimensions and $C$ is the feature dimension. Values are typically stored as 32-bit floats.
*   **Deformation MLP:** A Multi-Layer Perceptron. For example, 4 fully connected layers with width 64, ReLU activations, input dimension 33 (3 pos + 1 time + 44 feat), output dimension 44 [S1].

**Steps:**
1.  **Grid Interpolation:** Initialize the fine model's deformation feature grid by interpolating the optimized coarse model's feature grid.
    *   If $R_{fine} = 2 \times R_{coarse}$, use trilinear interpolation or nearest-neighbor upscaling to map coarse grid values to the finer grid.
    *   This preserves the learned global structure at higher spatial fidelity [S1].
2.  **Weight Transfer:** Copy the trained weights $W_{deform}^{coarse}$ from the coarse deformation MLP to the fine deformation MLP.
    *   This is valid because the MLP architecture (layer count, width, activation functions) remains identical between coarse and fine models [S1].
3.  **Progressive Scale Training (Optional/Variant):** Instead of a single jump, apply progressive scale training.
    *   Start optimization at $R_{coarse}$.
    *   After a fixed number of iterations, double the voxel resolution ($R_{next} = 2 \times R_{current}$).
    *   Re-initialize the new grid levels using interpolation from the previous resolution.
    *   Continue optimization at the new resolution [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_fine_model(coarse_model, target_resolution):
    """
    Initializes a fine-resolution model using a trained coarse model.
    
    Args:
        coarse_model: Object containing:
            - coarse_grid: 3D tensor (D, H, W, C)
            - coarse_mlp_weights: List of numpy arrays for MLP layers
        target_resolution: Tuple (D_fine, H_fine, W_fine)
        
    Returns:
        fine_model: Initialized model structure
    """
    # Step 1: Interpolate Feature Grid
    # Assuming trilinear interpolation for upscaling
    fine_grid = interpolate_3d(coarse_model.coarse_grid, target_resolution)
    
    # Step 2: Transfer MLP Weights
    # Architecture is identical, so direct copy is valid
    fine_mlp_weights = [w.copy() for w in coarse_model.coarse_mlp_weights]
    
    # Step 3: Construct Fine Model
    fine_model = {
        'grid': fine_grid,
        'mlp_weights': fine_mlp_weights,
        'resolution': target_resolution
    }
    
    return fine_model

def progressive_scale_training(model, max_resolution, step_iterations=10000):
    """
    Doubles voxel resolution every step_iterations.
    """
    current_res = model.resolution
    while current_res < max_resolution:
        # Double resolution
        new_res = tuple(2 * d for d in current_res)
        
        # Re-initialize grid via interpolation
        model.grid = interpolate_3d(model.grid, new_res)
        model.resolution = new_res
        
        # Continue optimization for step_iterations
        optimize(model, iterations=step_iterations)
        
        current_res = new_res
```

**Parameters:**
*   **Interpolation Method:** Trilinear or nearest-neighbor. Not given in the sources.
*   **Step Iterations:** Number of iterations before doubling resolution. Not given in the sources.
*   **MLP Width:** 64 for deformation module, 128 for color module [S1].

**Edge Cases and Pitfalls:**
*   **Resolution Mismatch:** If the fine resolution is not a power-of-two multiple of the coarse resolution, interpolation artifacts may occur. Not given in the sources.
*   **Feature Dimension Mismatch:** The feature dimension $C$ must remain constant between coarse and fine models for direct weight transfer to the MLP. If $C$ changes, the MLP weights cannot be directly transferred [S1].

## Subtopics

### [Progressive Coarse-to-Fine Training Strategy](8900-progressive-coarse-to-fine-training-strategy.md)
Progressive Coarse-to-Fine Training is a strategy that gradually introduces finer levels of detail during optimization to ensure stable convergence and prevent overfitting. It solves the problem of complex loss landscapes and sparse initialization errors that occur when training directly at high resolution by establishing a coherent global structure before fitting high-frequency features. It differs from standard full-resolution training by deferring the fitting of fine features until the global geometry is established, and differs from simple two-stage pipelines by potentially using continuous or multi-stage resolution scaling rather than a single abrupt switch.

### [NeRF Restarting Approach](12980-nerf-restarting-approach.md)
The NeRF Restarting Approach is an optimization protocol for jointly refining camera poses and neural radiance fields (NeRFs) from sparse, noisy initial poses. It solves the problem of artifacts persisting in the scene representation due to the coupling of pose and geometry optimization. Unlike [Staged Training](12984-staged-training.md), which freezes poses after initial convergence but retains the learned network weights, this method re-initializes the network from scratch after pose convergence, thereby removing artifacts learned during the pose optimization phase.

### [Overall Training Pipeline](13114-overall-training-pipeline.md)
This pipeline defines the end-to-end training procedure for unconstrained neural radiance fields, specifically integrating pose estimation, transient artifact handling, and hierarchical sampling. It solves the problem of training NeRFs on unposed, internet-scale image collections where initial camera poses are unknown and images contain transient objects (e.g., people, vehicles). It distinguishes itself from standard NeRF training by jointly optimizing camera poses and scene representation using a coarse-to-fine hierarchical sampling strategy, while explicitly decoupling transient artifacts via a dedicated transient network and depth-supervised scheduling.

### LiDAR Initialization
LiDAR Initialization is an initialization technique using LiDAR point clouds to provide prior information for the scene representation, helping to avoid local minima in optimization [S2]. On Argoverse2, the combination of Hybrid Scene Representation (HSR) and LiDAR Initialization (LI) yields more than 4 times boost in convergence speed compared with NGP [S2]. Without LI, the model cannot converge to NGP's PSNR, and training time is infinite ('Inf.') because it struggles to represent space sparsely [S2].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** The coarse-to-fine approach requires storing both the coarse and fine models during the transition phase if progressive training is used. The fine model's memory usage scales with $R_{fine}^3$.
*   **Time:** Initialization is fast (interpolation). However, the total training time is reduced compared to training from scratch at high resolution because the optimizer starts closer to the global optimum [S1, S2].

**Measured Performance:**
*   In [S2], the combination of HSR and LI achieved a >4x speedup in convergence compared to NGP.
*   Rendering speed was 10x faster due to reduced sample points per ray [S2].

**Strengths and Limitations:**
*   **Strengths:** Faster convergence, better final PSNR, avoids local minima [S1, S2].
*   **Limitations:** Requires a valid coarse model first. If the coarse model is poor, the fine model may inherit bad priors.

**When to Use:**
*   When training high-resolution NeRFs or voxel grids.
*   When initial poses are noisy or unknown (combined with restarting).
*   When LiDAR data is available for initialization.

**When Not to Use:**
*   When computational resources are extremely limited and even the coarse model cannot be trained.
*   When the scene is static and low-resolution is sufficient.

## Background
Coarse-to-Fine Model Initialization originates from the need to stabilize the training of high-dimensional neural radiance fields. It was notably applied in Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis [S1] and Lightning NeRF for Efficient Hybrid Scene Representation for Autonomous Driving [S2]. The method leverages the observation that global geometry converges faster than high-frequency details, allowing a hierarchical optimization strategy.

## Sources
[S1] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
[S2] Lightning NeRF: Efficient Hybrid Scene Representation for Autonomous Driving, https://doi.org/10.1109/icra57147.2024.10611130, fetched 2026-10-27
