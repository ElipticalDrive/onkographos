# Scene Data Loading & Resizing
Also called: Pose Normalization and Loading, Trajectory Loading and Normalization

## Overview
This method loads camera parameters and 3D point clouds from Structure-from-Motion (SfM) reconstructions, resizes source images to a target resolution, and adjusts coordinate systems to match rendering requirements [S2, unofficial]. It serves as the initial data preparation step in the Voxel Grid Data Generation Pipeline, ensuring that geometric and photometric inputs are normalized for downstream voxelization [S2, unofficial]. It differs from sibling methods by focusing specifically on the ingestion and normalization of external SfM data rather than grid construction or compression [S2, unofficial].

## How it works
**Inputs:**
- `model_path`: Path to the directory containing COLMAP reconstruction files (typically `.bin`) [S2, unofficial].
- `images_path`: Path to the directory containing source images [S2, unofficial].
- `target_res`: Target pixel size for the image long edge (typical: 400) [S2, unofficial].
- `max_points`: Maximum number of 3D points to load to prevent hardware overflow (typical: 1000) [S2, unofficial].

**Outputs:**
- `cam_infos`: A list of `CameraInfo` objects containing resized intrinsics, extrinsics, and image paths [S2, unofficial].
- `points`: A subsampled set of 3D points (`xyz`) and normalized colors (`rgb`) [S2, unofficial].

**Data Structures:**
- `CameraInfo`:
  - `R`: Rotation matrix, shape $(3, 3)$, `torch.float` [S2, unofficial].
  - `T`: Translation vector, shape $(3)$, `torch.float` [S2, unofficial].
  - `fx`, `fy`: Focal lengths, `float` [S2, unofficial].
  - `cx`, `cy`: Principal point offsets, `float` [S2, unofficial].
  - `width`, `height`: Resized image dimensions, `int` [S2, unofficial].
  - `image_path`: Absolute path to the image file, `str` [S2, unofficial].
  - `image_name`: Original filename, `str` [S2, unofficial].

**Steps:**
1. **Load Data**: Read camera extrinsics, image intrinsics, and 3D points using `read_model(path=model_path, ext=".bin")` [S2, unofficial].
2. **Iterate Images**: For each image in the reconstruction:
   a. Retrieve the corresponding camera model and image metadata [S2, unofficial].
   b. Calculate the scaling factor: $s = \frac{\text{target\_res}}{\max(\text{orig\_w}, \text{orig\_h})}$ [S2, unofficial].
   c. Compute new dimensions: $\text{new\_w} = \lfloor \text{orig\_w} \times s \rfloor$, $\text{new\_h} = \lfloor \text{orig\_h} \times s \rfloor$ [S2, unofficial].
   d. Convert the quaternion $qvec$ to a rotation matrix $R$ using `qvec2rotmat` [S2, unofficial].
   e. Adjust coordinates: Negate the Y and Z axes of $R$ and $T$ to convert from COLMAP's coordinate system to the rendering system:
      $$ R_{new}[1:3, :] = -R_{old}[1:3, :] $$
      $$ T_{new}[1:3] = -T_{old}[1:3] $$
      [S2, unofficial].
   f. Scale intrinsics by $s$:
      - If model is `PINHOLE`: $f_x, f_y, c_x, c_y$ are scaled individually [S2, unofficial].
      - If model is `SIMPLE_PINHOLE`: $f_x = f_y = \text{params}[0] \times s$, $c_x, c_y = \text{params}[1] \times s, \text{params}[2] \times s$ [S2, unofficial].
      - Fallback: Apply similar scaling logic if other models are encountered [S2, unofficial].
   g. Construct `CameraInfo` with resized dimensions and scaled intrinsics [S2, unofficial].
3. **Process Points**:
   a. Extract $xyz$ and $rgb$ from `points3D` [S2, unofficial].
   b. Normalize RGB values: $\text{rgb}_{norm} = \frac{\text{rgb}}{255.0}$ [S2, unofficial].
   c. If $\text{len}(xyz) > \text{max\_points}$, randomly subsample the points to `max_points` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import os
import torch
import numpy as np
from src.utils.read_write_model import read_model, qvec2rotmat

class CameraInfo:
    def __init__(self, R, T, fx, fy, cx, cy, width, height, image_path, image_name):
        self.R = R
        self.T = T
        self.fx = fx
        self.fy = fy
        self.cx = cx
        self.cy = cy
        self.width = width
        self.height = height
        self.image_path = image_path
        self.image_name = image_name

