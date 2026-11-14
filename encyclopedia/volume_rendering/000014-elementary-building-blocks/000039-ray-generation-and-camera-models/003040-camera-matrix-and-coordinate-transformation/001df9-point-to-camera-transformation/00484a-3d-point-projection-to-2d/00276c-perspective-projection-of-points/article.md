# Perspective Projection of Points

Also called: Camera Class, Camera Transformation, Perspective Division, Pinhole Camera Model, Point Projection, Point Projection to 2D, Point to Camera Transformation with Perspective Division

## Overview
Perspective projection maps 3D world points to 2D image coordinates using a pinhole camera model, where a line joining the 3D point to the center of projection intersects the image plane [S1, unofficial]. It solves the problem of determining the 2D pixel location and visibility of 3D geometry for a specific camera view [S1, unofficial]. It differs from orthographic projection by incorporating depth-dependent scaling (perspective division) and from stereo projection by generating a single view frustum rather than correlated pairs [S1, unofficial].

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
- Points array: Typically batched as $[B, N, 3]$ where $B$ is batch size and $N$ is number of points [S2, unofficial].

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
5. Perform perspective division by dividing the $x$ and $y$ components by $\lambda$ (the $z$ component in camera space) to obtain normalized image coordinates:
   $$x = \frac{\lambda x}{\lambda}, \quad y = \frac{\lambda y}{\lambda}$$ [S1, S2, unofficial].
6. Handle edge cases: Points with $\lambda \le 0$ are behind the camera and should be discarded or flagged [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def perspective_projection(points_3d, K, R, t):
    """
    Projects 3D points to 2D using perspective projection.
    
    Args:
        points_3d: Tensor of shape (N, 3) or (B, N, 3) containing world coordinates.
        K: Intrinsic matrix (3, 3).
        R: Rotation matrix (3, 3).
        t: Translation vector (3, 1).
        
    Returns:
        points_2d: Tensor of shape (N, 2) or (B, N, 2) containing pixel coordinates.
    """
    # Add homogeneous coordinate (1) to 3D points
    # If points_3d is (B, N, 3), expand to (B, N, 4)
    if points_3d.dim() == 2:
        points_hom = torch.cat([points_3d, torch.ones(points_3d.shape[0], 1)], dim=1) # (N, 4)
    else:
        points_hom = torch.cat([points_3d, torch.ones(points_3d.shape[:-1] + (1,))], dim=-1) # (B, N, 4)
        
    # Construct extrinsic matrix [R|t] of shape (3, 4)
    # t should be column vector (3, 1)
    extrinsic = torch.cat([R, t], dim=1) # (3, 4)
    
    # Compute projective matrix P = K @ [R|t]
    P = K @ extrinsic # (3, 4)
    
    # Project points: (B, N, 4) @ (4, 3)^T -> (B, N, 3)
    # Note: torch.matmul handles broadcasting for batch dimensions
    projected_hom = torch.matmul(points_hom, P.t()) # (B, N, 3)
    
    # Extract lambda (z-coordinate in camera space)
    lambda_z = projected_hom[..., 2] # (B, N)
    
    # Perspective division
    # Avoid division by zero
    epsilon = 1e-8
    valid = lambda_z > epsilon
    
    x = projected_hom[..., 0] / (lambda_z + epsilon)
    y = projected_hom[..., 1] / (lambda_z + epsilon)
    
    points_2d = torch.stack([x, y], dim=-1) # (B, N, 2)
    
    # Optionally mask invalid points
    # points_2d[~valid] = -1 # Or handle as needed
    
    return points_2d
```

## Subtopics

### [Stereo Projection](5789-stereo-projection.md)
Applies perspective projection twice with adjustments for left and right eyes to create a stereo effect. Choose when depth perception from two correlated views is required.

### [MCOP Projection Geometry](6527-mcop-projection-geometry.md)
Decouples horizontal and vertical projection centers for light field displays. Choose for holographic screens requiring horizontal view separation and vertical flexibility.

### [Eye Projection](12794-eye-projection.md)
Projects 3D vectors parallel to the viewing direction. Choose for intuitive visualization of vector fields in single images, accepting viewpoint-dependent discontinuities.

## Operating characteristics
- **Time Complexity:** $O(N)$ for $N$ points, dominated by matrix-vector multiplications and division [S1, S2, unofficial].
- **Memory Complexity:** $O(N)$ to store input and output coordinates [S2, unofficial].
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
