# Elliptical Camera Path Generation

Also called: Elliptical Camera Path with Y Variation, Elliptical Path Generation, Elliptical Render Path (Y Variation), Elliptical Render Path Generation, elliptical camera trajectory

## Overview
Generates a closed-loop elliptical camera trajectory centered on a focal point derived from input training poses, with optional asymmetric height variation along the vertical axis [S1, S3, S4, unofficial]. It differs from [Orbit Camera Path Generation](7811-orbit-camera-path-generation.md) by allowing independent scaling of the ellipse axes and non-circular vertical oscillation, and from [Spiral Camera Path Generation](1635-spiral-camera-path-generation.md) by maintaining a constant radius (or bounded height range) rather than a continuously expanding or contracting radius [S1, S3, S4, unofficial].

## How it works
**Inputs:**
- `poses`: Array of shape $(N, 4, 4)$ containing $N$ input camera poses [S1, S2, S3, S4, unofficial].
- `n_frames`: Integer, number of output frames. Default 120 [S4, unofficial].
- `z_variation`: Float, amplitude of height variation. Default 0 [S1, S4, unofficial].
- `z_phase`: Float, phase shift for height variation. Default 0 [S1, S4, unofficial].
- `const_speed`: Boolean, whether to resample for constant velocity. Default True [S4, unofficial].

**Outputs:**
- `path`: Array of shape $(n\_frames, 4, 4)$ containing the generated camera poses [S1, S2, S3, S4, unofficial].

**Data Structures:**
- `center`: Vector of shape $(3,)$ representing the focal point [S1, S2, S3, S4, unofficial].
- `offset`: Vector of shape $(3,)$ defining the center of the ellipse bounds [S1, S3, S4, unofficial].
- `sc`: Vector of shape $(3,)$ representing the semi-axis lengths (scaling factors) [S1, S2, S3, S4, unofficial].
- `low`, `high`: Vectors of shape $(3,)$ defining the bounding box of the ellipse [S1, S2, S3, S4, unofficial].
- `z_low`, `z_high`: Scalars defining the vertical bounds [S1, S3, S4, unofficial].
- `up`: Vector of shape $(3,)$ defining the camera up direction [S1, S2, S3, S4, unofficial].
- `positions`: Array of shape $(n\_frames, 3)$ for intermediate position calculation [S1, S2, S3, S4, unofficial].

**Steps:**

1.  **Scene Analysis**: Compute the focal point `center` using a function `focus_point_fn(poses)` [S1, S2, S3, S4, unofficial].
2.  **Offset and Scaling**:
    -   Determine the `offset` vector. For the primary X-Y plane variant, `offset = [center[0], center[1], 0]` [S1, S3, S4, unofficial].
    -   Compute scaling factors `sc` using the 90th percentile of the absolute distances of camera positions from `offset`:
        $$ \text{sc} = \text{percentile}(|\text{poses}[:, :3, 3] - \text{offset}|, 90, \text{axis}=0) $$
        [S1, S2, S3, S4, unofficial].
    -   Define ellipse bounds:
        $$ \text{low} = -\text{sc} + \text{offset} $$
        $$ \text{high} = \text{sc} + \text{offset} $$
        [S1, S2, S3, S4, unofficial].
3.  **Vertical Bounds**:
    -   Compute `z_low` and `z_high` using the 10th and 90th percentiles of the z-coordinates of the camera positions:
        $$ \text{z\_low} = \text{percentile}(\text{poses}[:, :3, 3][:, 2], 10, \text{axis}=0) $$
        $$ \text{z\_high} = \text{percentile}(\text{poses}[:, :3, 3][:, 2], 90, \text{axis}=0) $$
        [S1, S3, S4, unofficial].
4.  **Position Parameterization**: Define a function `get_positions(theta)` that returns the 3D position for a given angle $\theta \in [0, 2\pi]$ [S1, S3, S4, unofficial]:
    $$ x(\theta) = \text{low}[0] + (\text{high}[0] - \text{low}[0]) \cdot (\cos(\theta) \cdot 0.5 + 0.5) $$
    $$ y(\theta) = \text{low}[1] + (\text{high}[1] - \text{low}[1]) \cdot (\sin(\theta) \cdot 0.5 + 0.5) $$
    $$ z(\theta) = \text{z\_variation} \cdot (\text{z\_low}[2] + (\text{z\_high}[2] - \text{z\_low}[2]) \cdot (\cos(\theta + 2\pi \cdot \text{z\_phase}) \cdot 0.5 + 0.5)) $$
    [S1, S4, unofficial].
