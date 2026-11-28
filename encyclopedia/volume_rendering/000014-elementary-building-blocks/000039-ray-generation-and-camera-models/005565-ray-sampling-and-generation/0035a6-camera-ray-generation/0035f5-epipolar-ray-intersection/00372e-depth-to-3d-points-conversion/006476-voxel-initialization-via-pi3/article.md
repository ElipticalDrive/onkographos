# Voxel Initialization via PI3

Also called: PI3-based Voxel Initialization

## Overview
Voxel Initialization via PI3 is a method for establishing a geometrically accurate Signed Distance Field (SDF) prior for volume rendering or reconstruction tasks using unposed images. It solves the problem of initializing sparse voxel grids from depth maps by leveraging a visual geometry model (PI3) to estimate 3D points in world coordinates, transforming them to ground truth poses, and allocating voxels with SDF values derived from minimum distances to these points [S1, unofficial]. It differs from sibling methods like [Depth Map to Absolute Camera Coordinates](7984-depth-map-to-absolute-camera-coordinates.md) by explicitly handling camera pose misalignment via Umeyama’s method and using PI3’s world-coordinate estimates rather than direct backprojection from known poses [S1, unofficial].

## How it works
**Inputs:**
*   Unposed images (or depth maps derived from them).
*   PI3 model outputs: Point maps in world coordinates ($P_{world}$).
*   Ground truth camera poses (or estimated poses to align with).
*   Voxel grid parameters: Corner position $v_{min}$, side length $v_l$.

**Outputs:**
*   Initialized sparse voxel grid with SDF values.

**Data Structures:**
*   **Voxel:** Parameterized by corner SDF values. A voxel $v$ is defined by its corner closest to the world origin $v_{min}$ and side length $v_l$ [S1, unofficial].
*   **Point Map:** Set of 3D points $p$ in world coordinates.

**Steps:**
1.  **Estimate World Points:** Use the PI3 model to estimate 3D points in world coordinates from the unposed images [S1, unofficial]. Let this set be $P_{world}$.
2.  **Transform to Ground Truth Poses:** Transform the points $P_{world}$ into the coordinate system of the ground truth camera poses using Umeyama’s method [S1, unofficial]. Let the transformed points be $P_{transformed}$.
3.  **Allocate Voxels:** Allocate voxels in the sparse grid at the locations of $P_{transformed}$ [S1, unofficial].
4.  **Compute SDF:** For each allocated voxel, compute the SDF value by calculating the minimum Euclidean distance from the voxel to the points in $P_{transformed}$ [S1, unofficial].

**Formulas:**
The SDF for a voxel is based on the minimum distance to the transformed points. The local coordinate $q$ of a point $p$ within a voxel $v$ is defined as [S1, unofficial]:
$$ q = \frac{p - v_{min}}{v_l} $$
where $v_{min}$ is the voxel corner closest to the world origin and $v_l$ is the voxel side length. The SDF value is derived from the minimum distance from the transformed points to the voxel geometry.

**Parameters:**
*   $v_{min}$: Voxel corner position.
*   $v_l$: Voxel side length.
*   PI3 Model: Visual geometry model for 3D point estimation.
*   Umeyama’s Method: Rigid body transformation estimation.

**Edge Cases and Pitfalls:**
*   **Precision:** The accuracy of the initialization depends heavily on the PI3 model's ability to estimate correct world coordinates and the precision of Umeyama’s transformation [S1, unofficial].
*   **Boundaries:** Voxels are only allocated at transformed points; empty space is not explicitly initialized via free-space carving in this specific PI3-based step, relying on the SDF formulation for convergence [S1, unofficial].
*   **Degenerate Input:** If PI3 fails to estimate points or Umeyama’s method encounters degenerate configurations, the initialization will be incomplete or incorrect [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_voxels_via_pi3(unposed_images, ground_truth_poses, pi3_model, umeyama_transform, voxel_config):
    """
    Initialize voxels using PI3 and Umeyama's method.
    
    Args:
        unposed_images: List of unposed images or depth maps.
        ground_truth_poses: Ground truth camera poses.
        pi3_model: PI3 visual geometry model.
        umeyama_transform: Function to apply Umeyama's method.
        voxel_config: Configuration for voxel grid (v_min, v_l).
        
    Returns:
        voxel_grid: Initialized sparse voxel grid with SDF values.
    """
    # Step 1: Estimate 3D points in world coordinates using PI3
    # PI3 outputs point maps in world coordinates
    points_world = pi3_model.estimate_3d_points(unposed_images)
    
    # Step 2: Transform points to ground truth camera poses using Umeyama's method
    # Umeyama's method computes the rigid transformation between two point sets
    # Here, we assume we are aligning PI3's world points to the ground truth coordinate system
    # If ground_truth_poses are in a different frame, we compute the transformation
    # Note: The exact implementation of Umeyama's method involves finding rotation R and translation t
    # that minimize the error between two point sets.
    transformed_points = umeyama_transform(points_world, ground_truth_poses)
    
    # Step 3: Allocate voxels at the transformed points
    voxel_grid = SparseVoxelGrid(voxel_config)
    for point in transformed_points:
        voxel_grid.allocate_voxel_at(point)
        
    # Step 4: Compute SDF for each allocated voxel
    # SDF is the minimum distance from the voxel to the transformed points
    for voxel in voxel_grid:
        min_dist = infinity
        for point in transformed_points:
            dist = euclidean_distance(voxel.center, point)
            if dist < min_dist:
                min_dist = dist
        voxel.sdf = min_dist
        
    return voxel_grid
```

## Subtopics

### Testing Points Generation
A method to generate testing points $P_{t\{i\}}$ from $P_{v\{i\}}$ and sensor origin to update distance information for voxels within the current sensor field of view, improving efficiency over conventional free-space carving [S2]. It ray-casts from the sensor origin to each voxel in $P_{v\{i\}}$, queries only voxels with previous surface measurements, generates voxels along surface normals up to a certain distance, and uses L-GPDF to infer distance, surface properties, and variances [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the PI3 model inference time and the number of points. Umeyama’s method is $O(N)$ for $N$ points. SDF computation is $O(N \cdot M)$ where $N$ is points and $M$ is voxels, but optimized by allocating only at points [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of allocated voxels and points [S1, unofficial].
*   **Strengths:** Provides a stable and physically meaningful initialization, enabling faster convergence for SDF-based learning [S1, unofficial]. Handles unposed images effectively [S1, unofficial].
*   **Limitations:** Relies on the accuracy of PI3 and Umeyama’s method. Does not explicitly handle free-space carving in the initialization step [S1, unofficial].
*   **When to Use:** When initializing SDF-based reconstruction from unposed images is required, and a geometric prior is beneficial for convergence [S1, unofficial].
*   **When Not to Use:** When ground truth poses are not available for Umeyama’s alignment, or when PI3 is not suitable for the scene [S1, unofficial].

## Background
Originated in SVRecon, which uses PI3, a visual geometry model, to estimate 3D points in world coordinates from unposed images, transforming them via Umeyama’s method to initialize SDF-based voxel grids [S1, unofficial]. It is used in sparse voxel rasterization for surface reconstruction [S1, unofficial].

## Sources
[S1] SVRecon: Sparse Voxel Rasterization for Surface Reconstruction, https://doi.org/10.48550/arxiv.2511.17364, fetched 2026-10-26 (unofficial)
[S2] VDB-GPDF: Online Gaussian Process Distance Field With VDB Structure, https://doi.org/10.1109/lra.2024.3505814, fetched 2026-10-26
