# Extrapolated View Partitioning Procedure

Also called: Testing View Design for Extrapolation

## Overview
The Extrapolated View Partitioning Procedure generates a sparse trajectory for volume rendering by creating testing views that lie outside the convex hull of the training data, thereby enabling the evaluation and training of models on extrapolated geometry [S1]. It differs from sibling methods like Trajectory Alignment (RANSAC) by focusing on the deliberate spatial partitioning of camera poses to create "relative" or absolute extrapolation scenarios rather than aligning existing trajectories [S1, 7876-trajectory-alignment-ransac.md]. This method ensures that ground truth images exist for these extrapolated views, which is critical for supervised learning or validation in datasets where free-form trajectory generation is constrained by data availability [S1].

## How it works
The procedure adapts its strategy based on the dataset characteristics and the availability of ground truth for extrapolated regions. It involves three primary phases: trajectory design, view partitioning, and optional point cloud generation for geometric supervision.

### Inputs and Outputs
*   **Inputs**: A full camera trajectory (sequence of poses), a dataset type identifier (e.g., Waymo, LLFF, Mip-NeRF 360), and optionally a dense correspondence model (e.g., MASt3R) for SA-Points generation [S1, S2].
*   **Outputs**: A partitioned set of training views and testing (extrapolated) views, and optionally a set of 3D points (SA-Points) derived from these views [S1, S2].

### Data Structures
*   **Camera Pose**: A 4x4 transformation matrix or a tuple $(t_x, t_y, t_z, r_x, r_y, r_z)$ representing position and orientation.
*   **View Set**: A list of camera poses classified as `training` or `testing`.
*   **SA-Points**: A list of 3D coordinates $(X, Y, Z)$ filtered by reprojection error [S2].

### Steps

1.  **Dataset-Specific Trajectory Design**:
    *   If the dataset is **Waymo**: Generate testing views by applying a rigid transformation to the training camera. Translate the camera by $2$ to $3$ meters or rotate it by $10$ to $15$ degrees horizontally [S1].
    *   If the dataset is **LLFF** or **Mip-NeRF 360**: Partition the existing full trajectory into training and testing sets. The partitioning must ensure that the testing views correspond to locations where ground truth images are available, as free-form trajectory generation is not permitted [S1].
    *   If the scene lacks obvious extrapolated trajectories: Sparsify the training views to create "relative" extrapolated trajectories. This involves selecting a subset of training views such that the remaining views (or interpolated gaps) serve as extrapolation targets [S1].

2.  **SA-Points Generation (Optional Geometric Augmentation)**:
    If geometric consistency is required for the extrapolated views, generate 3D points using the following sub-procedure [S2]:
    *   **Dense Correspondence**: Compute dense correspondences between the ground-truth image of an original training view and the rendering from an extrapolated view using a pretrained model (e.g., MASt3R ViTLarge_BaseDecoder_512_catmlpdpt_metric) [S2].
    *   **Subsampling**: Subsample correspondence pairs using a spatial stride of $5$ pixels along both the $x$ and $y$ directions for efficiency [S2].
    *   **Triangulation**: Triangulate 3D points from the matched correspondences. Parallelize this step across multiple CPU threads [S2].
    *   **Reprojection Filtering**: Discard points whose bidirectional reprojection error exceeds $1$ pixel [S2].

