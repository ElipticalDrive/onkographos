# Pinhole Camera Projection (Pixel to Camera)

Also called: Camera Ray Lifting, Depth Map to Camera Coordinates, Inverse Projection, Pixel to Camera, Ray Generation from Pixel Coordinates using Intrinsics

## Overview
This method converts 2D pixel coordinates $(u, v)$ and a depth value $z$ into 3D camera-space homogeneous coordinates using the camera's intrinsic parameters [S1, S2, S3, unofficial]. It differs from sibling methods like [Ray Generation from Pixel Coordinates](2514-ray-generation-from-pixel-coordinates.md) by explicitly accepting a depth input $z$ to produce a specific 3D point in camera space rather than a ray direction vector [S1, S2, S3, unofficial].

## How it works
**Inputs:**
- Pixel coordinates: $u, v$ (float arrays, shape $[B, N]$) [S2, S3, unofficial].
- Depth: $z$ (float array, shape $[B, N]$) [S2, S3, unofficial].
- Intrinsics: A $3 \times 3$ matrix $K$ (float array, shape $[B, 3, 3]$) [S2, S3, unofficial].

**Data Structures:**
- The intrinsic matrix $K$ is structured as:
  $$
  K = \begin{bmatrix}
  f_x & s & c_x \\
  0 & f_y & c_y \\
  0 & 0 & 1
  \end{bmatrix}
  $$
  Where:
  - $f_x = K_{0,0}$: Focal length in pixels (x-axis) [S1, S2, S3, unofficial].
  - $f_y = K_{1,1}$: Focal length in pixels (y-axis) [S1, S2, S3, unofficial].
  - $c_x = K_{0,2}$: Principal point x-coordinate [S1, S2, S3, unofficial].
  - $c_y = K_{1,2}$: Principal point y-coordinate [S1, S2, S3, unofficial].
  - $s = K_{0,1}$: Skew coefficient [S1, S3, unofficial]. Source [S2, unofficial] assumes $s=0$.

**Steps:**
1. Extract parameters $f_x, f_y, c_x, c_y, s$ from the intrinsic matrix $K$ [S1, S2, S3, unofficial].
2. Compute the camera-space x-coordinate $x_{lift}$:
   $$
   x_{lift} = \frac{u - c_x + \frac{c_y \cdot s}{f_y} - \frac{s \cdot v}{f_y}}{f_x} \cdot z
   $$
   [S1, S3, unofficial]. Source [S1, unofficial] applies an absolute value to depth ($|z|$), while [S2, S3, unofficial] use $z$ directly [S1, S2, S3, unofficial].
3. Compute the camera-space y-coordinate $y_{lift}$:
   $$
   y_{lift} = \frac{v - c_y}{f_y} \cdot z
   $$
   [S1, S2, S3, unofficial].
4. Construct the homogeneous coordinate vector by stacking $x_{lift}, y_{lift}, z, 1$ along the last dimension [S1, S2, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def pixel_to_camera(u, v, z, intrinsics):
    """
    Lifts pixel coordinates (u, v) with depth z to camera space homogeneous coordinates.
    
    Args:
        u: Tensor of shape [B, N], pixel x-coordinates.
        v: Tensor of shape [B, N], pixel y-coordinates.
        z: Tensor of shape [B, N], depth values.
        intrinsics: Tensor of shape [B, 3, 3], camera intrinsic matrix K.
        
    Returns:
        Tensor of shape [B, N, 4], homogeneous camera coordinates [x, y, z, 1].
    """
    # Extract intrinsic parameters
    fx = intrinsics[:, 0, 0]
    fy = intrinsics[:, 1, 1]
    cx = intrinsics[:, 0, 2]
    cy = intrinsics[:, 1, 2]
    skew = intrinsics[:, 0, 1]
    
    # Compute x_lift
    x_lift = ((u - cx + cy * skew / fy - skew * v / fy) / fx) * z
    
    # Compute y_lift
    y_lift = ((v - cy) / fy) * z
    
    # Stack into homogeneous coordinates [x, y, z, 1]
    ones = torch.ones_like(z)
    result = torch.stack([x_lift, y_lift, z, ones], dim=-1)
    
    return result
```

**Edge Cases and Pitfalls:**
- **Skew Handling:** If the camera has no skew ($s=0$), the term $\frac{c_y \cdot s}{f_y} - \frac{s \cdot v}{f_y}$ vanishes [S1, S3, unofficial].
- **Depth Sign:** Source [S1, unofficial] applies absolute value to $z$ ($|z|$), while [S2, S3, unofficial] use $z$ directly [S1, S2, S3, unofficial]. Implementers should verify if depth is signed or unsigned in their specific pipeline [G].
- **Division by Zero:** Ensure $f_x$ and $f_y$ are non-zero to prevent numerical instability [G].

## Subtopics

### Ray Generation from Pixel Coordinates
- **What it is:** Generates a ray direction vector from pixel coordinates without depth, assuming a unit depth or normalized device coordinates.
- **Difference:** Returns a direction vector (ray) rather than a specific 3D point.
- **When to choose:** When performing ray marching or unbounded volume rendering where depth is determined by intersection, not input.
- **Link:** [Ray Generation from Pixel Coordinates](2514-ray-generation-from-pixel-coordinates.md)

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Requires depth input; does not account for extrinsic camera transformation (view matrix) [S1, S2, unofficial].
- **When to use:** When converting depth maps or pixel-wise depth estimates into 3D camera-space points for further processing (e.g., point cloud generation, volume rendering initialization) [S1, S2, unofficial].

## Background
Not given in the sources.

## Sources
[S1] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
[S2] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S3] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
