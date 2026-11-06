# Voxel Grid Data Generation Pipeline

Also called: Labeled Point Cloud Voxelization with Occlusion Handling, Voxel Grid Generation from Scans

## Overview
This method generates labeled 3D voxel grids from sequential scan data, specifically designed to handle occlusions and temporal consistency between prior and current observations [S1, unofficial]. It differs from standard voxelization by maintaining two distinct grid states (prior and past) and explicitly updating occlusion masks based on camera pose transformations, ensuring that newly observed geometry correctly occludes previous estimates [S1, unofficial]. It builds on [Voxelization](12-voxelization.md) by adding a temporal, scan-based processing loop [G].

## How it works
**Inputs:**
- A sequence of poses (`poses`), where each pose is a 4x4 transformation matrix [S1, unofficial].
- Point clouds associated with each pose (`pastPoints`, `priorPoints`) [S1, unofficial].
- Labels associated with each point (`pastLabels`, `priorLabels`) [S1, unofficial].
- Configuration parameters: `stride_num`, `stride_distance` [S1, unofficial].
- Output directory and naming convention (`output_voxel_dirname`, `outname`) [S1, unofficial].

**Outputs:**
- Two voxel grids: `priorGrid` and `pastGrid`, saved to disk [S1, unofficial].

**Data Structures:**
- `VoxelGrid`: Contains an occupancy map and an occlusion mask. The specific internal layout is implementation-dependent but must support `updateOcclusions()` and `updateInvalid()` [S1, unofficial].
- `Pose`: 4x4 matrix representing camera position and orientation [G].

**Steps:**

1. **Initialization**: Initialize `priorGrid` and `pastGrid` with the target resolution and bounds [G].

