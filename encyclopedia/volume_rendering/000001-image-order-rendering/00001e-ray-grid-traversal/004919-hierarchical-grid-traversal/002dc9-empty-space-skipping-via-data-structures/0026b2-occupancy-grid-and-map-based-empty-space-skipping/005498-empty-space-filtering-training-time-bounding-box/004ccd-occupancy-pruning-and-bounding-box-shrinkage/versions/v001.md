# Occupancy Pruning and Bounding Box Shrinkage

Also called: Occupancy Grid Pre-calculation and Bounding Box Shrinkage

## Overview
Occupancy Pruning and Bounding Box Shrinkage is a training and inference acceleration technique that pre-calculates an occupancy grid to prune non-occupied points along rays and shrinks the scene bounding box for more accurate modeling [S1]. It differs from its parent, [Empty Space Filtering (Training-Time Bounding Box)](6752-empty-space-filtering-training-time-bounding-box.md), by dynamically updating the occupancy grid during training and using it for active ray pruning, rather than just static preprocessing [S1]. It differs from sibling [Depth-guided Voxel Pre-pruning](6897-depth-guided-voxel-pre-pruning.md) by relying on learned occupancy grids rather than depth-guided geometric masks [S1].

## How it works
**Inputs:**
- Scene representation (e.g., factorized matrices for density in tensor rank decomposition) [S1].
- Camera rays for training or inference [S1].
- Current training step count [S1].

**Outputs:**
- Pruned set of sample points along rays [S1].
- Shrunk bounding box parameters [S1].
- Occupancy grid data structure [S1].

**Data Structures:**
- **Occupancy Grid:** A 3D grid storing binary or probabilistic occupancy values for voxels. The grid is recorded per single object to support scene composition [S1].
- **Bounding Box:** Defined by minimum and maximum coordinates, shrunk based on the occupancy grid at step 2,000 [S1].

**Steps:**
1. **Initialize Training:** Begin training the model (e.g., for 30,000 iterations) [S1].
2. **Pre-calculate Occupancy Grid at Step 2,000:**
   - Query the current scene representation to determine voxel occupancy [S1].
   - Generate the occupancy grid [S1].
   - **Shrink Bounding Box:** Use the occupancy grid from step 2,000 to compute a tighter bounding box for the scene [S1]. This ensures more accurate modeling by excluding empty space [S1].
3. **Pre-calculate Occupancy Grid at Step 4,000:**
   - Update the occupancy grid based on the refined scene representation at step 4,000 [S1].
4. **Ray Pruning During Training/Inference:**
   - For each ray, query the occupancy grid to identify non-occupied voxels [S1].
   - Prune sample points that fall within non-occupied voxels [S1].
   - Use warped rays to query occupancy if coordinate transformations are applied [S1].
5. **Scene Composition:**
   - Keep the occupancy grid for scene composition tasks [S1].
   - Record the occupancy grid for each single object to enable modular scene assembly [S1].
6. **Regularization (Training Specific):**
   - Apply L1 regularization to the factored matrices for density [S1].
   - Formula: $\mathcal{L}_{total} = \mathcal{L}_{rendering} + \lambda \sum |W_i|$ where $W_i$ are the parameters of the factored matrices [S1].
   - This encourages sparsity in the tensor rank decomposition parameters, which aids in novel view extrapolation and removing floaters [S1].

**Parameters:**
- **Pre-calculation Steps:** 2,000 and 4,000 [S1].
- **L1 Regularization:** Applied to factored matrices for density [S1]. Typical value not given in sources.
- **Learning Rate:** 0.02 for factorized matrices, 0.001 for singular values (Adam optimizer) [S1].

**Edge Cases and Pitfalls:**
- **Precision:** Not given in the sources.
- **Boundaries:** The bounding box shrinkage is based on the occupancy grid at step 2,000, which may be coarse if the model has not converged sufficiently [S1].
- **Degenerate Input:** Not given in the sources.

## Operating characteristics
**Time and Memory Complexity:**
- Pre-calculation of the occupancy grid adds overhead at steps 2,000 and 4,000 [S1].
- Ray pruning reduces the number of sample points evaluated per ray, accelerating both training and inference [S1].
- Memory usage increases due to the storage of the occupancy grid, particularly when recorded per object for scene composition [S1].

**Measured Performance:**
- Specific performance metrics are not given in the sources.

**Scaling:**
- The technique scales with the resolution of the occupancy grid and the complexity of the scene [S1].

**Strengths and Limitations:**
- **Strengths:** Accelerates training and inference by avoiding computation in empty space; improves modeling accuracy via bounding box shrinkage; L1 regularization aids in removing floaters and improving novel view extrapolation [S1].
- **Limitations:** Requires pre-calculation steps; memory overhead for storing per-object occupancy grids [S1].

**When to Use:**
- Use when training or rendering large scenes with significant empty space to accelerate computation [S1].
- Use when scene composition is required, as the grid is kept for this purpose [S1].

**When Not to Use:**
- Not given in the sources.

## Background
**Origin:**
- The technique is adopted from prior work [8, 27] [S1].
- [8] refers to TensoRF, which introduced tensor rank decomposition for neural radiance fields [S1].

**History and Development:**
- The method was used in the context of Compressible-Composable NeRF Via Rank-Residual Decomposition [S1].
- It builds on the idea of using occupancy grids for empty space skipping, as seen in TensoRF [S1].

**Where it is Used:**
- Used in training and inference of compressible and composable NeRF models [S1].

## Sources
[S1] Compressible-Composable NeRF Via Rank-Residual Decomposition, https://doi.org/10.52202/068431-1076, fetched 2026-10-26