### Parameters
*   **Waymo Translation**: $2$ to $3$ meters [S1].
*   **Waymo Rotation**: $10$ to $15$ degrees horizontally [S1].
*   **SA-Points Reprojection Threshold**: $1$ pixel [S2].
*   **SA-Points Spatial Stride**: $5$ pixels [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_extrapolated_views(dataset_type, full_trajectory):
    """
    Generates training and testing view partitions based on dataset type.
    """
    training_views = []
    testing_views = []

    if dataset_type == "Waymo":
        # Translate or rotate camera by specified amounts
        for pose in full_trajectory:
            # Create extrapolated pose
            extrapolated_pose = apply_translation(pose, dx=2.5, dy=0, dz=0) # Example: 2.5m translation
            # Or apply rotation
            # extrapolated_pose = apply_rotation(pose, angle=12.5, axis='horizontal')
            
            training_views.append(pose)
            testing_views.append(extrapolated_pose)
            
    elif dataset_type in ["LLFF", "Mip-NeRF 360"]:
        # Partition existing trajectory to ensure ground truth exists
        # Logic depends on specific scene structure; typically split by spatial region
        # Not given in the sources: specific partitioning algorithm for LLFF/Mip-NeRF 360
        # Placeholder for dataset-specific partitioning logic
        training_views, testing_views = partition_trajectory(full_trajectory)
        
    else:
        # Sparsify training views to create relative extrapolation
        # Select a sparse subset as training, rest as testing
        stride = select_sparse_stride(full_trajectory)
        for i, pose in enumerate(full_trajectory):
            if i % stride == 0:
                training_views.append(pose)
            else:
                testing_views.append(pose)

    return training_views, testing_views

def generate_sa_points(training_view_gt, extrapolated_view_rendering):
    """
    Generates 3D points from correspondences between training and extrapolated views.
    """
    # 1. Dense Correspondence using MASt3R
    # Not given in the sources: exact API call for MASt3R
    correspondences = mast3r_compute_correspondences(training_view_gt, extrapolated_view_rendering)
    
    # 2. Subsample correspondences
    subsampled_correspondences = subsample_with_stride(correspondences, stride=5)
    
    # 3. Triangulate 3D points (parallelized)
    # Not given in the sources: exact triangulation implementation
    points_3d = triangulate_matches(subsampled_correspondences)
    
    # 4. Filter by reprojection error
    valid_points = []
    for point, match in zip(points_3d, subsampled_correspondences):
        error = compute_bidirectional_reprojection_error(point, match)
        if error <= 1.0: # 1 pixel threshold
            valid_points.append(point)
            
    return valid_points
```

### Edge Cases and Pitfalls
*   **Ground Truth Availability**: In LLFF and Mip-NeRF 360, generating arbitrary extrapolated trajectories is impossible if ground truth images do not exist for those poses. The method must rely on partitioning existing data rather than synthesizing new poses [S1].
*   **Geometric Artifacts**: Extrapolated renderings may contain geometric artifacts. The use of robust dense correspondence models (like MASt3R) is necessary to handle these distortions during SA-Points generation [S2].
*   **Uncertainty vs. Certainty**: Using uncertainty masks as guidance for denoising can lead to significant inconsistency. The method prefers certainty masks (or $1 - M_{\bar{c}}$) for guidance to maintain consistency [S1].

## Subtopics

### SA-Points Generation (MASt3R)
A variant of the geometric augmentation step that uses the MASt3R model to generate 3D points from extrapolated views. It is chosen when high-fidelity geometric structure is needed from single-view observations augmented with extrapolated viewpoints. It differs from other lifting methods by using context-aware dense correspondences over 16x16 local patches, making it robust to moderate viewpoint perturbations and geometric artifacts in synthesized views [S2].

## Operating characteristics
*   **Time Complexity**: The SA-Points generation step is computationally intensive due to dense correspondence matching and triangulation. However, it is optimized by subsampling correspondences with a spatial stride of 5 pixels and parallelizing triangulation across CPU threads [S2].
*   **Memory Complexity**: Depends on the resolution of the input images and the number of correspondences. The subsampling reduces memory footprint significantly [S2].
*   **Strengths**: Ensures that extrapolated views have ground truth for evaluation (LLFF/Mip-NeRF 360) or creates controlled extrapolation scenarios (Waymo). The SA-Points generation provides robust geometric supervision despite rendering artifacts [S1, S2].
*   **Limitations**: Restricted to datasets with existing ground truth for extrapolated regions (LLFF/Mip-NeRF 360). Waymo-specific parameters (2-3m translation) may not generalize to other datasets without adjustment [S1].
*   **When to Use**: When evaluating or training volume rendering models on extrapolated geometry is required, and ground truth data is available for the extrapolated poses.

## Background
The method originates from research into boosting 3D Gaussian Splatting via diffusion models and self-augmented residual splatting. It addresses the challenge of training and evaluating neural radiance fields and Gaussian splats in regions outside the observed camera poses [S1, S2]. The specific partitioning strategies for LLFF and Mip-NeRF 360 were developed to handle the constraint that ground truth images must exist for testing views [S1]. The SA-Points generation procedure was introduced to leverage dense correspondence models for geometric consistency in next-best-view selection and self-augmentation [S2].

## Sources
[S1] FreeFix: Boosting 3D Gaussian Splatting via Fine-Tuning-Free Diffusion Models, https://doi.org/10.1109/3dv69130.2026.00109, fetched 2026-10-30
[S2] SA-ResGS: Self-augmented Residual 3D Gaussian Splatting for Next Best View Selection, https://doi.org/10.1007/978-3-032-37013-6_17, fetched 2026-10-29
