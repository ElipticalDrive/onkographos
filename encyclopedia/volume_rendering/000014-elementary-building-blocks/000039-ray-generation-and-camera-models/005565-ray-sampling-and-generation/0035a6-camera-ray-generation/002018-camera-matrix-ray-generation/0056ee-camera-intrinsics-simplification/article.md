# Camera Intrinsics Simplification

Also called: FOV from Intrinsics, Field of View Calculation, Simplified Camera Intrinsics, Square Pixel Camera Model

## Overview
This method computes the horizontal and vertical field of view (FOV) angles from an intrinsic camera matrix [S1, S2, unofficial]. It solves the problem of deriving geometric FOV parameters required for ray generation when only the calibration matrix $K$ is available, bypassing the need for explicit FOV inputs [S2, unofficial]. It differs from [Pinhole Camera Ray Generation](5058-pinhole-camera-ray-generation.md) by providing a direct geometric conversion without full matrix decomposition, and from [Camera Matrix Ray Generation](2172-camera-matrix-ray-generation.md) by operating directly on the intrinsic matrix to extract FOV angles [S2, unofficial].

## How it works
**Inputs:**
- Intrinsic matrix $K$ ($3 \times 3$) [S1, S2, unofficial].
- Image width $W$ and height $H$ in pixels (implicit in $K$ for the square pixel model) [S3, unofficial].

**Outputs:**
- Horizontal FOV $\text{fov\_x}$ in radians [S2, unofficial].
- Vertical FOV $\text{fov\_y}$ in radians [S2, unofficial].

**Data Structures:**
- $K$: $3 \times 3$ matrix. The simplified square pixel camera model layout is:
  $$
  K = \begin{bmatrix}
  f_{\text{camera}} & 0 & W/2 \\
  0 & f_{\text{camera}} & H/2 \\
  0 & 0 & 1
  \end{bmatrix}
  $$
  where $f_{\text{camera}}$ is the focal length, $W$ is image width, and $H$ is image height [S3, unofficial].
- `intrinsics_inv`: $3 \times 3$ matrix, the inverse of $K$ [S1, S2, unofficial].

