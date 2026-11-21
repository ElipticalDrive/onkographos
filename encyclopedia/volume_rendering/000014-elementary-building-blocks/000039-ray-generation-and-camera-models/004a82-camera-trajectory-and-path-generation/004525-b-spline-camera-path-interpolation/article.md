# B-Spline Camera Path Interpolation

Also called: B-Spline Camera Path, Multidimensional B-Spline Interpolation for Camera Poses, Spline Interpolated Camera Path

## Overview
B-Spline Camera Path Interpolation generates a smooth, continuous camera trajectory between discrete input keyframe poses using multidimensional B-spline interpolation [S1, S2, unofficial]. It solves the problem of jittery or discontinuous motion in volume rendering and novel view synthesis by ensuring positional and rotational continuity across the entire path [S1, S2, unofficial]. It differs from linear interpolation by providing $C^2$ continuity (smooth acceleration) and from least-squares approximation by allowing exact interpolation of keyframes when the smoothing factor is zero, avoiding the introduction of artifacts not present in the source data [S1, unofficial].

## How it works
**Inputs:**
- `poses`: Array of shape $(N, 3, 4)$ representing $N$ input camera poses in world-to-camera or camera-to-world format (typically extrinsic matrix columns 0-2 and translation vector column 3) [S1, S2, unofficial].
- `n_interp`: Integer, number of interpolated poses to generate between each pair of consecutive keyframes [S1, S2, unofficial].
- `spline_degree`: Integer, polynomial degree of the B-spline. Default 5 [S1, S2, unofficial].
- `smoothness`: Float, smoothing factor for the spline fit. $0$ forces exact interpolation through all points. Default 0.03 [S1, S2, unofficial].
- `rot_weight`: Float, relative weighting of rotation vs. translation in the pose decomposition. Default 0.1 [S1, S2, unofficial].

**Outputs:**
- `new_poses`: Array of shape $(M, 3, 4)$, where $M = n\_interp \times (N - 1)$, containing the interpolated camera poses [S1, S2, unofficial].

**Data Structures:**
The method internally converts $3 \times 4$ pose matrices into a structured point representation $(P, L, U)$ where $P$ is position, $L$ is look-at, and $U$ is up vector. These are treated as a single multidimensional array of shape $(N, 9)$ for interpolation [S1, S2, unofficial].

**Steps:**
1. **Pose Decomposition:** Convert each input pose matrix $W \in \mathbb{R}^{3 \times 4}$ into three 3D vectors: Position ($pos$), Look-at ($lookat$), and Up ($up$) [S1, S2, unofficial].
   - $pos = W[:, 0:3, 3]$
   - $lookat = W[:, 0:3, 3] - rot\_weight \times W[:, 0:3, 2]$
   - $up = W[:, 0:3, 3] + rot\_weight \times W[:, 0:3, 1]$
   - Here, $W[:, 0:3, 2]$ is the z-axis (forward/look direction) and $W[:, 0:3, 1]$ is the y-axis (up direction) of the camera basis [S1, S2, unofficial].

2. **Spline Fitting:** Construct a multidimensional B-spline curve $S(u)$ that fits the sequence of points $(pos_i, lookat_i, up_i)$ for $i=0 \dots N-1$ [S1, S2, unofficial].
   - Input to spline fitter: Transposed array of shape $(9, N)$ containing all position, look-at, and up coordinates [S1, unofficial].
   - Degree $k = \min(spline\_degree, N - 1)$ [S1, unofficial].
   - Smoothness parameter $s = smoothness$ [S1, S2, unofficial].
   - The fitter returns a tuple $(tck, u)$, where $tck$ contains the knots, coefficients, and degree, and $u$ are the parameter values (typically uniform or chord-length) [S1, S2, unofficial].

3. **Evaluation:** Sample the spline curve at $M = n\_interp \times (N - 1)$ points [S1, S2, unofficial].
   - Parameter vector $u_{eval} = \text{linspace}(0, 1, M, \text{endpoint=False})$ [S1, unofficial].
   - Evaluate $S(u_{eval})$ to get interpolated points of shape $(M, 9)$ [S1, S2, unofficial].

4. **Pose Reconstruction:** Convert the interpolated $(pos, lookat, up)$ vectors back into $3 \times 4$ pose matrices [S1, S2, unofficial].
   - Extract $pos$, $lookat$, $up$ from the interpolated points.
   - Reconstruct the camera basis vectors:
     - $z_{axis} = -(lookat - pos) / \|lookat - pos\|$ (normalized forward vector) [G].
     - $x_{axis} = z_{axis} \times up / \|z_{axis} \times up\|$ (normalized right vector) [G].
     - $y_{axis} = x_{axis} \times z_{axis}$ (normalized up vector) [G].
   - Construct the pose matrix $W_{new}$ with columns $[x_{axis}, y_{axis}, z_{axis}, pos]$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.interpolate import splprep, splev

def poses_to_points(poses, rot_weight=0.1):
    """
    Convert N poses of shape (N, 3, 4) to (N, 9) array of (pos, lookat, up).
    """
    pos = poses[:, :3, 3]
    lookat = pos - rot_weight * poses[:, :3, 2]
    up = pos + rot_weight * poses[:, :3, 1]
    return np.concatenate([pos, lookat, up], axis=-1)

