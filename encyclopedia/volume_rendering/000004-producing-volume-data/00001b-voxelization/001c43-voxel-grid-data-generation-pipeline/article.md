# Voxel Grid Data Generation Pipeline

Also called: Labeled Point Cloud Voxelization with Occlusion Handling, Voxel Grid Generation from Scans

## Overview
This method generates labeled 3D voxel grids from sequential scan data, specifically designed to handle occlusions and temporal consistency between prior and current observations. It differs from standard voxelization by maintaining two distinct grid states (prior and past) and explicitly updating occlusion masks based on camera pose transformations, ensuring that newly observed geometry correctly occludes previous estimates. It builds on [Voxelization](12-voxelization.md) by adding a temporal, scan-based processing loop.

## How it works
**Inputs:**
- A sequence of poses (`poses`), where each pose is a 4x4 transformation matrix.
- Point clouds associated with each pose (`pastPoints`, `priorPoints`).
- Labels associated with each point (`pastLabels`, `priorLabels`).
- Configuration parameters: `stride_num`, `stride_distance`.
- Output directory and naming convention.

**Outputs:**
- Two voxel grids: `priorGrid` and `pastGrid`, saved to disk.

**Data Structures:**
- `VoxelGrid`: Contains an occupancy map and an occlusion mask. The specific internal layout is implementation-dependent but must support `updateOcclusions()` and `updateInvalid()`.
- `Pose`: 4x4 matrix representing camera position and orientation.

**Steps:**

1. **Initialization**: Initialize `priorGrid` and `pastGrid` with the target resolution and bounds.

2. **Iteration Loop**: Iterate through the scan sequence. For each current scan index `current`:
   a. Retrieve `priorPoints`, `priorLabels`, `pastPoints`, and `pastLabels` for the current index.
   b. **Label Counting**:
      - Initialize `labelCount = 0`.
      - Iterate through `pastLabels`. If `label > 0`, increment `labelCount`.
      - Calculate `pointCount` as the total number of points in `pastPoints`.
      - Calculate `percentageLabeled = 100.0f * labelCount / pointCount`.
   c. **Validation**: If `percentageLabeled <= 0.0f`, skip grid generation for this step to avoid empty or invalid data [S1, unofficial].
   d. **Occlusion Update**:
      - Call `priorGrid.updateOcclusions()`.
      - Call `pastGrid.updateOcclusions()`.
   e. **Invalid Voxel Marking**:
      - Iterate through `pastPoints`.
      - For each point `i`, compute the endpoint in the anchor frame:
        $$ \text{endpoint} = (\text{anchor\_pose}^{-1} \times \text{pastPoints}[i].\text{pose}).\text{col}(3).\text{head}(3) $$
        where `anchor_pose` is the reference frame transformation, `col(3)` extracts the translation vector, and `head(3)` takes the first three components [S1, unofficial].
      - Call `pastGrid.updateInvalid(endpoint)` to mark voxels along the ray or at the endpoint as invalid/occluded [S1, unofficial].
   f. **Save Grids**:
      - Call `saveVoxelGrid(priorGrid, output_voxel_dirname, outname.str(), "input")`.
      - Call `saveVoxelGrid(pastGrid, output_voxel_dirname, outname.str(), "target")`.
   g. **Advance Index**:
      - Initialize `distance = 0`, `count = 0`.
      - While `(count < config.stride_num || distance < config.stride_distance || count == 0) && current + 1 < poses.size()`:
        - `distance += poseDistance(poses[current], poses[current + 1])`.
        - `count += 1`.
        - `current += 1`.
      - Break if no further scan is possible.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_grids(poses, pastPoints, pastLabels, priorPoints, priorLabels, config, anchor_pose, output_dir, outname):
    # Initialize grids
    priorGrid = VoxelGrid()
    pastGrid = VoxelGrid()
    
    current = 0
    while current < len(poses):
        # Retrieve data for current index
        # Note: In a real implementation, these would be fetched based on 'current'
        # Assuming pastPoints/Labels etc. are aligned with poses[current]
        
        # Count labeled points
        labelCount = 0
        pointCount = len(pastLabels)
        for label in pastLabels:
            if label > 0:
                labelCount += 1
        
        percentageLabeled = 100.0 * labelCount / pointCount
        
        # Skip if no labels
        if percentageLabeled <= 0.0:
            current += 1
            continue
            
        # Update occlusions
        priorGrid.updateOcclusions()
        pastGrid.updateOcclusions()
        
        # Mark invalid voxels
        for i in range(len(pastPoints)):
            # Compute endpoint in anchor frame
            # Assuming pastPoints[i].pose is a 4x4 matrix or similar structure
            # anchor_pose.inverse() * pastPoints[i].pose
            transformed = matrix_multiply(inverse(anchor_pose), pastPoints[i].pose)
            endpoint = transformed.get_column(3)[:3] # col(3).head(3)
            pastGrid.updateInvalid(endpoint)
            
        # Save grids
        saveVoxelGrid(priorGrid, output_dir, outname, "input")
        saveVoxelGrid(pastGrid, output_dir, outname, "target")
        
        # Advance scan index
        distance = 0.0
        count = 0
        while (count < config.stride_num or distance < config.stride_distance or count == 0) and current + 1 < len(poses):
            distance += pose_distance(poses[current], poses[current + 1])
            count += 1
            current += 1
            
        if current >= len(poses):
            break

