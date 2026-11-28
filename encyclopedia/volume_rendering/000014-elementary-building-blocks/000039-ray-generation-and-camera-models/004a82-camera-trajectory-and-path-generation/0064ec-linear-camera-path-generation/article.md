# Linear Camera Path Generation
Also called: Camera Translation Methods

## Overview
Linear Camera Path Generation creates a sequence of camera poses along a straight line segment between a start and end position, maintaining a fixed look-at point [S1, unofficial]. It solves the problem of generating consistent, evenly spaced viewpoints for volume rendering or 3D visualization without rotational artifacts [S1, unofficial][S2]. It is distinguished from sibling methods like [B-Spline Camera Path Interpolation](11453-b-spline-camera-path-interpolation.md) by its lack of curvature and from [Arcball Camera](1639-arcball-camera.md) by its restriction to translational motion only.

## How it works
**Inputs and Outputs**
*   **Inputs**: `start_pos` (3D vector), `end_pos` (3D vector), `look_at` (3D vector), `num_views` (integer), and optional intrinsics (`width`, `height`, `fx`, `fy`) [S1, unofficial].
*   **Outputs**: A list of `PinholeCamera` objects, each containing extrinsic parameters (position, orientation) and intrinsic parameters [S1, unofficial].

**Data Structures**
*   `PinholeCamera`: Contains fields for position $\mathbf{p}$, look-at point $\mathbf{l}$, and intrinsics $f_x, f_y, w, h$ [S1, unofficial].
*   The path is represented as a list of these objects [S1, unofficial].

**Parameters**
*   `start_pos`: Starting 3D coordinate.
*   `end_pos`: Ending 3D coordinate.
*   `look_at`: Fixed focal point in 3D space.
*   `num_views`: Number of intermediate poses to generate. Default: 30 [S1, unofficial].
*   `width`: Image width in pixels. Default: 1280 [S1, unofficial].
*   `height`: Image height in pixels. Default: 720 [S1, unofficial].
*   `fx`: Focal length in pixels (x). Default: 960.0 [S1, unofficial].
*   `fy`: Focal length in pixels (y). Default: 960.0 [S1, unofficial].

**Algorithm**
1.  Calculate the direction vector $\mathbf{d} = \text{end\_pos} - \text{start\_pos}$.
2.  For each view index $i$ from $0$ to $\text{num\_views} - 1$:
    a.  Compute the interpolation parameter $t = \frac{i}{\text{num\_views} - 1}$ if $\text{num\_views} > 1$, else $t=0$ [S1, unofficial].
    b.  Compute the camera position $\mathbf{p}_i = \text{start\_pos} + t \cdot \mathbf{d}$ [S1, unofficial].
    c.  Construct a `PinholeCamera` with position $\mathbf{p}_i$, fixed look-at point $\text{look\_at}$, and specified intrinsics [S1, unofficial].