5.  **Sampling**:
    -   Generate $\theta$ values:
        $$ \theta = \text{linspace}(0, 2\pi, n\_frames + 1, \text{endpoint}=\text{True}) $$
        [S1, S3, S4, unofficial].
    -   Evaluate `get_positions(theta)` for all $\theta$ to get `positions` of shape $(n\_frames + 1, 3)$ [S1, S2, S3, S4, unofficial].
    -   Remove the last duplicate point (where $\theta = 2\pi$) to close the loop without redundancy:
        $$ \text{positions} = \text{positions}[0:n\_frames] $$
        [S1, S2, S3, S4, unofficial].
6.  **Constant Speed Resampling (Optional)**: If `const_speed` is true, calculate the arc lengths between consecutive positions and resample $\theta$ to ensure uniform spacing [S1, S4, unofficial].
    -   Calculate lengths:
        $$ \text{lengths}_i = \| \text{positions}_{i+1} - \text{positions}_i \|_2 $$
        [S1, S4, unofficial].
    -   Resample $\theta$ using cumulative arc length interpolation to maintain constant velocity [G].
7.  **Up Vector Determination**:
    -   Compute the average up vector from input poses:
        $$ \text{avg\_up} = \text{mean}(\text{poses}[:, :3, 1], \text{axis}=0) $$
        [S1, S2, S3, S4, unofficial].
    -   Normalize `avg_up` [S1, S2, S3, S4, unofficial].
    -   Find the index `ind_up` of the maximum absolute component of `avg_up` [S1, S2, S3, S4, unofficial].
    -   Set the up vector:
        $$ \text{up} = \mathbf{e}_{\text{ind\_up}} \cdot \text{sign}(\text{avg\_up}[\text{ind\_up}]) $$
        where $\mathbf{e}_{\text{ind\_up}}$ is the basis vector with 1 at `ind_up` and 0 elsewhere [S1, S2, S3, S4, unofficial].
