# OpenSfM Reconstruction Loader & Coordinate Transformation

Also called: OpenSfM Data Reading, OpenSfM Pose and Point Cloud Loading

## Overview
This method loads camera poses, intrinsics, and 3D point clouds from OpenSfM's `reconstruction.json` and `image_list.txt` [S1, unofficial]. It converts the native OpenCV-like coordinate system to an OpenGL-compatible system, normalizes focal lengths using image dimensions, and applies auto-scaling and centering to the scene [S1, unofficial]. It differs from generic COLMAP loaders by parsing OpenSfM's specific JSON structure and applying these coordinate and scale transformations [S1, unofficial].

## How it works
**Inputs:**
- `projectRoot`: Path to the OpenSfM project directory [S1, unofficial].
- `reconstruction.json`: OpenSfM's reconstruction file containing camera poses, intrinsics, and 3D points [S1, unofficial].
- `image_list.txt`: Maps image filenames to absolute paths [S1, unofficial].
- `projectionType`: String parameter, must be `"perspective"` or `"brown"` [S1, unofficial].

**Outputs:**
- `InputData` struct containing:
  - `cameras`: List of `Camera` objects with intrinsics, distortion coefficients, and poses [S1, unofficial].
  - `points`: 3D point cloud with XYZ coordinates and RGB values [S1, unofficial].
  - `translation`: 3D vector representing the scene center offset [S1, unofficial].
  - `scale`: Float representing the normalization scale factor [S1, unofficial].

**Data Structures:**
- `Camera`:
  - `fx, fy`: Normalized focal lengths [S1, unofficial].
  - `cx, cy`: Normalized principal point [S1, unofficial].
  - `k1, k2, k3, p1, p2`: Distortion coefficients loaded from JSON [S1, unofficial].
  - `R`: 3x3 rotation matrix [S1, unofficial].
  - `T`: 3x1 translation vector [S1, unofficial].
  - `imagePath`: String path to the image [S1, unofficial].
- `Point`:
  - `xyz`: 3D position [S1, unofficial].
  - `rgb`: 3-channel uint8 color [S1, unofficial].

**Steps:**

1. **Validation and Path Resolution:**
   - Verify `reconstruction.json` exists and contains at least one reconstruction [S1, unofficial].
   - Read `image_list.txt` to map relative filenames to absolute paths [S1, unofficial].

2. **Camera Pose Extraction:**
   - For each shot, extract the Rodrigues rotation vector and translation vector [S1, unofficial].
   - Convert the Rodrigues rotation vector to a 3x3 rotation matrix $R$ [S1, unofficial].
   - Construct the 4x4 world-to-camera matrix $W2C$ with $R$ in the top-left 3x3 and translation in the right column [S1, unofficial].
   - Invert $W2C$ to obtain the camera-to-world pose $C2W = W2C^{-1}$ [S1, unofficial].

3. **Coordinate System Conversion:**
   - Convert the OpenCV coordinate system to OpenGL by negating the Y and Z axes of the camera-to-world pose [S1, unofficial].
   - Specifically, multiply the elements at row indices 0–2 and column indices 1–2 of the pose matrix by $-1.0$ [S1, unofficial].

4. **Auto-Scaling and Centering:**
   - Apply `autoScaleAndCenterPoses` to normalize the scene scale and center it at the origin [S1, unofficial].
   - Store the resulting translation vector and scale factor [S1, unofficial].
   - Apply the inverse transformation to all camera positions and 3D points using $P_{normalized} = (P_{original} - \text{translation}) \cdot \text{scale}$ [S1, unofficial].

5. **Intrinsic Normalization:**
   - For each camera, compute $\text{normalizer} = \max(\text{width}, \text{height})$ [S1, unofficial].
   - Normalize focal lengths and principal points by dividing by the normalizer [S1, unofficial].

6. **Point Cloud Processing:**
   - Extract XYZ and RGB from the reconstruction [S1, unofficial].
   - Apply the inverse scale and translation to the points: $\text{xyz}_{final} = (\text{xyz}_{raw} - \text{translation}) \cdot \text{scale}$ [S1, unofficial].
   - Store RGB values as a uint8 tensor [S1, unofficial].
   - If `projectionType` is not `"perspective"` or `"brown"`, throw a `runtime_error` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import json
import numpy as np

def rodrigues_to_rotation(rvec):
    """Convert Rodrigues vector to 3x3 rotation matrix [G]."""
    theta = np.linalg.norm(rvec)
    if theta < 1e-10:
        return np.eye(3)
    r = rvec / theta
    r_x = np.array([[0, -r[2], r[1]],
                    [r[2], 0, -r[0]],
                    [-r[1], r[0], 0]])
    return np.eye(3) + np.sin(theta) * r_x + (1 - np.cos(theta)) * np.dot(r_x, r_x)

