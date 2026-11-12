# Coarse-to-Fine Model Initialization

Also called: Coarse-to-Fine Initialization, Progressive Scale Training

## Overview
Coarse-to-Fine Model Initialization is a training strategy for radiance field optimization that initializes a high-resolution "fine" model using the learned parameters from a lower-resolution "coarse" model. It solves the problem of optimization instability and local minima in complex scenes by providing a geometrically coherent starting point for high-frequency detail recovery. It sets itself apart from single-resolution training by preventing the model from committing to high-frequency noise before establishing global structure [S2, unofficial], and differs from the NeRF Restarting Approach by using a continuous schedule of complexity rather than discrete restarts [12980-nerf-restarting-approach.md].

## How it works
The method initializes the fine model's parameters using the optimized weights of the coarse model, typically involving interpolation for grid-based features and direct weight copying for MLPs.

**Inputs and Outputs:**
*   **Input:** Trained coarse model parameters (deformation MLP weights, feature grid), target fine model architecture (same MLP structure, higher resolution grid).
*   **Output:** Initialized fine model ready for high-resolution training.

**Data Structures:**
*   **Deformation Feature Grid:** A 3D voxel grid storing deformation features. In the coarse model, it has resolution $R_c$. In the fine model, it has resolution $R_f = 2 R_c$ (or similar doubling).
*   **Deformation MLP:** A Multi-Layer Perceptron mapping position/time to deformation features. The architecture (layer widths, activation functions) is identical between coarse and fine models.

**Steps:**
1.  **MLP Weight Transfer:** Copy the trained weights of the coarse deformation MLP directly to the fine deformation MLP. Since the architecture is identical, this provides a good initialization for the deformation field [S1].
2.  **Feature Grid Interpolation:** Initialize the fine model's deformation feature grid by interpolating the optimized feature grid from the coarse model. This ensures the fine model starts with a smooth, low-frequency approximation of the deformation field [S1].
3.  **Progressive Scale Training:** During the fine model training, progressively increase the voxel resolution (e.g., doubling it) after a certain number of iterations. This allows the model to refine details gradually [S1].

**Formulas:**
*   Not given in the sources.

**Parameters:**
*   **Voxel Resolution Doubling Interval:** Not given in the sources.
*   **Interpolation Method:** Not given in the sources (typically trilinear or nearest-neighbor).

**Edge Cases and Pitfalls:**
*   **Architecture Mismatch:** The method relies on the coarse and fine deformation MLPs having the exact same architecture. If they differ, direct weight transfer is not possible [S1].
*   **Grid Resolution:** The fine grid resolution must be compatible with the interpolation from the coarse grid (e.g., powers of two) to ensure efficient interpolation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_fine_model(coarse_model, fine_model):
    """
    Initializes the fine model using the trained coarse model.
    
    Args:
        coarse_model: The trained coarse model containing deformation_mlp and deformation_grid.
        fine_model: The fine model with identical deformation_mlp architecture but higher resolution deformation_grid.
        
    Returns:
        fine_model: The initialized fine model.
    """
    # Step 1: Transfer MLP weights
    # Since the architecture is identical, we can directly copy the weights
    fine_model.deformation_mlp.load_state_dict(coarse_model.deformation_mlp.state_dict())
    
    # Step 2: Interpolate feature grid
    # Interpolate the coarse grid to the fine grid resolution
    # Assuming trilinear interpolation for simplicity
    coarse_grid = coarse_model.deformation_grid
    fine_grid = interpolate_grid(coarse_grid, fine_model.deformation_grid.shape)
    fine_model.deformation_grid.data = fine_grid
    
    return fine_model

def interpolate_grid(coarse_grid, target_shape):
    """
    Interpolates a 3D grid to a target shape.
    
    Args:
        coarse_grid: 3D numpy array of shape (H, W, D, C)
        target_shape: Tuple (H_new, W_new, D_new, C)
        
    Returns:
        3D numpy array of shape target_shape
    """
    # Placeholder for actual interpolation logic (e.g., using scipy.ndimage.zoom or torch.nn.functional.interpolate)
    # This is a generic implementation detail not specified in the sources [G]
    import numpy as np
    from scipy.ndimage import zoom
    
    # Calculate zoom factors for each dimension
    zoom_factors = [target_shape[i] / coarse_grid.shape[i] for i in range(3)]
    
    # Perform interpolation
    # The last dimension (channels) is not interpolated
    interpolated_grid = zoom(coarse_grid, zoom_factors + [1.0], order=1) # order=1 for linear interpolation
    
    return interpolated_grid
```

## Subtopics

### [Progressive Coarse-to-Fine Training Strategy](8900-progressive-coarse-to-fine-training-strategy.md)
A training methodology that decouples the recovery of global scene structure from the refinement of high-frequency details by progressively increasing input complexity or model capacity. It solves the problem of optimization instability and local minima in radiance field reconstruction, particularly in dynamic scenes or when camera poses are imperfect. It sets itself apart from single-resolution training by preventing the model from committing to high-frequency noise before establishing geometric coherence, and differs from the NeRF Restarting Approach by using a continuous schedule of complexity rather than discrete restarts [12980-nerf-restarting-approach.md].

### [NeRF Restarting Approach](12980-nerf-restarting-approach.md)
The NeRF Restarting Approach is a training schedule for joint pose and radiance field optimization that eliminates artifacts learned during the pose refinement phase by discarding the neural network weights and retraining from scratch. Unlike staged training, which freezes poses and continues training the existing networks, this method re-initializes both coarse and fine Multi-Layer Perceptrons (MLPs) after pose convergence. It is chosen when the initial joint optimization introduces persistent artifacts into the scene representation that persist even after pose stabilization.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Shortens training time and results in higher performance by providing a good initialization [S1].
*   **Limitations:** Requires the coarse and fine models to have compatible architectures (specifically the MLP structure) [S1].
*   **When to Use:** When training dynamic scenes or scenes with imperfect camera poses where optimization instability is a concern [S1][S3, S4, unofficial].
*   **When Not to Use:** When the coarse and fine models have different architectures, or when the scene is simple enough that single-resolution training converges reliably.

## Background
The method is used in the context of Neural Deformable Voxel Grids for fast optimization of dynamic view synthesis [S1] and UP-NeRF for unconstrained pose-prior-free neural radiance fields [S2, unofficial]. It builds on the idea of hierarchical optimization to improve convergence in complex radiance field reconstruction tasks.

## Sources
[S1] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
[S2] UP-NeRF: Unconstrained Pose-Prior-Free Neural Radiance Fields, https://arxiv.org/abs/2311.03784v2, fetched 2026-10-26 (unofficial)
