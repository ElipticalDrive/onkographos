# Coarse-to-fine feature grid
Also called: Progressive Upscaling of Feature Grids, Coarse-to-fine grid optimization

## Overview
A training optimization scheme that reduces initial training cost and regularizes feature matrices and vectors by initializing a 3D feature grid at a coarse resolution and progressively upscaling it to higher resolutions [S1]. It improves model convergence and makes camera refinement more effective compared to training at full resolution from the start [S2, unofficial]. Unlike static grid methods, this approach dynamically adapts the spatial detail of the feature representation during the optimization process [S1][S2, unofficial].

## How it works
**Inputs and Outputs**
- **Input:** Initial coarse 3D feature grid (matrices and vectors), current iteration count [S1].
- **Output:** Updated feature grid parameters at the current resolution stage.

**Data Structures**
The method operates on the 3D feature grid components defined in the parent [TensoRF Decomposition Models](13296-tensorf-decomposition-models.md), which consist of feature matrices and feature vectors [S1].
- **Feature Matrices:** 2D tensors representing features along specific axes [S1].
- **Feature Vectors:** 1D tensors representing features at grid nodes [S1].

**Numbered Steps**
1. **Initialization:** Initialize the 3D feature grid with a coarse resolution [S1].
2. **Start Optimization:** Begin model optimization with an initial grid resolution of $d = 32$ [S2, unofficial].
3. **Iterative Upscaling:**
   - Check if the current iteration count is a multiple of 5,000 [S2, unofficial].
   - If yes, upsample the grid to a resolution that is two times larger in each dimension [S2, unofficial].
   - Apply bi-linear upscaling to the feature matrices [S1].
   - Apply linear upscaling to the feature vectors [S1].
   - [G] Initialize the new grid entries with interpolated values from the coarse grid.
4. **Repeat:** Continue the optimization and upscaling process every 5,000 iterations [S2, unofficial].
5. **Termination:** Stop upscaling when the final dimension reaches $d = 256$ or the desired fine resolution determined by scene complexity [S1][S2, unofficial].

**Formulas**
- **Upsampling Interval:** Upsampling occurs at iterations $k$ where $k \mod 5000 = 0$ [S2, unofficial].
- **Resolution Progression:** $d_{new} = 2 \times d_{old}$ [S2, unofficial].
- **Matrix Upscaling:** Bi-linear interpolation is used for 2D feature matrices [S1].
- **Vector Upscaling:** Linear interpolation is used for 1D feature vectors [S1].

**Parameters**
- **Initial Resolution ($d_{init}$):** 32 [S2, unofficial].
- **Final Resolution ($d_{final}$):** 256 [S2, unofficial] or determined by scene complexity (input DVR or IR images) [S1].
- **Upsampling Frequency:** Every 5,000 iterations [S2, unofficial].
- **Upsampling Factor:** 2x per step [S2, unofficial].

**Edge Cases and Pitfalls**
- **Scene Complexity:** If the scene is simple, a lower final resolution may suffice, reducing memory usage [S1].
- **Camera Refinement:** Coarse-to-fine optimization is critical for effective camera refinement; skipping it may lead to poor pose estimation [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def coarse_to_fine_feature_grid(initial_resolution=32, final_resolution=256, upsample_interval=5000):
    resolution = initial_resolution
    iteration = 0
    # [G] Initialize coarse feature matrices and vectors
    feature_matrices = create_tensor(resolution, resolution, resolution)
    feature_vectors = create_tensor(resolution, resolution, resolution)
    
    while resolution < final_resolution:
        # [Comment] Perform standard gradient descent optimization for the current resolution
        # optimize_step(feature_matrices, feature_vectors, loss_fn, optimizer)
        iteration += 1
        
        if iteration % upsample_interval == 0:
            new_resolution = resolution * 2
            # Bi-linear upscaling for feature matrices
            feature_matrices = interpolate_2d(feature_matrices, new_resolution, mode='bilinear')
            # Linear upscaling for feature vectors
            feature_vectors = interpolate_1d(feature_vectors, new_resolution, mode='linear')
            resolution = new_resolution
            
    return feature_matrices, feature_vectors
```

## Subtopics

### Coarse-to-Fine Optimization Transition
A transition strategy from a coarse feature grid to a fine grid to improve shape refinement, triggered at a specific iteration count [S3, unofficial]. At iteration 5000, it samples the coarse color feature at each grid coordinate and copies it to the new $R_{min}$ level (fine grid) [S3, unofficial]. It reinitializes the alpha feature during this transition to avoid oversaturated alpha values from coarse-stage training hindering shape refinement [S3, unofficial]. The transition iteration is fixed at 5000 [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Reduces initial training cost by starting with a smaller grid [S1]. Total training time may increase due to upscaling steps, but convergence is faster [S2, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Regularizes the optimization process, prevents overfitting in early stages, and improves camera refinement [S1][S2, unofficial].
- **Limitations:** Requires careful tuning of upscaling frequency and final resolution [S1][S2, unofficial].
- **When to Use:** When training cost is a concern, or when camera refinement is critical [S1][S2, unofficial].

## Background
Originated in the context of TensoRF and HexPlane representations to address the high computational cost of training large 3D feature grids [S1]. It was adopted in ViSNeRF for efficient multidimensional neural radiance field representation [S1] and in Neural Assets for volumetric object capture [S2, unofficial]. The method reduces initial training cost and regularizes matrices and vectors [S1], while improving model convergence and making camera refinement more effective [S2, unofficial].

## Sources
[S1] ViSNeRF: Efficient Multidimensional Neural Radiance Field Representation for Visualization Synthesis of Dynamic Volumetric Scenes, https://doi.org/10.1109/pacificvis64226.2025.00029, fetched 2026-10-26
[S2] Neural Assets: Volumetric Object Capture and Rendering for Interactive Environments, https://core.ac.uk/download/552466290.pdf, fetched 2026-10-30 (unofficial)
[S3] DiffSoup: Direct Differentiable Rasterization of Triangle Soup for Extreme Radiance Field Simplification, https://doi.org/10.48550/arXiv.2603.27151, fetched 2026-10-30 (unofficial)