def load_opensfm(project_root):
    # 1. Load and validate reconstruction
    with open(f"{project_root}/reconstruction.json") as f:
        recon = json.load(f)
    if not recon.get("reconstructions"):
        raise ValueError("No reconstructions found")
    recon_data = recon["reconstructions"][0]

    cameras = []
    points = []
    all_positions = []

    # 2. Process Cameras
    for shot_id, shot in recon_data["shots"].items():
        cam_data = recon_data["cameras"][shot["camera"]]
        fx = cam_data["focal"]
        fy = cam_data["focal"]
        cx = cam_data["cx"]
        cy = cam_data["cy"]
        width = cam_data["width"]
        height = cam_data["height"]

        k1 = cam_data.get("k1", 0.0)
        k2 = cam_data.get("k2", 0.0)
        k3 = cam_data.get("k3", 0.0)
        p1 = cam_data.get("p1", 0.0)
        p2 = cam_data.get("p2", 0.0)

        rvec = np.array(shot["rotation"])
        tvec = np.array(shot["translation"])

        R = rodrigues_to_rotation(rvec)
        W2C = np.eye(4)
        W2C[:3, :3] = R
        W2C[:3, 3] = tvec
        C2W = np.linalg.inv(W2C)

        # Coordinate conversion: negate Y and Z axes of the pose matrix
        # Matches evidence: index_put_({Slice(0, 3), Slice(1,3)}, ... * -1.0f)
        C2W[:3, 1] *= -1.0
        C2W[:3, 2] *= -1.0

        all_positions.append(C2W[:3, 3].copy())

        cameras.append({
            "id": shot_id, "R_raw": C2W[:3, :3].copy(), "T_raw": C2W[:3, 3].copy(),
            "fx": fx, "fy": fy, "cx": cx, "cy": cy,
            "width": width, "height": height,
            "k1": k1, "k2": k2, "k3": k3, "p1": p1, "p2": p2,
            "image_path": shot.get("image", "")
        })

    # 3. Process Points
    for pid, point in recon_data["points"].items():
        xyz = np.array(point["xyz"])
        rgb = np.array(point["color"])
        points.append({"xyz": xyz, "rgb": rgb})
        all_positions.append(xyz)

    # 4. Auto-Scaling and Centering
    all_positions = np.array(all_positions)
    translation = np.mean(all_positions, axis=0)
    scale = 1.0 / np.max(np.linalg.norm(all_positions - translation, axis=1))

    # 5. Normalize and Store
    final_cameras = []
    for cam in cameras:
        # Transform pose: only translation is scaled and shifted, rotation remains unitary
        T_final = (cam["T_raw"] - translation) * scale
        R_final = cam["R_raw"]

        normalizer = max(cam["width"], cam["height"])
        final_cameras.append({
            "R": R_final, "T": T_final,
            "fx": cam["fx"] / normalizer, "fy": cam["fy"] / normalizer,
            "cx": cam["cx"] / normalizer, "cy": cam["cy"] / normalizer,
            "k1": cam["k1"], "k2": cam["k2"], "k3": cam["k3"],
            "p1": cam["p1"], "p2": cam["p2"],
            "image_path": cam["image_path"]
        })

    final_points = []
    for pt in points:
        xyz_final = (pt["xyz"] - translation) * scale
        final_points.append({"xyz": xyz_final, "rgb": pt["rgb"]})

    return {"cameras": final_cameras, "points": final_points, "translation": translation, "scale": scale}
```

**Edge Cases and Pitfalls:**
- **Projection Type:** The loader expects `"perspective"` or `"brown"`; other types throw a `runtime_error` [S1, unofficial].
- **Scale Distortion:** Auto-scaling changes the absolute metric scale of the scene to fit numerical stability requirements [S1, unofficial].
- **Coordinate Mismatch:** Failing to negate the Y and Z axes of the camera-to-world pose results in inverted camera orientations [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of cameras and points $O(N_c + N_p)$ [S1, unofficial].
- **Memory Complexity:** Linear with respect to the number of cameras and points [S1, unofficial].
- **Strengths:** Automatically handles OpenSfM's JSON structure, coordinate conversion, and scene normalization [S1, unofficial].
- **Limitations:** Requires `reconstruction.json` to be present; auto-scaling changes the absolute metric scale of the scene [S1, unofficial].
- **When to use:** When working with OpenSfM outputs and needing OpenGL-compatible poses [S1, unofficial].
- **When not to use:** When using COLMAP or NerfStudio formats directly, as sibling loaders handle those natively [S1, unofficial].

## Background
OpenSfM is an open-source Structure-from-Motion library [S1, unofficial]. The loader is part of the OpenSplat pipeline, which supports multiple SfM backends [S1, unofficial]. The coordinate transformation is necessary because OpenSfM uses a right-handed coordinate system with Z-forward and Y-down, while many rendering engines use Z-back and Y-up [S1, unofficial].

## Sources
[S1] OpenSplat – 3D gaussian splatting in C++ with CPU and GPU support, https://github.com/pierotofy/OpenSplat, fetched 2026-10-26 (unofficial)
