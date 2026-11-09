# Elliptical Camera Path Generation

Also called: Elliptical Camera Path with Y Variation, Elliptical Render Path (Y Variation), elliptical camera trajectory

## Overview
Generates a smooth, closed-loop elliptical camera trajectory for volume rendering or NeRF-style scenes, interpolating positions and orientations from a set of input camera poses [S2, unofficial]. It differs from [Orbit Camera Path Generation](7811-orbit-camera-path-generation.md) by allowing non-circular elliptical bounds and optional vertical oscillation, and from [Spiral Camera Path Generation](1635-spiral-camera-path-generation.md) by maintaining a constant radius/height profile rather than expanding or contracting [S1, S2, unofficial].

## How it works
**Inputs:**
- `poses`: Array of shape $(N, 4, 4)$ representing input camera view matrices (world-to-camera or camera-to-world, depending on convention; typically world-to-camera in these implementations, but the logic relies on position extraction).
- `n_frames`: Integer, number of output frames.
- `variation`: Float, amplitude of vertical oscillation (default 1.0).
- `phase`: Float, phase shift for vertical oscillation (default 0.0).

**Outputs:**
- `out_poses`: Array of shape $(n\_frames, 4, 4)$ containing the generated view matrices.

**Data Structures:**
- `center`: 3D vector $(x, y, z)$.
- `offset`: 3D vector defining the ellipse center in the horizontal plane.
- `sc`: 3D vector of scaling factors derived from percentiles.
- `low`, `high`: 3D vectors defining the bounding box of the ellipse in the horizontal plane.
- `z_low`, `z_high`: Scalars defining the vertical bounds.
- `avg_up`: 3D vector for the up direction.

**Steps:**

1.  **Compute Center:** Calculate the focal point center using `focus_point_fn(poses)`. This function typically computes the median or mean of the camera positions extracted from `poses` [S1, S2, unofficial].
    *   Let $P_i$ be the position (translation component) of the $i$-th pose.
    *   $center = \text{median}(P_0, \dots, P_{N-1})$ [G: Median is robust to outliers; mean is also used in some variants].

2.  **Define Offset and Scaling:**
    *   Set `offset` based on the plane of rotation.
        *   For X-Y plane rotation (Z variation): `offset` = $[center_x, center_y, 0]$ [S2, unofficial].
        *   For X-Z plane rotation (Y variation): `offset` = $[center_x, 0, center_z]$ [S1, unofficial].
    *   Compute scaling factors `sc` using the 90th percentile of the absolute distances of camera positions from `offset` [S1, S2, unofficial]:
        $$ sc = \text{percentile}(|P_i - offset|, 90, \text{axis}=0) $$
    *   Define ellipse bounds:
        $$ low = -sc + offset $$
        $$ high = sc + offset $$

3.  **Compute Vertical Bounds:**
    *   Extract the vertical coordinate $z_i$ (or $y_i$ depending on plane) from all positions $P_i$.
    *   Compute lower and upper bounds using 10th and 90th percentiles [S1, S2, unofficial]:
        $$ z_{low} = \text{percentile}(z_i, 10) $$
        $$ z_{high} = \text{percentile}(z_i, 90) $$

4.  **Generate Positions:**
    *   Define a function `get_positions(theta)` that returns the 3D position for a given angle $\theta \in [0, 2\pi]$.
    *   **Case 1: X-Y Plane with Z Variation** [S2, unofficial]:
        $$ x(\theta) = low_x + (high_x - low_x) \left( \frac{\cos(\theta)}{2} + \frac{1}{2} \right) $$
        $$ y(\theta) = low_y + (high_y - low_y) \left( \frac{\sin(\theta)}{2} + \frac{1}{2} \right) $$
        $$ z(\theta) = variation \cdot \left( z_{low} + (z_{high} - z_{low}) \left( \frac{\cos(\theta + 2\pi \cdot phase)}{2} + \frac{1}{2} \right) \right) + offset_z $$
        *Note: `offset_z` is typically 0 in this configuration.*
    *   **Case 2: X-Z Plane with Y Variation** [S1, unofficial]:
        $$ x(\theta) = low_x + (high_x - low_x) \left( \frac{\cos(\theta)}{2} + \frac{1}{2} \right) $$
        $$ y(\theta) = variation \cdot \left( y_{low} + (y_{high} - y_{low}) \left( \frac{\cos(\theta + 2\pi \cdot phase)}{2} + \frac{1}{2} \right) \right) + height $$
        $$ z(\theta) = low_z + (high_z - low_z) \left( \frac{\sin(\theta)}{2} + \frac{1}{2} \right) $$
        *Note: `height` is typically `center_y` or a specific offset.*