8.  **Pose Construction**:
    -   For each position `p` in `positions`, construct a view matrix using `viewmatrix(p - center, up, p)` [S1, S2, S3, S4, unofficial].
    -   Stack all poses into the output `path` [S1, S2, S3, S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def focus_point_fn(poses):
    """Calculate the focal point (center) of the camera poses.
    Typically the mean of the camera positions."""
    return np.mean(poses[:, :3, 3], axis=0)

def viewmatrix(look_at, up, position):
    """Construct a view matrix from look-at direction, up vector, and position.
    look_at: direction the camera looks towards (from position).
    up: up vector.
    position: camera position.
    Returns a 4x4 matrix."""
    z = look_at / np.linalg.norm(look_at)
    x = np.cross(up, z)
    # Guard against parallel up and look_at [G]
    if np.linalg.norm(x) < 1e-6:
        x = np.array([1.0, 0.0, 0.0])
    x = x / np.linalg.norm(x)
    y = np.cross(z, x)
    
    R = np.column_stack((x, y, z))
    t = -R.T @ position
    
    P = np.eye(4)
    P[:3, :3] = R
    P[:3, 3] = t
    return P

def generate_ellipse_path(poses, n_frames, z_variation=0.0, z_phase=0.0, const_speed=True):
    """Generate an elliptical camera path."""
    
    # 1. Scene Analysis
    center = focus_point_fn(poses)
    
    # 2. Offset and Scaling
    offset = np.array([center[0], center[1], 0.0])
    sc = np.percentile(np.abs(poses[:, :3, 3] - offset), 90, axis=0)
    low = -sc + offset
    high = sc + offset
    
    # 3. Vertical Bounds
    z_low = np.percentile(poses[:, :3, 3][:, 2], 10, axis=0)
    z_high = np.percentile(poses[:, :3, 3][:, 2], 90, axis=0)
    
    # 4. Position Parameterization
    def get_positions(theta):
        x = low[0] + (high[0] - low[0]) * (np.cos(theta) * 0.5 + 0.5)
        y = low[1] + (high[1] - low[1]) * (np.sin(theta) * 0.5 + 0.5)
        z = z_variation * (z_low[2] + (z_high[2] - z_low[2]) * (np.cos(theta + 2 * np.pi * z_phase) * 0.5 + 0.5))
        return np.stack([x, y, z], axis=-1)
    
    # 5. Sampling
    theta = np.linspace(0, 2 * np.pi, n_frames + 1, endpoint=True)
    positions = get_positions(theta)
    positions = positions[:-1]
    
    # 6. Constant Speed Resampling (Optional)
    if const_speed:
        lengths = np.linalg.norm(positions[1:] - positions[:-1], axis=-1)
        cum_lengths = np.insert(np.cumsum(lengths), 0, 0.0)
        uniform_t = np.linspace(0, cum_lengths[-1], n_frames)
        theta = np.interp(uniform_t, cum_lengths, theta)
        positions = get_positions(theta)
        
    # 7. Up Vector Determination
    avg_up = np.mean(poses[:, :3, 1], axis=0)
    norm = np.linalg.norm(avg_up)
    # Guard against zero-length average up vector [G]
    if norm > 1e-6:
        avg_up = avg_up / norm
    ind_up = np.argmax(np.abs(avg_up))
    up = np.eye(3)[ind_up] * np.sign(avg_up[ind_up])
    
    # 8. Pose Construction
    path = np.stack([viewmatrix(p - center, up, p) for p in positions])
    return path
```

**Edge Cases and Pitfalls:**
-   **Degenerate Input**: If all input poses are collinear or identical, `sc` may be zero or `avg_up` may be undefined [S4, unofficial]. Guard against division by zero in normalization [G].
-   **Boundary Conditions**: The mapping from $\theta$ to position uses a shifted cosine/sine, ensuring the path starts and ends at the same point without discontinuity [S1, S2, S3, S4, unofficial].
-   **Axis Variation**: The coordinate axes can be swapped (e.g., X-Z plane with Y variation) depending on the specific variant implementation [S2, unofficial].

## Subtopics

### [Spiral Camera Path Generation](1635-spiral-camera-path-generation.md)
Generates a path with a continuously changing radius, suitable for capturing scenes with depth variation. Choose when the scene has significant depth and a circular orbit is insufficient.

### [Orbit Camera Path Generation](7811-orbit-camera-path-generation.md)
Generates a circular path around a center point. Choose when the scene is roughly spherical and a constant radius is appropriate.

## Operating characteristics
-   **Strengths**: Simple to implement, provides smooth and symmetric paths, allows for controlled height variation [S1, S2, S4, unofficial].
-   **Limitations**: Assumes the scene is roughly centered and bounded by the input poses [S1, S2, S4, unofficial].
-   **When to Use**: When a smooth, closed-loop camera path is needed for novel view synthesis or visualization, and the scene is well-contained within the input poses [S1, S2, S4, unofficial].
-   **When Not to Use**: When the scene is very large or unbounded, or when the camera needs to follow a specific non-elliptical trajectory [S1, S2, S4, unofficial].
-   **Time and Memory Complexity**: Not given in the sources [S1, S2, S3, S4, unofficial].
-   **Measured Performance**: Not given in the sources [S1, S2, S3, S4, unofficial].
-   **Scaling**: Not given in the sources [S1, S2, S3, S4, unofficial].

## Background
The method originates from the NeRF and multi-NeRF literature, specifically implemented in the `multinerf` repository by Google Research [S1, unofficial]. It has been adopted and adapted in various other neural rendering frameworks such as `mrGSplat`, `VolSplat`, and `VoLoDSplat` for generating training or evaluation camera paths [S2, S3, S4, unofficial].

## Sources
[S1] google-research/multinerf, https://github.com/google-research/multinerf, fetched 2026-10-26 (unofficial)
[S2] meshroomHub/mrGSplat, https://github.com/meshroomHub/mrGSplat, fetched 2026-10-27 (unofficial)
[S3] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S4] pulangk97/VoLoDSplat, https://github.com/pulangk97/VoLoDSplat, fetched 2026-10-27 (unofficial)