def fetch_scene_data(model_path, images_path, target_res=400, max_points=1000):
    # 1. Load COLMAP data
    cameras_extrinsic, images_intrinsic, points3D = read_model(path=model_path, ext=".bin")
    cam_infos = []

    # 2. Process each image
    for key in images_intrinsic:
        colmap_image = images_intrinsic[key]
        colmap_cam = cameras_extrinsic[colmap_image.camera_id]

        # Scale calculation
        orig_w = colmap_cam.width
        orig_h = colmap_cam.height
        scale = target_res / max(orig_w, orig_h)
        new_w = int(orig_w * scale)
        new_h = int(orig_h * scale)

        # Coordinate transformation
        R = np.array(torch.from_numpy(qvec2rotmat(colmap_image.qvec)).float())
        T = np.array(torch.from_numpy(colmap_image.tvec).float())
        R[1:3, :] *= -1
        T[1:3] *= -1
        R = torch.from_numpy(R).float()
        T = torch.from_numpy(T).float()

        # Intrinsic scaling
        params = colmap_cam.params
        if colmap_cam.model == "PINHOLE":
            fx, fy, cx, cy = params[0] * scale, params[1] * scale, params[2] * scale, params[3] * scale
        elif colmap_cam.model == "SIMPLE_PINHOLE":
            fx = fy = params[0] * scale
            cx, cy = params[1] * scale, params[2] * scale
        else:
            # Fallback for other models
            fx = fy = params[0] * scale
            cx, cy = params[1] * scale, params[2] * scale

        img_path = os.path.join(images_path, colmap_image.name)
        cam_infos.append(CameraInfo(R=R, T=T, fx=fx, fy=fy, cx=cx, cy=cy,
                                    width=new_w, height=new_h,
                                    image_path=img_path, image_name=colmap_image.name))

    # 3. Process points
    xyz = np.array([p.xyz for p in points3D.values()])
    rgb = np.array([p.rgb for p in points3D.values()]) / 255.0

    # Subsample if necessary
    if len(xyz) > max_points:
        indices = np.random.choice(len(xyz), max_points, replace=False)
        xyz = xyz[indices]
        rgb = rgb[indices]

    return cam_infos, xyz, rgb
```

**Edge Cases:**
- **Coordinate System Mismatch**: COLMAP uses a right-handed coordinate system with Z-forward, while many renderers use Y-up or Z-up. Negating Y and Z axes handles this [S2, unofficial].
- **Intrinsic Model Variance**: Different camera models (PINHOLE, SIMPLE_RADIAL, etc.) have different parameter counts. The method must handle these variations explicitly [S2, unofficial].
- **Memory Limits**: Large point clouds can exceed GPU memory. Subsampling is mandatory when point count exceeds `max_points` [S2, unofficial].

## Subtopics

### [COLMAP Binary Model Parsing](3455-colmap-binary-model-parsing.md)
Parses COLMAP reconstruction files stored in binary format (`.bin`) into structured Python objects (`Camera`, `Image`, `Point3D`). It provides a memory-efficient alternative to text parsing for large-scale SfM/MVS datasets, serving as the primary data loading step in the [Scene Data Loading & Resizing](3420-scene-data-loading-resizing.md) pipeline.

### [OpenSfM Reconstruction Loader & Coordinate Transformation](6855-opensfm-reconstruction-loader-coordinate-transformation.md)
This method loads camera poses, intrinsics, and 3D point clouds from OpenSfM's `reconstruction.json` and `image_list.txt`, converting them into a format suitable for volume rendering or neural rendering pipelines. It specifically handles the conversion from OpenSfM's native coordinate system to an OpenGL-compatible system, normalizes focal lengths based on image dimensions, and applies auto-scaling to center and normalize the scene scale. It differs from generic COLMAP loaders by handling OpenSfM's specific JSON structure and coordinate conventions.

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of images and points, $O(N_{img} + N_{pts})$ [S2, unofficial].
- **Memory Complexity**: Proportional to the number of points and images loaded. Subsampling limits peak memory usage to `max_points` [S2, unofficial].
- **Strengths**: Standardizes heterogeneous SfM outputs into a uniform format suitable for neural rendering; handles coordinate system conversions automatically [S2, unofficial].
- **Limitations**: Tightly coupled to COLMAP's binary format structure; requires manual adjustment for other SfM tools [S2, unofficial].
- **When to Use**: When initializing a volume rendering pipeline with COLMAP-generated data [S2, unofficial].

## Background
This method originates from the `mini-splat` project, which implements efficient 3D Gaussian Splatting pipelines [S2, unofficial]. It addresses the need for robust data ingestion from SfM tools, which often output data in formats not directly compatible with PyTorch-based renderers [S2, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] metin-yat/mini-splat, https://github.com/metin-yat/mini-splat, fetched 2026-10-28 (unofficial)
