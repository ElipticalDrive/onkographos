# SDF Value Computation

Also called: IsoVal.compute

## Overview
SDF Value Computation is a compute shader kernel that calculates the Signed Distance Function (SDF) value for each voxel in a grid [S1, unofficial]. It transforms grid coordinates to world space, evaluates the SDF function, and stores the result [S1, unofficial]. It serves as the atomic evaluation step within batched inference pipelines [G].

## How it works
**Inputs and Outputs:**
*   **Input:** A linearized 3D grid of voxel indices (IDs), a $4 \times 4$ transformation matrix `_ShapeToWorld` mapping local shape coordinates to world space, and the shape's intrinsic size `SHAPE_SIZE` [S1, unofficial].
*   **Output:** A 1D array or texture `_IsoVals` storing the computed SDF value for each voxel [S1, unofficial].

**Data Structures:**
*   `_IsoVals`: A 1D array storing SDF values, indexed by a linearized 3D coordinate ID [S1, unofficial].
*   `_ShapeToWorld`: A $4 \times 4$ matrix [S1, unofficial].
*   `SHAPE_SIZE`: A scalar defining the scale or extent of the primitive shape [S1, unofficial].

**Steps:**
1.  **Grid Coordinate Retrieval:** For each voxel, determine its 3D grid coordinates from its linear ID [S1, unofficial].
2.  **Local Position Calculation:** Convert the grid coordinates into a local surface position using a helper `GetPosSurface(id)` [S1, unofficial].
3.  **World Space Transformation:** Transform the local position $\mathbf{p}_{local}$ to world space $\mathbf{p}_{world}$ using the transformation matrix:
    $$ \mathbf{p}_{world} = \mathbf{M}_{shape\_to\_world} \cdot \mathbf{p}_{local} $$
    where $\mathbf{M}_{shape\_to\_world}$ is `_ShapeToWorld` [S1, unofficial].
4.  **SDF Evaluation:** Evaluate the SDF function $F(\mathbf{p})$ at the world position [S1, unofficial]. The function depends on the specific primitive shape defined by `SHAPE_SIZE` [S1, unofficial].
    $$ d = \text{SDF}(\mathbf{p}_{world}, \text{SHAPE\_SIZE}) $$
5.  **Storage:** Store the computed distance $d$ in the output array `_IsoVals` at the index corresponding to the voxel ID [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sdf_value(voxel_id, shape_to_world_matrix, shape_size, grid_dims, grid_origin):
    # Step 1 & 2: Get local position from grid ID using GetPosSurface(id) [S1, unofficial]
    # [G] Standard linear ID to 3D coordinate conversion for a regular grid
    x = (voxel_id % grid_dims[0]) / grid_dims[0]
    y = ((voxel_id // grid_dims[0]) % grid_dims[1]) / grid_dims[1]
    z = (voxel_id // (grid_dims[0] * grid_dims[1])) / grid_dims[2]
    local_pos = (x, y, z)

    # Step 3: Transform to world space using _ShapeToWorld [S1, unofficial]
    world_pos_x = shape_to_world_matrix[0][0] * local_pos[0] + shape_to_world_matrix[0][1] * local_pos[1] + shape_to_world_matrix[0][2] * local_pos[2] + shape_to_world_matrix[0][3]
    world_pos_y = shape_to_world_matrix[1][0] * local_pos[0] + shape_to_world_matrix[1][1] * local_pos[1] + shape_to_world_matrix[1][2] * local_pos[2] + shape_to_world_matrix[1][3]
    world_pos_z = shape_to_world_matrix[2][0] * local_pos[0] + shape_to_world_matrix[2][1] * local_pos[1] + shape_to_world_matrix[2][2] * local_pos[2] + shape_to_world_matrix[2][3]
    world_pos = (world_pos_x, world_pos_y, world_pos_z)

    # Step 4: Evaluate SDF function at the world position with SHAPE_SIZE [S1, unofficial]
    # [G] The specific primitive function depends on the shape type and uses shape_size as a parameter.
    # Common implementations: sphere (length(point) - size), box (length(max(abs(point) - size, 0.0)) - size)
    sdf_value = evaluate_sdf_primitive(world_pos, shape_size)

    # Step 5: Store result in _IsoVals at index voxel_id
    return sdf_value

def evaluate_sdf_primitive(point, size):
    # [G] Shape-specific SDF evaluation. Sources do not specify the exact primitive function.
    raise NotImplementedError("Specific SDF primitive function not defined in source.")
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision errors in the transformation matrix multiplication can cause slight inaccuracies in the distance value, potentially affecting marching convergence [G].
*   **Boundary Conditions:** The definition of `GetPosSurface` must correctly handle grid boundaries to avoid out-of-bounds access or incorrect coordinate mapping [G].
*   **Shape Definition:** The SDF evaluation must be consistent with the `SHAPE_SIZE` interpretation (e.g., radius vs. half-extent) [G].

## Subtopics

### [Per-Frame SDF Computation](6846-per-frame-sdf-computation.md)
Calculates SDF and RGB for 3D points relative to a single camera view, solving geometric and photometric consistency. It differs by incorporating view-dependent color and depth comparison, whereas this method is view-agnostic and geometry-only.

## Operating characteristics
Not given in the sources.

## Background
This method is derived from the `vinceniko/Isosurface` repository, which implements an isosurface extraction pipeline [S1, unofficial].

## Sources
[S1] vinceniko/Isosurface, https://github.com/vinceniko/Isosurface, fetched 2026-10-27 (unofficial)