def points_to_poses(points):
    """
    Convert (N, 9) array of (pos, lookat, up) back to (N, 3, 4) poses.
    """
    pos = points[:, :3]
    lookat = points[:, 3:6]
    up = points[:, 6:9]
    
    # Reconstruct basis vectors
    # Forward vector (negative z-axis in standard OpenGL/Camera coords)
    forward = lookat - pos
    z_axis = -forward / np.linalg.norm(forward, axis=1, keepdims=True)
    
    # Right vector (x-axis)
    right = np.cross(z_axis, up)
    x_axis = right / np.linalg.norm(right, axis=1, keepdims=True)
    
    # Up vector (y-axis)
    y_axis = np.cross(x_axis, z_axis)
    
    # Construct pose matrix [x, y, z, pos]
    poses = np.zeros((points.shape[0], 3, 4))
    poses[:, :3, 0] = x_axis
    poses[:, :3, 1] = y_axis
    poses[:, :3, 2] = z_axis
    poses[:, :3, 3] = pos
    return poses

def generate_interpolated_path(poses, n_interp=10, spline_degree=5, smoothness=0.03, rot_weight=0.1):
    """
    Generate smooth camera path using B-spline interpolation.
    
    Args:
        poses: (N, 3, 4) array of input poses.
        n_interp: Number of interpolated poses between keyframes.
        spline_degree: Degree of B-spline.
        smoothness: Smoothing factor (0 for exact interpolation).
        rot_weight: Weight for rotation vs translation in decomposition.
        
    Returns:
        new_poses: (M, 3, 4) array of interpolated poses, M = n_interp * (N-1).
    """
    N = poses.shape[0]
    
    # Step 1: Decompose poses
    points = poses_to_points(poses, rot_weight=rot_weight)
    
    # Step 2: Fit B-spline
    # splprep expects points in shape (ndim, N), so we transpose
    tck, u = splprep(points.T, k=min(spline_degree, N - 1), s=smoothness)
    
    # Step 3: Evaluate spline
    M = n_interp * (N - 1)
    u_eval = np.linspace(0, 1, M, endpoint=False)
    new_points = splev(u_eval, tck)
    
    # splev returns list of arrays, one for each dimension. Stack them.
    new_points = np.array(new_points).T
    
    # Step 4: Reconstruct poses
    new_poses = points_to_poses(new_points)
    
    return new_poses
```

## Subtopics

### Spline Render Path Creation with Exposure Interpolation
This variant extends the base B-spline interpolation to simultaneously interpolate per-frame exposure values, preventing flickering in rendered sequences [S1, unofficial]. It identifies keyframe indices from a configuration file or directory listing, generates the camera path using the standard B-spline method, and then applies a separate 1D B-spline interpolation to the logarithm of exposure values [S1, unofficial]. If exposure interpolation is enabled but exposure data is missing, it raises an error [S1, unofficial]. A higher smoothness value (e.g., 20) is typically used for exposure interpolation to ensure stability [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of keyframes and $M$ is the number of output poses. The spline fitting is $O(N)$ and evaluation is $O(M)$ [S1, S2, unofficial].
- **Memory Complexity:** $O(N)$ for storing the spline coefficients and $O(M)$ for the output poses [S1, S2, unofficial].
- **Strengths:** Produces visually smooth camera motion with continuous derivatives; handles both position and orientation in a unified framework; configurable smoothness allows trade-off between fidelity to keyframes and path smoothness [S1, S2, unofficial].
- **Limitations:** Requires at least $k+1$ keyframes for a spline of degree $k$; exact interpolation ($s=0$) may lead to oscillations if keyframes are noisy or poorly distributed; the linear approximation of rotation via look-at/up vectors does not guarantee unit quaternion interpolation, potentially introducing slight scaling artifacts in the basis vectors [G].
- **When to use:** When generating smooth fly-throughs or orbit paths from sparse keyframes in volume rendering or NeRF training [S1, S2, unofficial].
- **When not to use:** When exact physical camera constraints (e.g., fixed focal length, specific lens distortion) must be preserved without post-processing, or when the input poses are highly erratic and require robust fitting rather than interpolation [S1, unofficial].

## Background
The method originates from the need to generate smooth camera trajectories for novel view synthesis and volume rendering, particularly in the context of NeRF (Neural Radiance Fields) and Gaussian Splatting pipelines [S1, S2, unofficial]. It was popularized by implementations in Google's MultiNeRF repository and adapted in various open-source Gaussian Splatting frameworks like mrGSplat [S1, S2, unofficial]. The approach leverages the mathematical properties of B-splines, specifically their local control and smoothness, to interpolate multidimensional data (position and orientation) simultaneously [S1, unofficial].

## Sources
[S1] google-research/multinerf, https://github.com/google-research/multinerf, fetched 2026-10-26 (unofficial)
[S2] meshroomHub/mrGSplat, https://github.com/meshroomHub/mrGSplat, fetched 2026-10-27 (unofficial)
