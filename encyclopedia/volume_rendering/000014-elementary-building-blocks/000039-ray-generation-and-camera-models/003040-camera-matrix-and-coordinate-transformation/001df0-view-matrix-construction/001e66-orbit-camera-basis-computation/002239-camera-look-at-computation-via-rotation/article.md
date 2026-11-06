# Camera Look-at Computation via Rotation

Also called: Euler Angle Camera Look-at, Spherical Coordinate Camera Orientation

## Overview
This method computes a camera's look-at point by rotating a forward vector around the camera's local X and Z axes using spherical coordinate angles, then adding the result to the camera position [S1, unofficial]. It solves the problem of determining the viewing direction for an orbit camera based on user input angles, enabling navigation in 3D space [S1, unofficial].

## How it works
**Inputs:**
- Camera position $P = (ax, ay, az)$ [S1, unofficial].
- Rotation angles: $x\text{-angle}$ and $z\text{-angle}$ in degrees [S1, unofficial].
- A base forward vector $V = (x, y, z)$ [S1, unofficial].

**Outputs:**
- Look-at point $L = (x', y', z')$ [S1, unofficial].

**Data Structures:**
Not given in the sources.

**Steps:**
1.  Convert angles from degrees to radians:
    $$x_{\text{rad}} = x\text{-angle} \cdot \frac{\pi}{180}$$
    $$z_{\text{rad}} = z\text{-angle} \cdot \frac{\pi}{180}$$
    [S1, unofficial]

2.  Compute trigonometric values:
    $$c_x = \cos(x_{\text{rad}}), \quad s_x = \sin(x_{\text{rad}})$$
    $$c_z = \cos(z_{\text{rad}}), \quad s_z = \sin(z_{\text{rad}})$$
    [S1, unofficial]

3.  Compute rotated vector components:
    $$x_r = c_x c_z x - s_x y + c_x s_z z$$
    $$y_r = c_x s_z x + s_x y + c_x c_z z$$
    The source cuts off the formula for $z_r$ [S1, unofficial].

4.  Add the rotated vector to the camera position to get the look-at point:
    $$x' = x_r + ax$$
    $$y' = y_r + ay$$
    $$z' = z_r + az$$
    [S1, unofficial]

**Parameters:**
- RAD: Conversion factor $\pi/180$ [S1, unofficial].
- Default angles: $x\text{-rotate} = 255$, $z\text{-rotate} = 15$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- The source cuts off the $z'$ formula, leaving the full transformation incomplete [S1, unofficial].
- Coordinate system handedness and axis orientation are not specified [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_look_at(camera_pos, x_angle_deg, z_angle_deg, base_forward=(0, 0, 1)):
    """
    Computes the look-at point for a camera using rotation angles.
    
    Args:
        camera_pos: Tuple (ax, ay, az) representing camera position.
        x_angle_deg: Rotation angle around X axis in degrees.
        z_angle_deg: Rotation angle around Z axis in degrees.
        base_forward: Tuple (x, y, z) representing the initial forward vector.
        
    Returns:
        Tuple (x', y', z') representing the look-at point.
    """
    ax, ay, az = camera_pos
    x, y, z = base_forward
    
    # Convert degrees to radians
    x_rad = x_angle_deg * math.pi / 180.0
    z_rad = z_angle_deg * math.pi / 180.0
    
    # Compute trigonometric values
    c_x = math.cos(x_rad)
    s_x = math.sin(x_rad)
    c_z = math.cos(z_rad)
    s_z = math.sin(z_rad)
    
    # Apply rotation to the base forward vector
    # Formulas for x_r and y_r match the source exactly [S1, unofficial].
    # z_r is completed using a standard rotation matrix pattern [G].
    x_r = c_x * c_z * x - s_x * y + c_x * s_z * z
    y_r = c_x * s_z * x + s_x * y + c_x * c_z * z
    z_r = s_x * s_z * x + c_z * y + s_x * c_z * z  # Standard completion [G]
    
    # Add to camera position
    look_at_x = x_r + ax
    look_at_y = y_r + ay
    look_at_z = z_r + az
    
    return (look_at_x, look_at_y, look_at_z)
```

## Subtopics

### [Camera Control via Euler Angles (Pitch/Yaw)](3431-camera-control-via-euler-angles-pitch-yaw.md)
Computes orientation using yaw and pitch without roll. Chosen for low computational cost and explicit vector construction.

### [Arcball Camera View & Projection Matrix Computation](2210-arcball-camera-view-projection-matrix-computation.md)
Uses quaternion-based rotation for smoother, singularity-free orbiting.

### [Orbit Camera Controls](3262-orbit-camera-controls.md)
General interface for orbit camera input handling.

### [Spherical Camera Path Generation](3934-spherical-camera-path-generation.md)
Generates camera positions along a spherical path.

### [Volume Visualization Camera](4116-volume-visualization-camera.md)
Specific camera setup for volume rendering.

## Operating characteristics

## Background
Used in orbit camera implementations to compute viewing direction from rotation angles [S1, unofficial].

## Sources
[S1] Raymarching with Fennel and LÖVE · Andrey Listopadov, https://andreyor.st/posts/2020-10-15-raymarching-with-fennel-and-love/, fetched 2026-10-30 (unofficial)
