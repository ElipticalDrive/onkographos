# Trajectory Alignment (RANSAC)

Also called: RANSAC Trajectory Alignment, Trajectory Alignment via RANSAC

## Overview
Aligns a sparse camera trajectory to a ground truth trajectory using RANSAC-based point cloud registration [S1, unofficial]. It differs from standard global alignment by handling sparse, potentially noisy inputs through iterative sampling, and optionally sparsifies the input trajectory if a mapping file is provided and the trajectory length exceeds 1600 frames [S1, unofficial].

## How it works
**Inputs:**
- `gt_traj_col`: Ground truth trajectory (column-major format).
- `map_file`: Optional mapping file containing `n_sampled_frames`, `n_total_frames`, and a mapping array where each row is `[frame_index, total_frame_index]` [S1, unofficial].
- `gt_trans`: Optional transformation matrix to apply to the ground truth point cloud [S1, unofficial].
- `randomvar`: Noise magnitude multiplier. Default: 0.0. If `< 1e-5`, no noise is added [S1, unofficial].

**Outputs:**
- Aligned trajectory poses.

**Data Structures:**
- Point Cloud: A set of 3D points derived from trajectory camera positions.
- Correspondence Vector: A list of index pairs `[i, i]` indicating self-correspondence between the source and target point clouds [S1, unofficial].

**Steps:**
1. **Parse Mapping:** If `map_file` exists, parse `n_sampled_frames`, `n_total_frames`, and the mapping array [S1, unofficial].
2. **Sparsify (Conditional):** If `map_file` is provided AND the trajectory length `> 1600`, use the mapping to select specific frames. If trajectory length `<= 1600` or no `map_file` is present, use the full trajectory [S1, unofficial].
3. **Convert to Point Cloud:** Convert the selected ground truth trajectory `gt_traj_col` into a point cloud `traj_pcd_col` [S1, unofficial].
4. **Transform Ground Truth:** If `gt_trans` is provided, apply it to `traj_pcd_col` [S1, unofficial].
5. **Create Correspondences:** Generate a correspondence vector `corres` where each point in the source trajectory corresponds to itself in the target trajectory: `corres = [[0,0], [1,1], ..., [N-1, N-1]]` [S1, unofficial].
6. **RANSAC Alignment:** Perform RANSAC registration using `traj_pcd_col` and the source trajectory point cloud with the following parameters:
   - `max_iteration`: 100,000 [S1, unofficial].
   - `max_validation`: 100,000 [S1, unofficial].
   - Add noise to the source points scaled by `randomvar` if `randomvar >= 1e-5` [S1, unofficial].
