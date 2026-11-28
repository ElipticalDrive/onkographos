# KITTI Dataset Preprocessing

Also called: KITTI Voxelization Prep

## Overview
Prepares the KITTI dataset for training by parsing camera extrinsics and intrinsics, downsampling point clouds, adding a background shell, and splitting data into train/validation sets [S1, unofficial]. It differs from sibling methods like [DCX Pipeline](3770-dcx-pipeline.md) by focusing on raw LiDAR-to-voxel conversion for autonomous driving scenes rather than synthetic or facial data.

## How it works
**Inputs:** KITTI raw data (images, PLY point clouds, XML calibration files).
**Outputs:** Parquet files containing downsampled point clouds and metadata; JSON metadata files.

**Data Structures:**
1.  **Camera Intrinsics ($K$):** $3 \times 3$ matrix.
    $$K = \begin{bmatrix} f & 0 & w/2 \\ 0 & f & h/2 \\ 0 & 0 & 1 \end{bmatrix}$$
    Where $f$ is focal length, $w$ is width, $h$ is height [S1, unofficial].
2.  **Camera Extrinsics ($T_{pointcloud\_camera}$):** $4 \times 4$ transformation matrix from camera to point cloud frame, parsed from XML [S1, unofficial].
3.  **Point Cloud:** Array of $(x, y, z)$ coordinates.
4.  **Background Shell:** 1000 random points added around scene center [S1, unofficial].

**Steps:**
1.  **Parse Calibration:** Read XML to extract $T_{pointcloud\_camera}$, label, sensor_id, path, and compute $K$ [S1, unofficial].
2.  **Load Point Cloud:** Read PLY file [S1, unofficial].
3.  **Downsample:** Reduce point cloud size by 99% using `frac=0.01` [S1, unofficial].
4.  **Add Background:** Generate 1000 random points around the scene center/radius to form a background shell [S1, unofficial].
5.  **Save:** Store processed data in Parquet format [S1, unofficial].
6.  **Split Data:** Divide frames every 3 frames into training and validation sets [S1, unofficial].
7.  **Save Metadata:** Export split information to JSON [S1, unofficial].

**Parameters:**
*   `frac=0.01`: Downsampling fraction [S1, unofficial].
*   `num_shell_points=1000`: Number of background points [S1, unofficial].
*   `train every 3 frames`: Split ratio [S1, unofficial].

**Edge Cases:**
*   Precision issues in quaternion multiplication if not normalized [S1, unofficial].
*   Boundary conditions for random shell generation [S1, unofficial].

## Subtopics

### Head Alignment and Pose Estimation
Estimates camera parameters and aligns heads to a predefined canonical pose during data preprocessing [S2]. For studio settings, cameras are calibrated and 3D keypoints are estimated by triangulating detected 2D keypoints [S2]. For in-the-wild captures, Mediapipe is used to estimate camera positions and 3D keypoints [S2]. A similarity transform aligns five 3D keypoints (outer eye corners, nose, mouth centre, and chin) to a canonical pose defined as the median location of these keypoints across the first 260 identities of the training set [S2].

## Operating characteristics
Time and memory complexity depend on point cloud size and downsampling factor. Performance scales with resolution and scene complexity. Strengths include efficient data preparation for voxel-based training. Limitations include potential loss of detail due to aggressive downsampling.

## Background
Originated from the KITTI dataset preprocessing pipeline for 3D Gaussian Splatting [S1, unofficial]. Used in autonomous driving and volumetric rendering research.

## Sources
[S1] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
[S2] Preface: A Data-driven Volumetric Prior for Few-shot Ultra High-resolution Face Synthesis, https://doi.org/10.1109/iccv51070.2023.00315, fetched 2026-10-27
