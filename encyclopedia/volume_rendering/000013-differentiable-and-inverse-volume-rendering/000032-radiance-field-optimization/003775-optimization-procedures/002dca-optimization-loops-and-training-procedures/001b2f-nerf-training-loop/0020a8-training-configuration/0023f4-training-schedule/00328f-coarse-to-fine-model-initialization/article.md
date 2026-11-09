# Coarse-to-Fine Model Initialization

Also called: Coarse-to-Fine Initialization, Progressive Scale Training

## Overview
Coarse-to-Fine Model Initialization is a training strategy for dynamic radiance fields that accelerates convergence and improves final reconstruction quality by progressively increasing model complexity. It solves the problem of optimizing high-dimensional, view-dependent features in dynamic scenes, which is prone to local minima and slow convergence when trained from scratch. It sets itself apart from standard NeRF training by explicitly decoupling the learning of geometry/deformation (coarse stage) from view-dependent appearance (fine stage), and from [Hierarchical Spherical Harmonics Optimization](5444-hierarchical-spherical-harmonics-optimization.md) by using voxel grids and MLPs rather than spherical harmonics bases.

## How it works
The method consists of two distinct stages: a Coarse Stage to learn geometry and deformation, and a Fine Stage to learn view-dependent color, utilizing weight transfer and resolution scaling.

### Inputs and Outputs
- **Inputs**: A set of dynamic scene images with known camera poses and time steps.
- **Outputs**: A trained dynamic radiance field model (deformation grid, color MLPs) ready for novel view synthesis.

### Data Structures
1.  **Deformation Feature Grid**: A 3D voxel grid storing deformation features. In the coarse stage, this is initialized to zero or random. In the fine stage, it is initialized via interpolation from the coarse stage's optimized grid [S1].
2.  **Coarse Deformation MLP ($F_{\theta_{d1}}$)**: A fully connected network with 4 layers, width 64. Inputs: position (3), time (1), feature (4). Outputs: deformation vector [S1].
3.  **Fine Color MLP ($F_{\theta_{c2}}$)**: A fully connected network with 3 layers, width 128. Inputs: position (3), view direction (27), feature (12). Outputs: color [S1].
4.  **Proxy Geometry Bounding Box**: A smaller bounding box computed during the coarse stage to filter empty space in the fine stage [S2].

### Steps

#### Stage 1: Coarse Training
1.  **Initialize**: Set color features to be the color itself, ignoring ray direction dependency [S2].
2.  **Train Deformation**: Optimize the coarse deformation module ($F_{\theta_{d1}}$) and the deformation feature grid using images close to the canonical time first, progressively adding farther time steps [S2].
3.  **Filter Empty Space**: Compute a proxy geometry from the coarse stage and define a smaller bounding box. Filter out sampled points outside this box or with alpha below a threshold during all training times (only filtering empty points at canonical time for the canonical module) [S1, S2].

#### Stage 2: Fine Training (Initialization and Optimization)
1.  **Initialize Deformation Grid**: Interpolate the optimized feature grid from the coarse model into the fine model's deformation feature grid [S1].
2.  **Initialize MLP Weights**: Copy the trained network weights from the coarse deformation module to the fine model's deformation module, as the architecture is identical [S1].
3.  **Progressive Scale Training**: Begin with a base voxel resolution. After a fixed number of training iterations, double the voxel resolution [S1]. Repeat until the target resolution is reached.
4.  **Train View-Dependent Color**: Use a high-dimensional feature vector and a lightweight MLP ($F_{\theta_{c2}}$) to decode color, explicitly modeling ray direction dependency [S1, S2].
5.  **Finalize**: The model trained in this stage is the final model [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def coarse_to_fine_training(scene_data, canonical_time_idx):
    # Stage 1: Coarse Training
    coarse_deform_grid = initialize_voxel_grid(resolution=coarse_res)
    coarse_deform_mlp = initialize_mlp(layers=4, width=64, 
                                       input_dim=8, output_dim=3) # pos(3)+time(1)+feat(4) -> def(3)
    
    # Progressive time training
    time_steps = get_closest_times(scene_data, canonical_time_idx)
    for t in time_steps:
        optimize(coarse_deform_grid, coarse_deform_mlp, scene_data[t])
        
    # Filter empty space using proxy geometry
    proxy_bbox = compute_proxy_bbox(coarse_deform_grid)
    
    # Stage 2: Fine Training
    fine_deform_grid = interpolate_grid(coarse_deform_grid, fine_res)
    fine_deform_mlp = copy_weights(coarse_deform_mlp) # Same architecture
    
    # Progressive scale training
    current_res = base_res
    while current_res <= target_res:
        optimize_fine_stage(fine_deform_grid, fine_deform_mlp, scene_data, 
                            bbox=proxy_bbox, resolution=current_res)
        current_res *= 2 # Double resolution
        
    return fine_deform_grid, fine_deform_mlp
```

### Parameters
- **Coarse MLP Width**: 64 [S1].
- **Fine MLP Width**: 128 [S1].
- **Feature Dimension (Coarse)**: 4 [S1].
- **Feature Dimension (Fine)**: 12 [S1].
- **View Direction Dimension**: 27 (likely SH basis or similar encoding) [S1].

### Edge Cases and Pitfalls
- **Precision**: Interpolation of the feature grid must preserve the learned deformation patterns; naive zero-filling of new voxels may degrade performance [S1].
- **Boundaries**: The proxy bounding box must be tight enough to save computation but loose enough to avoid clipping valid geometry [S2].

## Subtopics

### Progressive Coarse-to-Fine Training Strategy
This variant emphasizes the temporal progression in the coarse stage. It starts training with images close to the canonical time and progressively adds images with farther time steps until all images are added [S2]. This helps stabilize the learning of deformation before introducing complex view-dependent effects in the fine stage [S2].

## Operating characteristics
- **Time Complexity**: Reduced compared to training from scratch due to warm-starting the deformation module [S1].
- **Memory Complexity**: Higher than single-stage training due to maintaining both coarse and fine grids/MLPs during transition, though the coarse stage uses lower resolution [S1].
- **Performance**: Results in higher final performance and shorter training time [S1].
- **Strengths**: Avoids local minima in view-dependent optimization; efficiently filters empty space [S2].
- **Limitations**: Requires careful tuning of the resolution doubling schedule [S1].

## Background
Originating from works on dynamic view synthesis such as Neural Deformable Voxel Grid [S1] and Forward Flow [S2], this method addresses the computational cost and optimization difficulty of learning dynamic radiance fields. It builds on the observation that geometry and deformation can be learned independently of view-dependent appearance [S1, S2].

## Sources
[S1] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
[S2] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