3.  Return the list of cameras [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_linear_path(start_pos, end_pos, look_at, num_views=30, 
                         width=1280, height=720, fx=960.0, fy=960.0):
    """
    Generate a list of PinholeCamera objects along a linear path.
    
    Args:
        start_pos: np.ndarray, shape (3,)
        end_pos: np.ndarray, shape (3,)
        look_at: np.ndarray, shape (3,)
        num_views: int
        width: int
        height: int
        fx: float
        fy: float
        
    Returns:
        list of PinholeCamera objects
    """
    cameras = []
    direction = end_pos - start_pos
    
    # Handle edge case of single view or zero distance
    if num_views <= 1:
        t = 0.0
    else:
        # Loop through all views
        for i in range(num_views):
            # Linear interpolation parameter
            t = i / (num_views - 1)
            
            # Calculate current position
            current_pos = start_pos + direction * t
            
            # Create camera object
            # Note: Orientation is derived from position and fixed look_at point.
            # The specific rotation matrix calculation (up vector, cross products)
            # is assumed to be handled by the PinholeCamera constructor 
            # when given position and look_at.
            cam = PinholeCamera(
                position=current_pos,
                look_at=look_at,
                width=width,
                height=height,
                fx=fx,
                fy=fy
            )
            cameras.append(cam)
            
    return cameras

class PinholeCamera:
    def __init__(self, position, look_at, width, height, fx, fy):
        self.position = np.array(position)
        self.look_at = np.array(look_at)
        self.width = width
        self.height = height
        self.fx = fx
        self.fy = fy
        # Orientation is computed internally based on position and look_at
        self._compute_orientation()

    def _compute_orientation(self):
        # [G] Standard look-at matrix construction
        # Forward vector
        forward = self.look_at - self.position
        forward = forward / np.linalg.norm(forward)
        
        # Assume standard up vector (0, 1, 0) unless specified otherwise
        up = np.array([0.0, 1.0, 0.0])
        
        # Right vector
        right = np.cross(forward, up)
        right = right / np.linalg.norm(right)
        
        # True up vector
        up = np.cross(right, forward)
        
        # Rotation matrix R (3x3)
        self.R = np.column_stack((right, up, -forward))
        
    def get_extrinsic_matrix(self):
        """Returns 4x4 extrinsic matrix [R|t; 0|1]"""
        M = np.eye(4)
        M[:3, :3] = self.R
        M[:3, 3] = self.position
        return M
```

**Edge Cases and Pitfalls**
*   **Zero Distance**: If `start_pos` equals `end_pos`, the direction vector is zero. The code handles this by keeping the position constant at `start_pos` for all views [S1, unofficial].
*   **Single View**: If `num_views` is 1, $t$ is forced to 0 to avoid division by zero, resulting in a single camera at `start_pos` [S1, unofficial].
*   **Precision**: Floating-point interpolation errors are negligible for typical volume rendering resolutions but may accumulate in extremely long paths [G].

## Subtopics

### Iterative Movement
`iter_move()` provides a generator-like interface to yield positions sequentially, useful for streaming or memory-constrained environments [S2, S3]. It differs from `generate_linear_path` by not storing all poses in memory at once.

### Single Position Set
`set_position()` updates the camera location in-place while keeping the focus fixed. This is useful for interactive viewports rather than batch generation [S2, S3].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is `num_views`, as each view requires a constant number of vector operations [S1, unofficial].
*   **Memory Complexity**: $O(N)$ to store the list of camera objects [S1, unofficial].
*   **Performance**: Extremely fast; negligible compared to the volume rendering step itself [S1, unofficial].
*   **Strengths**: Simple, deterministic, and easy to debug. Ideal for orthogonal scans or linear fly-throughs [S1, unofficial].
*   **Limitations**: Cannot represent curved trajectories or complex camera motions. May cause visual discontinuities if the look-at point is occluded or if the path passes through the volume data in a non-intuitive way [S1, unofficial].
*   **When to use**: When a straight-line traversal is required, such as in tomographic reconstruction validation or simple fly-throughs [S1, unofficial].
*   **When not to use**: For cinematic shots requiring smooth turns ([B-Spline](11453-b-spline-camera-path-interpolation.md)) or user-controlled exploration ([First-Person](2721-first-person-camera-control.md)) [S2].

## Background
The method originates from the `yt` project's volume rendering visualization tools, specifically the `Camera` class which provides translation-related methods like `set_position` and `iter_move` [S2, S3]. It is also implemented in the `GaussianScan-Recon` repository for generating camera poses from 3D points [S1, unofficial]. The approach is a standard technique in computer graphics for generating linear camera trajectories.

## Sources
[S1] Laityperfect7/GaussianScan-Recon, https://github.com/Laityperfect7/GaussianScan-Recon, fetched 2026-10-27 (unofficial)
[S2] 3D Visualization and Volume Rendering, https://yt-project.org/doc/visualizing/volume_rendering.html, fetched 2026-10-28
[S3] https://yt-project.org/docs/dev/visualizing/volume_rendering.html#volume-rendering-components, https://yt-project.org/docs/dev/visualizing/volume_rendering.html, fetched 2026-10-29