**Steps:**
1. Compute the inverse of the intrinsic matrix: $K^{-1} = K.\text{inverse}()$ [S1, S2, unofficial].
2. Define corner vectors in homogeneous coordinates: $\mathbf{v}_{\text{left}} = [0, 0.5, 1]^T$, $\mathbf{v}_{\text{right}} = [1, 0.5, 1]^T$, $\mathbf{v}_{\text{top}} = [0.5, 0, 1]^T$, $\mathbf{v}_{\text{bottom}} = [0.5, 1, 1]^T$ [S1, S2, unofficial].
3. Transform each corner vector to camera space using $K^{-1}$ and normalize the result: $\hat{\mathbf{v}} = \text{normalize}(K^{-1} \mathbf{v})$ [S1, S2, unofficial].
4. Compute horizontal FOV as the angle between the transformed left and right vectors: $\text{fov\_x} = \arccos(\hat{\mathbf{v}}_{\text{left}} \cdot \hat{\mathbf{v}}_{\text{right}})$ [S1, S2, unofficial].
5. Compute vertical FOV as the angle between the transformed top and bottom vectors: $\text{fov\_y} = \arccos(\hat{\mathbf{v}}_{\text{top}} \cdot \hat{\mathbf{v}}_{\text{bottom}})$ [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_fov_from_intrinsics(K: np.ndarray) -> tuple:
    """
    Computes horizontal and vertical FOV angles from an intrinsic matrix K.
    Returns (fov_x, fov_y) in radians.
    """
    # Step 1: Compute inverse intrinsic matrix
    K_inv = np.linalg.inv(K)
    
    # Step 2: Define corner vectors in homogeneous coordinates
    left = np.array([0.0, 0.5, 1.0])
    right = np.array([1.0, 0.5, 1.0])
    top = np.array([0.5, 0.0, 1.0])
    bottom = np.array([0.5, 1.0, 1.0])
    
    # Step 3: Transform to camera space and normalize
    def process_vector(v: np.ndarray) -> np.ndarray:
        vec = K_inv @ v
        return vec / np.linalg.norm(vec)
    
    left_cam = process_vector(left)
    right_cam = process_vector(right)
    top_cam = process_vector(top)
    bottom_cam = process_vector(bottom)
    
    # Step 4 & 5: Compute FOV using dot product and acos
    fov_x = math.acos(np.dot(left_cam, right_cam))
    fov_y = math.acos(np.dot(top_cam, bottom_cam))
    
    return fov_x, fov_y
```

**Parameters:**
- $f_{\text{camera}}$: Focal length of the camera [S3, unofficial].
- $W$: Width of the image in pixels [S3, unofficial].
- $H$: Height of the image in pixels [S3, unofficial].

**Edge Cases and Pitfalls:**
- Singular matrix: If $K$ is not invertible, the method fails; ensure $K$ has non-zero diagonal elements [S1, S2, unofficial].
- Precision: The `acos` function is sensitive to normalization errors; vectors must be strictly normalized before dot product computation [S1, S2, unofficial].
- Coordinate convention: The corner vector definitions assume a specific normalized image coordinate system; deviations require adjusting the vector definitions accordingly [S1, S2, unofficial].

## Subtopics

### [Optimized 3D to 2D Projection](15857-optimized-3d-to-2d-projection.md)
This method transforms 3D world-space vertices of geometric primitives (specifically cube faces) into 2D screen-space coordinates using a simplified perspective projection model. It solves the problem of determining which faces are visible on the screen by combining coordinate transformation, perspective division, and visibility culling (backface, near-plane, and screen-boundary checks). It differs from general-purpose graphics pipelines by operating on raw vertex arrays with explicit yaw/pitch rotations and a custom visibility mask calculation, rather than relying on a full view-projection matrix multiplication.

### Intrinsic Embedding via Spherical Harmonics
This variant generates an embedding of camera intrinsic parameters by converting ray directions to spherical harmonics coefficients [S2, unofficial]. It extracts batch, views, and image dimensions, calculates target dimensions after downsampling, and samples image grid coordinates [S2, unofficial]. It converts local ray directions to spherical harmonics coefficients based on a specified degree (0, 2, 4, or 8) [S2, unofficial]. The output shape depends on the `merge_hw` parameter: if True, shape is `[b, v, (h w), d]`; otherwise `[b, v, d, h, w]` [S2, unofficial].

### [Camera Intrinsics Simplification — FOV from Intrinsics, Field of View Calculation, Simplified Camera Intrinsics, Square Pixel Camera Model, get_fov): Computes the horizontal and vertical field of view (FOV) angles from intrinsic camera matrices.](18334-camera-intrinsics-simplification-fov-from-intrinsics-field-o.md)
This method computes the horizontal and vertical field of view (FOV) angles from intrinsic camera parameters, specifically converting between vertical and horizontal FOV based on image dimensions. It relies on the simplified square pixel camera model, assuming principal points are centered and pixel aspect ratios are unity. It sets itself apart from general intrinsic extraction by providing a direct geometric conversion without requiring full matrix decomposition, serving as a foundational step for ray generation in volume rendering.

## Operating characteristics
- **Time Complexity:** $O(1)$, involving a fixed number of matrix inversions, vector transformations, and trigonometric operations [S1, unofficial].
- **Memory Complexity:** $O(1)$ for the core FOV calculation, requiring temporary storage for four corner vectors and the inverse intrinsic matrix [S1, unofficial].
- **Performance:** Extremely fast, suitable for per-camera initialization in volume rendering pipelines [S2, unofficial].
- **Strengths:** Direct geometric conversion avoids full matrix decomposition; robust to intrinsic matrix variations when using the vector-based approach [S2, unofficial].
- **Limitations:** Assumes the intrinsic matrix accurately models the camera; corner vector convention is tied to a specific normalized coordinate system [S1, S2, unofficial].
- **When to use:** Deriving FOV angles from a known calibration matrix for ray generation [S2, unofficial].
- **When not to use:** When full camera pose decomposition or non-perspective models are required [S2, unofficial].

## Background
The method originates from implementations in 3D Gaussian Splatting dataset builders and novel view synthesis frameworks [S1, S2, unofficial]. The vector-based FOV calculation is implemented in `depthsplat` and `NoPoSplat` [S1, S2, unofficial]. The square pixel camera model definition is documented in NeRF implementation guides [S3, unofficial]. The spherical harmonics embedding variant is used in `NoPoSplat` for intrinsic parameter representation [S2, unofficial].

## Sources
[S1] cvg/depthsplat, https://github.com/cvg/depthsplat, fetched 2026-10-27 (unofficial)
[S2] cvg/NoPoSplat, https://github.com/cvg/NoPoSplat, fetched 2026-10-27 (unofficial)
[S3] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