7. **Apply Transformation:** Apply the best-fit transformation found by RANSAC to the source trajectory [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import open3d as o3d

def trajectory_alignment_ransac(gt_traj_col, map_file=None, gt_trans=None, randomvar=0.0):
    """
    Aligns a trajectory to a ground truth trajectory using RANSAC.
    
    Args:
        gt_traj_col: Ground truth trajectory in column-major format.
        map_file: Optional path to mapping file.
        gt_trans: Optional transformation matrix for ground truth.
        randomvar: Noise magnitude multiplier.
        
    Returns:
        Aligned trajectory.
    """
    # Step 1 & 2: Parse mapping and sparsify if necessary
    # Note: Implementation details for map_file parsing depend on specific file format.
    # Assuming a helper function exists or the data is pre-loaded.
    # If map_file is provided and len(gt_traj_col) > 1600, select frames based on mapping.
    # Otherwise, use full trajectory.
    # [G] Placeholder for actual file parsing logic not detailed in source.
    if map_file is not None and len(gt_traj_col) > 1600:
        # Parse map_file to get indices
        # indices = parse_mapping_file(map_file)
        # selected_traj = gt_traj_col[indices]
        selected_traj = gt_traj_col # Placeholder
    else:
        selected_traj = gt_traj_col

    # Step 3: Convert to point cloud
    # [G] Assuming convert_trajectory_to_pointcloud extracts positions from trajectory.
    traj_pcd_col = convert_trajectory_to_pointcloud(selected_traj)

    # Step 4: Transform ground truth if provided
    if gt_trans is not None:
        traj_pcd_col.transform(gt_trans)

    # Step 5: Create correspondence vector
    # Correspondence is self-matching: point i in source matches point i in target
    n_points = len(selected_traj)
    corres_indices = np.array([[i, i] for i in range(n_points)])
    corres = o3d.utility.Vector2iVector(corres_indices)

    # Step 6: RANSAC Alignment
    # [G] Assuming source_traj_pcd is the trajectory to be aligned.
    # source_traj_pcd = convert_trajectory_to_pointcloud(source_trajectory)
    
    # Add noise if randomvar >= 1e-5
    if randomvar >= 1e-5:
        # [G] Add Gaussian noise scaled by randomvar to source points
        pass 

    # Perform RANSAC registration
    # [G] open3d.registration_ransac_based_on_correspondence is used here.
    # max_correspondence_distance and other criteria are not specified in sources.
    # Using default values or typical values for point cloud registration.
    # max_iteration=100000, max_validation=100000 are specified.
    
    # result = o3d.registration.registration_ransac_based_on_correspondence(
    #     source_traj_pcd,
    #     traj_pcd_col,
    #     corres,
    #     max_correspondence_distance=0.1, # [G] Not given in sources. Typical value.
    #     estimation_method=o3d.pipelines.registration.TransformationEstimationPointToPoint(),
    #     ransac_n=3, # [G] Not given. Typical for point-to-point.
    #     checks=100000,
    #     criteria=o3d.pipelines.registration.RANSACConvergenceCriteria(100000, 100000)
    # )
    
    # Apply result transformation to source trajectory
    # aligned_traj = apply_transformation(source_trajectory, result.transformation)
    
    return selected_traj # Placeholder return

def convert_trajectory_to_pointcloud(traj):
    """
    Converts a trajectory to a point cloud.
    [G] Implementation detail not provided in source. Assumes extraction of 3D positions.
    """
    pcd = o3d.geometry.PointCloud()
    # Extract positions from traj
    # pcd.points = o3d.utility.Vector3dVector(traj_positions)
    return pcd
```

## Subtopics

### Pose Alignment (RANSAC-inspired)
For fewer than 9 input views, performs alignment by sampling every possible pair of cameras in one set, computing the Sim(3) transformation relating it to the same camera pair in the other trajectory. The global Sim(3) transformation leading to the lowest average camera alignment error is kept [S2]. For 9 or more views, it uses the standard Umeyama algorithm [S2].

## Operating characteristics
- **Time Complexity:** High for the RANSAC variant due to 100,000 iterations and validation steps [S1, unofficial]. The RANSAC-inspired variant for <9 views involves checking all camera pairs, which is $O(N^2)$ for $N$ views [S2].
- **Memory Complexity:** Linear with respect to the number of trajectory points for point cloud storage [S1, unofficial].
- **Strengths:** Robust to noise and outliers in sparse trajectories [S1, unofficial]. Handles very few views (<9) where standard algorithms fail [S2].
- **Limitations:** Unstable for very few views if not using the RANSAC-inspired approach [S2]. Sparsification is only applied if trajectory length > 1600 [S1, unofficial].
- **When to use:** When aligning sparse, noisy trajectories where standard global alignment (e.g., Umeyama) is unstable due to insufficient views [S2].

## Background
Originated in the context of 2D Gaussian Splatting and sparse pose alignment for NeRF reconstruction [S1, unofficial][S2]. The RANSAC-based approach was adopted to handle the instability of standard alignment methods (like Umeyama) when dealing with very few input views (<9) [S2]. The sparsification logic is specific to the 2D Gaussian Splatting implementation [S1, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