def pose_distance(pose1, pose2):
    # Euclidean distance between translation components
    t1 = pose1.get_translation()
    t2 = pose2.get_translation()
    return vector_norm(t1 - t2)
```

**Parameters:**
- `stride_num`: Maximum number of scans to skip in one stride [S1, unofficial].
- `stride_distance`: Maximum distance between scans to allow a stride [S1, unofficial].

**Edge Cases:**
- **Empty Labels**: If `percentageLabeled <= 0.0`, the grid generation is skipped to prevent invalid output [S1, unofficial].
- **Boundary Conditions**: The loop breaks if `current + 1 >= poses.size()` [S1, unofficial].

## Subtopics

### [Voxel Grid Generation & Expansion](2979-voxel-grid-generation-expansion.md)
Generates a 3D occupancy grid of voxels representing a smoke volume, expanded over time from an origin point using CPU iteration and GPU buffer upload [2979-voxel-grid-generation-expansion.md].

### [Scene Data Loading & Resizing](3420-scene-data-loading-resizing.md)
Loads COLMAP camera intrinsics/extrinsics and 3D points, resizes images to a target resolution, adjusts coordinate system, and subsamples points if exceeding a hardware limit [3420-scene-data-loading-resizing.md].

### [DCX Pipeline](3770-dcx-pipeline.md)
A pipeline for converting point clouds to meshes using voxelization, cube type classification, thinning, and reconstruction, exposed via a Python binding module named DCX_main [3770-dcx-pipeline.md].

### Direct Geometry Scan-Conversion
Scan-converts the geometry directly into the sampled array and renders the ensemble, allowing for true 3-D merge of visualizations [S2, unofficial].

### KITTI Dataset Preprocessing
Prepares KITTI dataset for training by parsing camera extrinsics/intrinsics, downsampling point clouds, adding background shell, and splitting into train/validation sets [S3, unofficial].
- **Definition**: Prepares KITTI dataset for training by parsing camera extrinsics/intrinsics, downsampling point clouds, adding background shell, and splitting into train/validation sets [S3, unofficial].
- **Steps**:
  1. Parse camera extrinsics from XML to get `T_pointcloud_camera`, `label`, `sensor_id`, `path`.
  2. Parse intrinsics to get `K`, `width`, `height`.
  3. Load point cloud from PLY.
  4. Downsample point cloud by 99% (`frac=0.01`).
  5. Add 1000 random shell points around scene center/radius.
  6. Save to parquet.
  7. Split frames every 3 for train/val.
  8. Save JSON metadata [S3, unofficial].
- **Formula**: Intrinsic matrix $K$ is computed as:
  $$ K = \begin{bmatrix} f & 0 & w/2 \\ 0 & f & h/2 \\ 0 & 0 & 1 \end{bmatrix} $$
  where $f$ is focal length, $w$ and $h$ are image dimensions [S3, unofficial].
- **Parameters**:
  - `frac=0.01`: downsampling fraction for the point cloud [S3, unofficial].
  - `num_shell_points=1000`: number of background points added around the scene center [S3, unofficial].
  - `train every 3 frames`: ratio used to split data into training and validation sets [S3, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The core pipeline described in [S1, unofficial] is part of the `jbehley/voxelizer` repository, which is an unofficial implementation likely related to 3D reconstruction or SLAM systems. The concept of direct geometry scan-conversion for merging volume and geometric data originates from earlier work on volume rendering and visualization, as cited in [S2, unofficial]. The KITTI dataset preprocessing steps are derived from a Taichi-based 3D Gaussian Splatting implementation [S3, unofficial].

## Sources
[S1] jbehley/voxelizer, https://github.com/jbehley/voxelizer, fetched 2026-10-27 (unofficial)
[S2] DTIC ADA193649: Rendering of Surfaces from Volume Data,, https://archive.org/details/DTIC_ADA193649, fetched 2026-10-26 (unofficial)
[S3] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
