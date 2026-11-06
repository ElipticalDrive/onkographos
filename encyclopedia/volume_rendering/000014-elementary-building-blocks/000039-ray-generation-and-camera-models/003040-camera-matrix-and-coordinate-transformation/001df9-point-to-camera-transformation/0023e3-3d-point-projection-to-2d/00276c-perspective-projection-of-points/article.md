# Perspective Projection of Points

Also called: Camera Class, Camera Transformation, Perspective Projection, Pinhole Camera Model, Point Projection, Point Projection to 2D

## Overview
Perspective projection maps 3D world points to 2D image coordinates by intersecting a ray from the center of projection through the point with an image plane [S1, unofficial]. It differs from orthographic projection by introducing depth-dependent scaling (perspective division), which is essential for realistic camera models and visibility computation in volume rendering pipelines [S1, S2, unofficial].

## How it works
**Inputs:**
- A 3D world point $M = [X, Y, Z]^T$ in homogeneous coordinates $[X, Y, Z, 1]^T$ [S1, unofficial].
- A projective camera matrix $P$ of size $3 \times 4$ [S1, unofficial].

**Outputs:**
- A 2D image point $m = [x, y]^T$ in homogeneous coordinates $[x, y, 1]^T$ [S1, unofficial].

**Data Structures:**
The core data structure is the camera matrix $P$, decomposed into intrinsic and extrinsic parameters:
$$P = K[R|t]$$
where $K$ is the $3 \times 3$ calibration matrix and $[R|t]$ is the $3 \times 4$ extrinsic matrix [S1, unofficial].

The calibration matrix $K$ layout:
$$K = \begin{pmatrix} f_x & s & u_0 \\ 0 & \alpha f_y & v_0 \\ 0 & 0 & 1 \end{pmatrix}$$
Fields:
- $f_x, f_y$: Focal lengths in pixels. Calculated as $f_x = f/k_x$ and $f_y = f/k_y$, where $f$ is focal length in mm and $k_x, k_y$ are pixels/mm [S1, unofficial].
- $s$: Skew parameter. Typically 0 [S1, unofficial].
- $\alpha$: Aspect ratio. Typically 1 [S1, unofficial].
- $u_0, v_0$: Principal point coordinates (intersection of optical axis with image plane) [S1, unofficial].

The extrinsic matrix $[R|t]$ consists of a $3 \times 3$ rotation matrix $R$ and a $3 \times 1$ translation vector $t$, transforming world coordinates to the camera coordinate system [S1, unofficial].

**Steps:**
1. Construct the homogeneous world point vector $\mathbf{M}_h = [X, Y, Z, 1]^T$ [S1, unofficial].
2. Compute the homogeneous image coordinates $\mathbf{m}_h$ by matrix multiplication:
   $$\mathbf{m}_h = P \mathbf{M}_h$$
   This yields a 3-element vector $[u, v, w]^T$ [S1, unofficial].
3. Perform perspective division to obtain normalized image coordinates:
   $$x = u / w$$
   $$y = v / w$$
   The scale factor $\lambda$ in the source equation $\lambda [x, y, 1]^T = P [X, Y, Z, 1]^T$ corresponds to $w$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def perspective_project(point_world, camera_matrix):
    """
    Projects a 3D world point to 2D image coordinates using a 3x4 camera matrix.
    
    Args:
        point_world: numpy array of shape (3,) or (4,) representing [X, Y, Z] or [X, Y, Z, 1].
        camera_matrix: numpy array of shape (3, 4) representing P.
        
    Returns:
        numpy array of shape (2,) representing [x, y] image coordinates.
    """
    # Ensure point is in homogeneous coordinates (4D)
    if point_world.shape[0] == 3:
        point_h = np.append(point_world, 1.0)
    else:
        point_h = point_world
        
    # Matrix multiplication: m_h = P * M_h
    # P is (3,4), M_h is (4,) -> result is (3,)
    m_h = np.dot(camera_matrix, point_h)
    
    # Perspective division
    # m_h = [u, v, w]
    # x = u / w, y = v / w
    w = m_h[2]
    
    # Guard against division by zero (point at or behind camera)
    if w == 0:
        raise ValueError("Point is at the camera center or projection is degenerate.")
        
    x = m_h[0] / w
    y = m_h[1] / w
    
    return np.array([x, y])
```

**Parameters:**
- $f_x, f_y$: Focal lengths in pixels [S1, unofficial].
- $s$: Skew, typically 0 [S1, unofficial].
- $\alpha$: Aspect ratio, typically 1 [S1, unofficial].
- $u_0, v_0$: Principal point [S1, unofficial].

**Edge Cases:**
- If $w=0$, the point lies on the plane passing through the center of projection parallel to the image plane (or at infinity in the projection direction). Division by zero occurs [S1, unofficial].
- Points with $w < 0$ are behind the camera. Standard implementations may clip these or produce inverted coordinates [S1, G, unofficial].

## Subtopics

### Stereo Projection
A perspective projection applied twice with adjustments to simulate the viewpoint for an observer's left and right eye, creating a stereo effect [S3, unofficial]. It is chosen when rendering for stereoscopic displays. The two resulting versions of the scene are drawn in succession using shuttered glasses synced with the monitor, or simultaneously using stereo viewers [S3, unofficial].

### MCOP Projection Geometry
Projects 3D world points to projector screen coordinates using two distinct viewpoints: an emitter $E$ for horizontal coordinates and a virtual viewer $V$ for vertical coordinates [S4]. It is used for interactive light field displays where the screen scatters widely in the vertical direction but is selective horizontally [S4]. The screen projected position $S$ is computed by intersecting the ray from emitter $E$ with the screen plane $z=0$ for $x$, and the ray to virtual viewer $V$ for $y$. The position is then remapped to normalized coordinates $H$ based on the image rectangle $R^+, R^-$ [S4]. Parameters include virtual viewer height $V_y$, distance $V_z$, emitter origin $E=(E_x, E_y, E_z)$, world point $P=(P_x, P_y, P_z)$, and rectangle bounds [S4].

## Operating characteristics
- **Time Complexity:** $O(1)$ per point for the matrix multiplication and division [S1, unofficial].
- **Memory Complexity:** $O(1)$ for storing the camera matrix and point coordinates [S1, unofficial].
- **Strengths:** Provides geometrically accurate mapping for perspective cameras; standard in computer vision and graphics [S1, unofficial].
- **Limitations:** Requires careful handling of points behind the camera ($w \le 0$); assumes a pinhole model which may not account for lens distortion [S1, unofficial].
- **When to use:** When realistic depth cues are required or when interfacing with standard camera calibration data [S1, unofficial].
- **When not to use:** For orthographic views or when lens distortion correction is required without additional steps [S1, unofficial].

## Background
The pinhole camera model is one of the most widely used models in computer vision to describe the imaging process [S1, unofficial]. It is based on projective geometry, where a 3D point projects onto an image plane via a line joining the point to the center of projection [S1, unofficial]. The decomposition $P=K[R|t]$ separates intrinsic (calibration) and extrinsic (pose) parameters, a standard formulation in epipolar and projective geometry literature [S1, unofficial].

## Sources
[S1] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
[S2] Total Internal Reflection, https://avishek.net/2022/12/05/pytorch-guide-plenoxels-nerf-part-2.html, fetched 2026-10-29 (unofficial)
[S3] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S4] GPU Accelerated Direct Volume Rendering on an Interactive Light Field Display, https://doi.org/10.1111/j.1467-8659.2008.01120.x, fetched 2026-10-29
