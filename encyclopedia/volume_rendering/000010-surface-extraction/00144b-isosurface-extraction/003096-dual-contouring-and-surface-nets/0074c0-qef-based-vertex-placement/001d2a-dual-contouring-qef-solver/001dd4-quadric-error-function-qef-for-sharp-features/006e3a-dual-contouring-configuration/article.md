# Dual Contouring Configuration

Also called: DualContouringConfig

## Overview
Dual Contouring Configuration is a setup method that initializes the parameters for a GPU-based dual contouring surface extraction pipeline. It bridges the gap between high-level volume rendering settings and the low-level compute shader execution by mapping `VoxelVolumeConfig` and `DualContouringConfig` values to shader uniforms. It ensures that the voxel grid dimensions, spacing, and sharp feature detection thresholds are correctly propagated to the vertex generation kernels.

## How it works
The method operates on a configuration change event (e.g., `OnDirtied`). It reads settings from two configuration objects and writes them to the compute shader's global state.

**Inputs:**
- `VoxelVolumeConfig`: Contains voxel grid dimensions and physical spacing.
- `DualContouringConfig`: Contains algorithmic parameters for sharp feature handling and optimization.

**Outputs:**
- Compute shader global variables:
  - `NumberOfVoxels` (`uint3`): The dimensions of the voxel grid.
  - `VoxelSpacing` (`float`): The physical size of a single voxel.
  - `CosOfHalfSharpFeatureAngle` (`float`): Precomputed cosine for sharp feature angle detection.
  - `SchmitzParticleIterations` (`int`): Iteration count for Schmitz particle optimization.
  - `SchmitzParticleStepSize` (`float`): Step size for Schmitz particle optimization.

**Data Structures:**
The configuration objects are assumed to have the following fields (based on usage in [S1, unofficial]):
- `VoxelVolumeConfig`:
  - `NumberOfVoxels`: `uint3` (x, y, z dimensions).
  - `VoxelSpacing`: `float`.
- `DualContouringConfig`:
  - `SharpFeatureAngle`: `float` (in degrees).
  - `SchmitzParticleIterations`: `int`.
  - `SchmitzParticleStepSize`: `float`.

**Steps:**
1. **Trigger:** The configuration update is triggered when the volume or dual contouring settings are marked as dirty.
2. **Voxel Configuration:** Read `NumberOfVoxels` and `VoxelSpacing` from `VoxelVolumeConfig` and assign them to the corresponding compute shader globals.
3. **Sharp Feature Angle:** Read `SharpFeatureAngle` from `DualContouringConfig`.
4. **Cosine Precomputation:** Calculate `CosOfHalfSharpFeatureAngle` using the formula:
   $$ \text{CosOfHalfSharpFeatureAngle} = \cos\left(\text{radians}\left(0.5 \times \text{SharpFeatureAngle}\right)\right) $$
   Assign the result to the compute shader global `CosOfHalfSharpFeatureAngle`.
5. **Schmitz Parameters:** Read `SchmitzParticleIterations` and `SchmitzParticleStepSize` from `DualContouringConfig` and assign them to the corresponding compute shader globals.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_dual_contouring_config(voxel_config, dual_contouring_config, shader_globals):
    """
    Applies configuration settings to the compute shader globals.
    
    Args:
        voxel_config: Object with NumberOfVoxels (uint3) and VoxelSpacing (float).
        dual_contouring_config: Object with SharpFeatureAngle (float), 
                                SchmitzParticleIterations (int), 
                                and SchmitzParticleStepSize (float).
        shader_globals: Object representing the compute shader's global state.
    """
    # Step 2: Apply Voxel Volume Config
    shader_globals.NumberOfVoxels = voxel_config.NumberOfVoxels
    shader_globals.VoxelSpacing = voxel_config.VoxelSpacing
    
    # Step 3 & 4: Apply Sharp Feature Angle
    sharp_feature_angle = dual_contouring_config.SharpFeatureAngle
    # Convert degrees to radians and take half, then cosine
    # radians(x) = x * pi / 180
    cos_of_half_angle = math.cos(math.radians(0.5 * sharp_feature_angle))
    shader_globals.CosOfHalfSharpFeatureAngle = cos_of_half_angle
    
    # Step 5: Apply Schmitz Particle Parameters
    shader_globals.SchmitzParticleIterations = dual_contouring_config.SchmitzParticleIterations
    shader_globals.SchmitzParticleStepSize = dual_contouring_config.SchmitzParticleStepSize
```

**Parameters:**
- `SharpFeatureAngle`: Float defining the angle threshold for sharp features. Typical values are not specified in the sources. [S1, unofficial]
- `SchmitzParticleIterations`: Integer controlling the number of iterations for Schmitz particle optimization. Typical values are not specified in the sources. [S1, unofficial]
- `SchmitzParticleStepSize`: Float controlling the step size for Schmitz particle optimization. Typical values are not specified in the sources. [S1, unofficial]

**Edge Cases and Pitfalls:**
- **Precision:** The cosine precomputation should be done in floating-point precision consistent with the shader's expectations. [S1, unofficial]
- **Boundary Conditions:** The configuration itself does not handle boundary conditions; these are handled in the vertex generation kernels (e.g., `GenerateOuterCellVertices` checks `IsOuterVoxel`). [S1, unofficial]

## Subtopics

### [Dynamic CSG Data Structure Management](23573-dynamic-csg-data-structure-management.md)
A data structure management scheme for maintaining a mesh generated from a multi-resolution octree during Constructive Solid Geometry (CSG) operations, involving dynamic addition and removal of vertices and polygons.

## Operating characteristics
- **Time Complexity:** O(1) for the configuration update itself, as it involves a fixed number of variable assignments and one trigonometric calculation.
- **Memory Complexity:** O(1) additional memory, as it only updates shader globals.
- **Performance:** The configuration update is negligible compared to the cost of the dual contouring mesh generation. [S1, unofficial]
- **Strengths:** Centralizes parameter management, ensuring consistency between volume settings and surface extraction algorithms. [S1, unofficial]
- **Limitations:** Requires a configuration change event to trigger updates; does not validate parameter ranges. [S1, unofficial]

## Background
This method is part of the dual contouring pipeline implemented in the `Voxels` project by Tantenfisch. It is designed to work with GPU compute shaders for efficient surface extraction from voxel data. The use of Schmitz particle optimization suggests an approach to handle sharp features and complex topology. [S1, unofficial]

## Sources
[S1] Tantenfisch/Voxels, https://github.com/Tantenfisch/Voxels, fetched 2026-10-26 (unofficial)
