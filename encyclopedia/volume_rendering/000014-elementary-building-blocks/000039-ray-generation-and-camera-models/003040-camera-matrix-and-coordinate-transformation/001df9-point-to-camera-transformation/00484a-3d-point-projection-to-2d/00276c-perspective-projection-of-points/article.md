# Perspective Projection of Points

Also called: Camera Class, Camera Transformation, Pinhole Camera Model, Point Projection, Point Projection to 2D

## Overview
Perspective projection maps 3D world points to 2D image coordinates using a pinhole camera model, where a line joining the 3D point to the center of projection intersects the image plane [S1, unofficial]. It differs from orthographic projection by accounting for depth-dependent scaling (perspective division) and from general projective transformations by explicitly decomposing the camera matrix into intrinsic calibration and extrinsic pose parameters [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D point in world coordinates $\mathbf{M} = [X, Y, Z, 1]^T$ [S1, unofficial].
- Input: A 3x4 projective camera matrix $P$ [S1, unofficial].
- Output: A 2D point in homogeneous image coordinates $\mathbf{m} = [x, y, 1]^T$ [S1, unofficial].

**Data Structures:**
- **Camera Matrix ($P$):** A 3x4 matrix containing intrinsic and extrinsic parameters [S1, unofficial].
- **Intrinsic Matrix ($K$):** A 3x3 upper-triangular matrix [S1, unofficial].
- **Extrinsic Matrix ($[R|t]$):** A 3x4 matrix composed of a 3x3 rotation matrix $R$ and a 3x1 translation vector $t$ [S1, unofficial].

**Formulas:**
The projection is defined by the equation [S1, unofficial]:
$$ \lambda \begin{bmatrix} x \\ y \\ 1 \end{bmatrix} = P \begin{bmatrix} X \\ Y \\ Z \\ 1 \end{bmatrix} $$
where $\lambda$ is an arbitrary scale factor [S1, unofficial].

The camera matrix $P$ is decomposed as [S1, unofficial]:
$$ P = K[R|t] $$
where $[R|t]$ represents the rotation and translation from the world coordinate system to the camera coordinate system (extrinsic parameters) [S1, unofficial], and $K$ is the calibration matrix (intrinsic parameters) [S1, unofficial]:
$$ K = \begin{pmatrix} f_x & s & u_0 \\ 0 & \alpha f_y & v_0 \\ 0 & 0 & 1 \end{pmatrix} $$
where:
- $f_x, f_y$ are the focal lengths in pixels [S1, unofficial].
- $s$ is the skew parameter [S1, unofficial].
- $\alpha$ is the aspect ratio [S1, unofficial].
- $(u_0, v_0)$ are the principal point coordinates [S1, unofficial].

The focal lengths in pixels are calculated from the physical focal length $f$ (in mm) and pixel density $k_x, k_y$ (pixels/mm) as [S1, unofficial]:
$$ f_x = f/k_x, \quad f_y = f/k_y $$

**Steps:**
1. Construct the extrinsic matrix $[R|t]$ from the camera's rotation $R$ and translation $t$ relative to the world frame [S1, unofficial].
2. Construct the intrinsic matrix $K$ using focal lengths $f_x, f_y$, skew $s$, aspect ratio $\alpha$, and principal point $(u_0, v_0)$ [S1, unofficial].
3. Compute the projective camera matrix $P = K[R|t]$ [S1, unofficial].
4. Multiply $P$ by the homogeneous 3D point $\mathbf{M}$ to obtain the homogeneous 2D point $\mathbf{m}' = [x', y', w]^T$ [S2, unofficial].
5. Perform perspective division by the $z$-coordinate (or $w$ component) to get the final 2D coordinates: $\mathbf{m} = [x'/w, y'/w, 1]^T$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_camera_matrix(R, t, fx, fy, s, alpha, u0, v0):
    """
    Computes the 3x4 projective camera matrix P = K[R|t].
    
    Args:
        R: 3x3 rotation matrix.
        t: 3x1 translation vector.
        fx: Focal length in x pixels.
        fy: Focal length in y pixels.
        s: Skew parameter.
        alpha: Aspect ratio.
        u0: Principal point x coordinate.
        v0: Principal point y coordinate.
        
    Returns:
        P: 3x4 camera matrix.
    """
    # Construct intrinsic matrix K
    K = np.array([
        [fx, s, u0],
        [0, alpha * fy, v0],
        [0, 0, 1]
    ])
    
    # Construct extrinsic matrix [R|t]
    Rt = np.hstack((R, t.reshape(3, 1)))
    
    # Compute P
    P = K @ Rt
    return P

def project_point(P, point_3d):
    """
    Projects a 3D point to 2D using the camera matrix P.
    
    Args:
        P: 3x4 camera matrix.
        point_3d: 3D point as a 3x1 vector [X, Y, Z].
        
    Returns:
        point_2d: 2D point as a 2x1 vector [x, y].
    """
    # Convert 3D point to homogeneous coordinates [X, Y, Z, 1]
    point_3d_h = np.append(point_3d, 1)
    
    # Multiply by camera matrix
    point_2d_h = P @ point_3d_h
    
    # Perspective division by the third component (z/w)
    w = point_2d_h[2]
    
    if w == 0:
        raise ValueError("Point is at infinity or on the camera plane.")
        
    x = point_2d_h[0] / w
    y = point_2d_h[1] / w
    
    return np.array([x, y])
```

**Parameters:**
- $f_x, f_y$: Focal lengths in pixels. Typical values depend on sensor and lens [S1, unofficial].
- $s$: Skew parameter. Typically 0 [S1, unofficial].
- $\alpha$: Aspect ratio. Typically 1 [S1, unofficial].
- $(u_0, v_0)$: Principal point coordinates. Typically at the image center [S1, unofficial].

**Edge Cases and Pitfalls:**
- If the depth $w$ (or $z$ in camera coordinates) is zero, the point lies on the camera plane, leading to division by zero [G].

## Subtopics

### [Stereo Projection](5789-stereo-projection.md)
Stereo projection is a technique that applies perspective projection twice with specific adjustments to simulate the distinct viewpoints of an observer's left and right eyes, thereby creating a stereo effect. It differs from standard perspective projection by generating two correlated view frustums rather than one, and differs from symmetric stereo methods by allowing for arbitrary frustum configurations (such as off-axis) to align viewing directions.

### [MCOP Projection Geometry](6527-mcop-projection-geometry.md)
MCOP Projection Geometry is a projection method for light field displays that uses two distinct viewpoints to map 3D world points to projector screen coordinates: an emitter $E$ for horizontal coordinates and a virtual viewer $V$ for vertical coordinates. It solves the problem of perspective-correct rendering on holographic screens that are sharply transmissive horizontally but scatter widely vertically, ensuring view separation is maintained horizontally while allowing vertical viewing flexibility. It differs from standard perspective projection by decoupling the horizontal and vertical projection centers, and from stereo projection by supporting a continuous light field rather than just two views.

## Operating characteristics
- **Strengths:** Simple, computationally efficient, and widely applicable for standard camera models [S1, unofficial].
- **Limitations:** Assumes a pinhole model; does not account for lens distortion or non-pinhole effects without modification [S1, unofficial].

## Background
The pinhole camera model is one of the most widely used models in computer vision to describe the imaging process [S1, unofficial]. It is based on projective geometry and epipolar geometry concepts [S1, unofficial]. The model relates world coordinates to image coordinates via a projective camera matrix [S1, unofficial].

## Sources
[S1] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
[S2] Total Internal Reflection, https://avishek.net/2022/12/05/pytorch-guide-plenoxels-nerf-part-2.html, fetched 2026-10-29 (unofficial)
