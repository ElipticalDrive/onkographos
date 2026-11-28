# Perspective Projection of Points

Also called: Camera Class, Camera Transformation, Perspective Division, Pinhole Camera Model, Point Projection, Point Projection to 2D, Point to Camera Transformation with Perspective Division

## Overview
Perspective projection maps 3D world points to 2D image coordinates using a pinhole camera model, where a line joining the 3D point to the center of projection intersects the image plane [S1, unofficial]. It solves the problem of determining the 2D pixel location and visibility of 3D geometry for a specific camera view [S1, unofficial].

## How it works
**Inputs:**
- 3D points $M$ with homogeneous coordinates $\begin{bmatrix} X & Y & Z & 1 \end{bmatrix}^T$ in world space [S1, unofficial].
- Camera calibration matrix $K$ (intrinsic parameters) [S1, unofficial].
- Extrinsic parameters: rotation matrix $R$ and translation vector $t$ defining the transformation from world to camera coordinates [S1, unofficial].

**Outputs:**
- 2D image coordinates $\begin{bmatrix} x & y & 1 \end{bmatrix}^T$ in homogeneous pixel space [S1, unofficial].

**Data Structures:**
- $K$: $3 \times 3$ matrix.
- $[R|t]$: $3 \times 4$ matrix, where $R$ is $3 \times 3$ and $t$ is $3 \times 1$.
- $P$: $3 \times 4$ projective camera matrix, computed as $P = K[R|t]$ [S1, unofficial].

**Parameters:**
- $f_x, f_y$: Focal lengths in pixels. Calculated as $f_x = f/k_x$ and $f_y = f/k_y$, where $f$ is focal length in mm and $k_x, k_y$ are pixels/mm [S1, unofficial].
- $s$: Skew parameter. Typically 0 [S1, unofficial].
- $\alpha$: Aspect ratio. Typically 1 [S1, unofficial].
- $(u_0, v_0)$: Principal point coordinates [S1, unofficial].

**Steps:**
1. Construct the extrinsic matrix $[R|t]$ representing the world-to-camera transformation [S1, unofficial].
2. Construct the intrinsic matrix $K$:
   $$K = \begin{pmatrix} f_x & s & u_0 \\ 0 & \alpha f_y & v_0 \\ 0 & 0 & 1 \end{pmatrix}$$ [S1, unofficial].
3. Compute the projective camera matrix $P = K[R|t]$ [S1, unofficial].
4. For each 3D point $M = \begin{bmatrix} X & Y & Z & 1 \end{bmatrix}^T$, compute the homogeneous image coordinates:
   $$\begin{bmatrix} \lambda x \\ \lambda y \\ \lambda \end{bmatrix} = P \begin{bmatrix} X \\ Y \\ Z \\ 1 \end{bmatrix}$$ [S1, unofficial].
5. Perform perspective division by dividing the $x$ and $y$ components by the third component $\lambda$ (the camera-space $z$ coordinate) to obtain normalized image coordinates [S1, S2, unofficial].
6. Handle edge cases: Points with $\lambda \le 0$ are behind the camera and should be discarded or flagged [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def perspective_projection(pts, calibs):
    """
    Projects 3D points to 2D using perspective projection.
    Args:
        pts: Tensor of shape [B, N, 3] containing world coordinates.
        calibs: Tensor of shape [B, 3, 4] containing the combined intrinsic/extrinsic matrix P.
    Returns:
        pts_2d: Tensor of shape [B, N, 2] containing pixel coordinates.
    """
    # 1. Permute pts from [B, N, 3] to [B, 3, N]
    pts_perm = pts.permute(0, 2, 1)
    
    # 2. Multiply by rotation part of calibs [B, 3, 3]
    pts_rot = torch.matmul(calibs[:, :3, :3], pts_perm)
    
    # 3. Add translation part of calibs [B, 3, 1]
    pts_cam = pts_rot + calibs[:, :3, 3:4]
    
    # 4. Divide x and y coordinates by z coordinate
    z = pts_cam[:, 2:3, :]
    pts_2d = pts_cam[:, :2, :] / z
    
    # 5. Permute back to [B, N, 2]
    return pts_2d.permute(0, 2, 1)
```

## Subtopics

### [Stereo Projection](5789-stereo-projection.md)
Stereo projection applies a perspective projection twice with adjustments to simulate the viewpoint for an observer's left and right eye, creating a stereo effect. It generates two correlated view frustums rather than one. Choose when depth perception from two correlated views is required.

### [MCOP Projection Geometry](6527-mcop-projection-geometry.md)
MCOP Projection Geometry is a projection method for light field displays that uses two distinct viewpoints to map 3D world points to projector screen coordinates: an emitter $E$ for horizontal coordinates and a virtual viewer $V$ for vertical coordinates. It solves the problem of perspective-correct rendering on holographic screens that are sharply transmissive horizontally but scatter widely vertically, ensuring view separation is maintained horizontally while allowing vertical viewing flexibility. It differs from standard perspective projection by decoupling the horizontal and vertical projection centers, and from stereo projection by supporting a continuous light field rather than just two views. Choose for holographic screens requiring horizontal view separation and vertical flexibility.

### [Eye Projection](12794-eye-projection.md)
Eye projection is a viewpoint-dependent technique for visualizing 3D vector fields by projecting 3D vectors onto a surface in a direction parallel to the viewing direction. It produces the most intuitive representation of a constant direction vector field for a single image, making the uniform direction of the field readily apparent. Unlike viewpoint-independent techniques, it introduces visual discontinuities (jumpiness or flashing) when the viewpoint changes in animation. Choose for intuitive visualization of vector fields in single images, accepting viewpoint-dependent discontinuities.

## Operating characteristics
- **Strengths:** Standard model for pinhole cameras; computationally efficient; widely supported in computer vision libraries [S1, unofficial].
- **Limitations:** Fails for points behind the camera ($\lambda \le 0$); assumes ideal pinhole geometry without lens distortion [S1, unofficial].
- **When to use:** When mapping 3D world points to 2D pixel coordinates for a calibrated camera [S1, unofficial].
- **When not to use:** For orthographic views or when lens distortion correction is required before projection [S1, unofficial].

## Background
The pinhole camera model is one of the most widely used models in computer vision for imaging processes [S1, unofficial]. It was formalized in the context of epipolar and projective geometry, with foundational reviews by Triggs [S1, unofficial]. The decomposition of the camera matrix into intrinsic and extrinsic parameters is attributed to Hartley and Zisserman [S1, unofficial].

## Sources
[S1] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
[S2] YaourtB/GPS_plus, https://github.com/YaourtB/GPS_plus, fetched 2026-10-27 (unofficial)
[S3] Total Internal Reflection, https://avishek.net/2022/12/05/pytorch-guide-plenoxels-nerf-part-2.html, fetched 2026-10-29 (unofficial)
