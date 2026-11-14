# Bounding Sphere Estimation

Also called: Camera Pose Normalization, Camera Radius Estimation, Scene Radius Estimation

## Overview
Estimates the center and radius of a bounding sphere for a scene based on a set of camera poses [S1, unofficial][S2, unofficial][S3, unofficial]. It differs from [Focus Point Calculation](7638-focus-point-calculation.md) by explicitly deriving a scalar radius representing the scene's extent, which is critical for normalizing depth values and initializing volume rendering bounds [S2, unofficial].

## How it works
**Inputs:** A list of camera objects containing world-to-view transformation matrices (`world_view_transform`) [S2, unofficial].
**Outputs:** A scene center vector `center` and a scalar radius `radius` [S3, unofficial].

**Data Structures:**
- `c2ws`: Array of shape $(N, 3, 4)$ containing camera-to-world transformation matrices [S3, unofficial].
- `poses`: Array of shape $(N, 3, 4)$ with coordinate system adjusted [S1, unofficial][S3, unofficial].
- `center`: Vector of shape $(3,)$ [S3, unofficial].
- `radius`: Scalar float [S2, unofficial][S3, unofficial].

**Steps:**
1. **Invert Transforms:** Convert world-to-view matrices to camera-to-world (c2w) matrices by inverting the transpose of the world-view transform [S1, unofficial][S2, unofficial][S3, unofficial].
   $$ \text{c2w}_i = (\text{world\_view\_transform}_i^T)^{-1} $$
2. **Coordinate Adjustment:** Apply a coordinate system flip to the c2w matrices to align with the expected rendering convention (flipping Y and Z axes) [S1, unofficial][S3, unofficial].
   $$ \text{poses}_i = \text{c2w}_i[:, :3, :] \times \text{diag}(1, -1, -1, 1) $$
3. **Compute Center:** Calculate the scene center (focus point) using the adjusted poses [S1, unofficial][S3, unofficial]. This delegates to the [Focus Point Calculation](7638-focus-point-calculation.md) method [S1, unofficial][S3, unofficial].
4. **Compute Radius:** Calculate the radius as the minimum Euclidean distance from the computed center to any camera position (the translation component of the c2w matrix) [S3, unofficial]. One source describes it as the maximum distance [S2, unofficial].
   $$ \text{radius} = \min_i \| \text{c2w}_i[:, 3] - \text{center} \|_2 $$
5. **Store:** Store the center as a CUDA tensor if required by the downstream pipeline [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch

def bounding_sphere_estimation(viewpoint_stack, focus_point_fn):
    c2ws = []
    for cam in viewpoint_stack:
        w2v = np.asarray(cam.world_view_transform.T.cpu().numpy())
        c2w = np.linalg.inv(w2v)
        adjustment = np.diag([1, -1, -1, 1])
        poses_i = c2w[:3, :] @ adjustment
        c2ws.append(poses_i)
    c2ws = np.array(c2ws)
    center = focus_point_fn(c2ws)
    camera_positions = c2ws[:, :, 3]
    distances = np.linalg.norm(camera_positions - center, axis=-1)
    radius = np.min(distances)
    center_tensor = torch.from_numpy(center).float().cuda()
    return center, radius, center_tensor
```

**Parameters:**
- `focus_point_fn`: Implementation of [Focus Point Calculation](7638-focus-point-calculation.md) [S1, unofficial][S3, unofficial].

**Edge Cases:**
- **Degenerate Poses:** If all cameras are at the same location, the radius will be 0 [G].
- **Precision:** Inverting matrices can introduce numerical instability if the world-view transform is ill-conditioned [G].
- **Coordinate Systems:** The specific flip `diag(1, -1, -1, 1)` is critical [S1, unofficial][S3, unofficial]. Using the wrong convention will result in an incorrect center and radius, leading to rendering artifacts or incorrect depth normalization [G].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of cameras, dominated by matrix inversion and norm calculations [G].
- **Memory Complexity:** $O(N)$ to store the c2w matrices [G].
- **Strengths:** Simple, fast, and provides a robust initial bound for scene normalization [G].
- **Limitations:** Relies on the accuracy of the [Focus Point Calculation](7638-focus-point-calculation.md) [S1, unofficial][S3, unofficial]. If the focus point is biased, the radius (being the *minimum* distance) may underestimate the scene extent if the center is far from the true centroid [G].

## Background
This method is used in the VGGGT and 2D Gaussian Splatting pipelines to initialize scene parameters for training [S1, unofficial][S3, unofficial]. It ensures that the camera frustums and depth values are normalized relative to the scene's scale [S2, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] minghanqin/LangSplat, https://github.com/minghanqin/LangSplat, fetched 2026-10-27 (unofficial)
[S3] Zhaoyibinn/VGVGGT, https://github.com/Zhaoyibinn/VGVGGT, fetched 2026-10-27 (unofficial)