5.  **Sample Angles:**
    *   Generate $n\_frames + 1$ evenly spaced angles from $0$ to $2\pi$:
        $$ \theta_k = \frac{2\pi k}{n\_frames}, \quad k = 0, \dots, n\_frames $$
    *   Compute positions $P_k = \text{get\_positions}(\theta_k)$.
    *   Discard the last duplicate point (where $\theta = 2\pi$ equals $\theta = 0$) to avoid redundant frames [S1, S2, unofficial].

6.  **Determine Up Vector:**
    *   Extract the up vector (typically the second column of the rotation matrix, index `[:, 1]`) from all input poses [S1, S2, unofficial].
    *   Compute the average up vector:
        $$ avg\_up = \frac{1}{N} \sum_{i=0}^{N-1} \text{up}_i $$
    *   Normalize $avg\_up$.
    *   Find the index `ind_up` of the component with the largest absolute value in $avg\_up$.
    *   Set the final up vector:
        $$ up = e_{ind\_up} \cdot \text{sign}(avg\_up_{ind\_up}) $$
        where $e_{ind\_up}$ is the basis vector with 1 at `ind_up` and 0 elsewhere [S1, S2, unofficial]. This ensures a stable, axis-aligned up vector if the average is noisy.

7.  **Construct View Matrices:**
    *   For each position $P_k$, construct the view matrix.
    *   **Convention 1 (S2):** View matrix looking from $P_k$ towards $center$.
        $$ \text{view} = \text{viewmatrix}(P_k - center, up, P_k) $$
        *Note: `viewmatrix` typically constructs a matrix with forward = normalize(center - P_k), right = normalize(forward x up), up = right x forward.*
    *   **Convention 2 (S1):** View matrix looking from $P_k$ with center as look-at.
        $$ \text{view} = \text{viewmatrix}(center - P_k, up, P_k) $$
    *   Return the stack of $n\_frames$ view matrices.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import math

def focus_point_fn(poses):
    """Compute median of camera positions."""
    positions = poses[:, :3, 3]
    return np.median(positions, axis=0)

def viewmatrix(look_dir, up, position):
    """Construct a view matrix given position, look direction, and up vector."""
    forward = np.array(look_dir) / np.linalg.norm(look_dir)
    right = np.cross(forward, up)
    right = right / np.linalg.norm(right)
    new_up = np.cross(right, forward)
    
    # Rotation matrix columns: right, new_up, -forward
    R = np.column_stack((right, new_up, -forward))
    # Translation: -R @ position
    T = -R @ position
    
    # Homogeneous matrix
    view = np.eye(4)
    view[:3, :3] = R
    view[:3, 3] = T
    return view