2. **Iteration Loop**: Iterate through the scan sequence [S1, unofficial]. For each current scan index `current`:
   a. Retrieve `priorPoints`, `priorLabels`, `pastPoints`, and `pastLabels` for the current index [S1, unofficial].
   b. **Label Counting**:
      - Initialize `labelCount = 0`.
      - Iterate through `pastLabels`. If `label > 0`, increment `labelCount` [S1, unofficial].
      - Calculate `pointCount` as the total number of points in `pastPoints` [S1, unofficial].
      - Calculate `percentageLabeled = 100.0f * labelCount / pointCount` [S1, unofficial].
   c. **Validation**: If `percentageLabeled <= 0.0f`, skip grid generation for this step to avoid empty or invalid data [S1, unofficial].
   d. **Occlusion Update**:
      - Call `priorGrid.updateOcclusions()` [S1, unofficial].
      - Call `pastGrid.updateOcclusions()` [S1, unofficial].
   e. **Invalid Voxel Marking**:
      - Iterate through `pastPoints` [S1, unofficial].
      - For each point `i`, compute the endpoint in the anchor frame:
        $$ \text{endpoint} = (\text{anchor\_pose}^{-1} \times \text{pastPoints}[i].\text{pose}).\text{col}(3).\text{head}(3) $$
        where `anchor_pose` is the reference frame transformation, `col(3)` extracts the translation vector, and `head(3)` takes the first three components [S1, unofficial].
      - Call `pastGrid.updateInvalid(endpoint)` to mark voxels along the ray or at the endpoint as invalid/occluded [S1, unofficial].
   f. **Save Grids**:
      - Call `saveVoxelGrid(priorGrid, output_voxel_dirname, outname.str(), "input")` [S1, unofficial].
      - Call `saveVoxelGrid(pastGrid, output_voxel_dirname, outname.str(), "target")` [S1, unofficial].
   g. **Advance Index**:
      - Initialize `distance = 0`, `count = 0` [G].
      - While `(count < config.stride_num || distance < config.stride_distance || count == 0) && current + 1 < poses.size()`:
        - `distance += poseDistance(poses[current], poses[current + 1])` [S1, unofficial].
        - `count += 1` [G].
        - `current += 1` [G].
      - Break if no further scan is possible [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class VoxelGrid:
    def __init__(self):
        self.occlusion_mask = {}
    def updateOcclusions(self):
        # Implementation-dependent occlusion update logic [G]
        pass
    def updateInvalid(self, endpoint):
        # Mark voxel at endpoint as invalid [G]
        self.occlusion_mask[tuple(endpoint)] = True

def saveVoxelGrid(grid, output_dir, outname, suffix):
    # Save grid data to disk [G]
    filepath = f"{output_dir}/{outname}_{suffix}.bin"
    with open(filepath, 'wb') as f:
        f.write(str(grid.occlusion_mask).encode())

def generate_voxel_grids(poses, pastPoints, pastLabels, priorPoints, priorLabels, config, anchor_pose, output_dir, outname):
    priorGrid = VoxelGrid()
    pastGrid = VoxelGrid()

    current = 0
    while current < len(poses):
        # Retrieve data for current index [S1, unofficial]
        # Data alignment assumed per source structure

        # Count labeled points [S1, unofficial]
        labelCount = 0
        pointCount = len(pastLabels)
        for label in pastLabels:
            if label > 0:
                labelCount += 1

        percentageLabeled = 100.0 * labelCount / pointCount

        # Skip if no labels [S1, unofficial]
        if percentageLabeled <= 0.0:
            current += 1
            continue

        # Update occlusions [S1, unofficial]
        priorGrid.updateOcclusions()
        pastGrid.updateOcclusions()

        # Mark invalid voxels [S1, unofficial]
        for i in range(len(pastPoints)):
            # Compute endpoint in anchor frame [S1, unofficial]
            # anchor_pose_inverse * pastPoints[i].pose
            transformed = mat4_multiply(mat4_inverse(anchor_pose), pastPoints[i].pose)
            endpoint = [transformed[0][3], transformed[1][3], transformed[2][3]]
            pastGrid.updateInvalid(endpoint)

        # Save grids [S1, unofficial]
        saveVoxelGrid(priorGrid, output_dir, outname, "input")
        saveVoxelGrid(pastGrid, output_dir, outname, "target")

        # Advance scan index [S1, unofficial]
        distance = 0.0
        count = 0
        while (count < config.stride_num or distance < config.stride_distance or count == 0) and current + 1 < len(poses):
            distance += pose_distance(poses[current], poses[current + 1])
            count += 1
            current += 1

        if current >= len(poses):
            break

def mat4_multiply(A, B):
    # Standard 4x4 matrix multiplication [G]
    return [[sum(A[i][k] * B[k][j] for k in range(4)) for j in range(4)] for i in range(4)]

def mat4_inverse(M):
    # Standard 4x4 matrix inverse [G]
    pass

def pose_distance(pose1, pose2):
    # Euclidean distance between translation components [G]
    t1 = [pose1[0][3], pose1[1][3], pose1[2][3]]
    t2 = [pose2[0][3], pose2[1][3], pose2[2][3]]
    return math.sqrt(sum((a - b) ** 2 for a, b in zip(t1, t2)))
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
Loads camera parameters and 3D point clouds from Structure-from-Motion (SfM) reconstructions, resizes source images to a target resolution, and adjusts coordinate systems to match rendering requirements. It serves as the initial data preparation step in the Voxel Grid Data Generation Pipeline, ensuring that geometric and photometric inputs are normalized for downstream voxelization. It differs from sibling methods by focusing specifically on the ingestion and normalization of external SfM data rather than grid construction or compression.

### [DCX Pipeline](3770-dcx-pipeline.md)
Converts point clouds or volumetric inputs into explicit surface triangle meshes by learning a discrete topology class and continuous vertex offset for each voxel grid cell, then extracting the mesh via a C++/CUDA extension. It distinguishes itself from direct surface reconstruction methods (like Poisson Reconstruction) by operating on a discrete voxel grid, allowing explicit control over topology via cube types and enabling supplementary sampling to fill geometric holes. It builds on [Voxel Grid Data Generation Pipeline](1208-voxel-grid-data-generation-pipeline.md).

### Direct Geometry Scan-Conversion
Scan-converts the geometry directly into the sampled array and renders the ensemble, allowing for true 3-D merge of visualizations [S2, unofficial].

### KITTI Dataset Preprocessing
Prepares KITTI dataset for training by parsing camera extrinsics/intrinsics, downsampling point clouds, adding background shell, and splitting into train/validation sets [S3, unofficial].
- **Definition**: Prepares KITTI dataset for training by parsing camera extrinsics/intrinsics, downsampling point clouds, adding background shell, and splitting into train/validation sets [S3, unofficial].
- **Steps**:
  1. Parse camera extrinsics from XML to get `T_pointcloud_camera`, `label`, `sensor_id`, `path` [S3, unofficial].
  2. Parse intrinsics to get `K`, `width`, `height` [S3, unofficial].
  3. Load point cloud from PLY [S3, unofficial].
  4. Downsample point cloud by 99% (`frac=0.01`) [S3, unofficial].
  5. Add 1000 random shell points around scene center/radius [S3, unofficial].
  6. Save to parquet [S3, unofficial].
  7. Split frames every 3 for train/val [S3, unofficial].
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
The core pipeline originates from sequential scan processing code that handles labeled point clouds and occlusion updates [S1, unofficial]. The concept of direct geometry scan-conversion for merging volume and geometric data originates from earlier work on volume rendering and visualization [S2, unofficial]. The KITTI dataset preprocessing steps are derived from a Taichi-based 3D Gaussian Splatting implementation [S3, unofficial].

## Sources
[S1] jbehley/voxelizer, https://github.com/jbehley/voxelizer, fetched 2026-10-27 (unofficial)
[S2] DTIC ADA193649: Rendering of Surfaces from Volume Data,, https://archive.org/details/DTIC_ADA193649, fetched 2026-10-26 (unofficial)
[S3] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