def generate_ellipse_path(poses, n_frames, variation=1.0, phase=0.0, plane='xy'):
    """
    Generate elliptical camera path.
    plane: 'xy' (Z variation) or 'xz' (Y variation)
    """
    center = focus_point_fn(poses)
    
    # Define offset based on plane
    if plane == 'xy':
        offset = np.array([center[0], center[1], 0.0])
        # Vertical axis is Z
        vert_idx = 2
        horiz_indices = [0, 1]
    elif plane == 'xz':
        offset = np.array([center[0], 0.0, center[2]])
        # Vertical axis is Y
        vert_idx = 1
        horiz_indices = [0, 2]
        height = center[1] # Use center Y as base height
    else:
        raise ValueError("Plane must be 'xy' or 'xz'")

    positions = poses[:, :3, 3]
    
    # Compute scaling factors (90th percentile of abs distance from offset)
    dists = np.abs(positions - offset)
    sc = np.percentile(dists, 90, axis=0)
    
    # Ellipse bounds
    low = -sc + offset
    high = sc + offset
    
    # Vertical bounds (10th and 90th percentiles)
    vert_coords = positions[:, vert_idx]
    z_low = np.percentile(vert_coords, 10)
    z_high = np.percentile(vert_coords, 90)
    
    # Get up vector
    avg_up = np.mean(poses[:, :3, 1], axis=0)
    avg_up = avg_up / np.linalg.norm(avg_up)
    ind_up = np.argmax(np.abs(avg_up))
    up = np.eye(3)[ind_up] * np.sign(avg_up[ind_up])
    
    # Generate positions
    thetas = np.linspace(0, 2 * np.pi, n_frames + 1)
    out_positions = []
    
    for theta in thetas:
        # Horizontal interpolation
        # Map cos/sin from [-1, 1] to [0, 1] for linear interpolation between low and high
        cos_val = math.cos(theta)
        sin_val = math.sin(theta)
        
        # X and Z (or Y) horizontal components
        x_frac = cos_val * 0.5 + 0.5
        z_frac = sin_val * 0.5 + 0.5
        
        # Vertical component with phase shift
        vert_frac = math.cos(theta + 2 * np.pi * phase) * 0.5 + 0.5
        
        pos = np.zeros(3)
        
        if plane == 'xy':
            pos[0] = low[0] + (high[0] - low[0]) * x_frac
            pos[1] = low[1] + (high[1] - low[1]) * z_frac # sin maps to Y
            pos[2] = variation * (z_low + (z_high - z_low) * vert_frac) + offset[2]
        elif plane == 'xz':
            pos[0] = low[0] + (high[0] - low[0]) * x_frac
            pos[1] = variation * (z_low + (z_high - z_low) * vert_frac) + height
            pos[2] = low[2] + (high[2] - low[2]) * z_frac # sin maps to Z
            
        out_positions.append(pos)
        
    # Remove last duplicate
    out_positions = out_positions[:-1]
    
    # Construct view matrices
    out_poses = []
    for p in out_positions:
        # Look at center
        look_dir = center - p
        vm = viewmatrix(look_dir, up, p)
        out_poses.append(vm)
        
    return np.array(out_poses)
```

**Edge Cases and Pitfalls:**
- **Degenerate Poses:** If all input poses are collinear or identical, `sc` may be zero, resulting in a single point. Guard against division by zero or zero-length vectors in `viewmatrix` [G].
- **Up Vector Ambiguity:** If `avg_up` is near zero (e.g., poses are evenly distributed in a sphere), the axis selection heuristic may be unstable. A fixed up vector (e.g., $[0, 1, 0]$) might be preferred in such cases [G].
- **Plane Selection:** Ensure the `plane` parameter matches the scene's dominant orientation. Incorrect plane selection leads to flat or inverted ellipses [S1, S2, unofficial].
- **Duplicate Frames:** The loop generates $n\_frames + 1$ points, and the last one is discarded to avoid closing the loop with a duplicate frame [S1, S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N + F)$, where $N$ is the number of input poses and $F$ is the number of output frames. The percentile computations are $O(N)$, and position/view matrix generation is $O(F)$ [S1, S2, unofficial].
- **Memory Complexity:** $O(N + F)$ to store input poses and output poses [S1, S2, unofficial].
- **Strengths:** Smooth, continuous path; handles arbitrary input pose distributions by adapting ellipse bounds; optional vertical oscillation adds visual interest [S1, S2, unofficial].
- **Limitations:** Assumes a roughly elliptical distribution of input poses; may not fit scenes with complex, non-elliptical camera trajectories [G].
- **When to Use:** When a smooth, closed-loop orbit is desired for fly-through visualizations [S1, S2, unofficial].
- **When Not to Use:** For scenes with highly irregular camera paths or when a spiral (expanding/contracting) path is needed [S1, S2, unofficial].

## Background
Originated in the context of NeRF and Gaussian Splatting visualization tools, specifically in repositories like `mrGSplat` and `VolSplat` [S1, S2, unofficial]. It provides a robust method for generating camera paths from sparse or irregular input poses, commonly used in 3D reconstruction and volume rendering pipelines [S1, S2, unofficial].

## Sources
[S1] meshroomHub/mrGSplat, https://github.com/meshroomHub/mrGSplat, fetched 2026-10-27 (unofficial)
[S2] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
