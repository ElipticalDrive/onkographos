# Elongated Structure Camera Trajectory Generation

Also called: Elongated-structure trajectory generator, PCA-based Camera Path Generation, PCA-based multi-view trajectory generation, Principal Axis Camera Trajectory

## Overview
This method generates multi-view camera poses for large-scale outdoor elongated structures (e.g., ancient city walls) from a 3D point cloud by fitting a principal axis via Principal Component Analysis (PCA), sampling stations along it, and validating positions against a density grid [S1, unofficial]. It replaces indoor room-centric hardcoded poses with data-driven trajectories aligned to the structure's geometry [S1, unofficial]. It differs from [Spiral Camera Path Generation](1635-spiral-camera-path-generation.md) by following a linear principal axis rather than a helical path, and from [Orbit Camera Path Generation](7811-orbit-camera-path-generation.md) by maintaining a fixed distance from a linear centerline rather than orbiting a central point.

## How it works
**Inputs:**
- `xyz`: A tensor of shape $[N, 3]$ containing 3D point cloud coordinates.
- `config`: A configuration object (see `TrajectoryConfig` below).

**Outputs:**
- A list of `CameraPose` objects, each containing:
  - `extrinsics`: A $4 \times 4$ world-to-camera matrix.
  - `intrinsics`: A vector $[f_x, f_y, c_x, c_y]$.
  - `position`: Camera center $(x, y, z)$ in world space.
  - `target`: Look-at target in world space.
  - `pitch_deg`: The pitch angle used.

**Data Structures:**
- `TrajectoryConfig`:
  - `num_stations`: `int`, default 10.
  - `distance`: `float`, default 15.0 (meters).
  - `far_distance`: `float`, default 0.0 (meters). If $>0$, adds a second track.
  - `pitch_angles_deg`: `Tuple[float, ...]`, default $(-30.0, 0.0, 30.0)$.
  - `yaw_offset_deg`: `float`, default 0.0.
  - `fov_y_deg`: `float`, default 60.0.
  - `image_width`: `int`, default 640.
  - `image_height`: `int`, default 480.
  - `view_both_sides`: `bool`, default `True`.
  - `margin_factor`: `float`, default 1.1.
- Density Grid: A 2D tensor of shape $[grid\_nx, grid\_ny]$ storing point counts per bin.

**Steps:**

1. **Compute OBB and Bounds:**
   - Extract XY coordinates from `xyz`.
   - Compute the covariance matrix of the XY points.
   - Perform Eigenvalue Decomposition on the covariance matrix. Sort eigenvalues and eigenvectors in descending order.
   - Let $\mathbf{e}_0$ be the eigenvector with the largest eigenvalue (principal axis). Let $\mathbf{e}_1$ be the second.
   - Project all XY points onto $\mathbf{e}_0$ and $\mathbf{e}_1$ to find projected min/max ranges.
   - Compute the center of the point cloud in XY: $\mathbf{c}_{xy} = \text{mean}(xy)$.
   - Compute global bounding box limits: $x_{min}, x_{max}, y_{min}, y_{max}, z_{min}, z_{max}$.
   - Compute $z_{centre} = (z_{min} + z_{max}) / 2$.
   - Ensure consistent axis orientation: if $\mathbf{e}_1[0] < 0$, negate both $\mathbf{e}_0$ and $\mathbf{e}_1$.
   - Set $\mathbf{main\_axis} = \mathbf{e}_0$ and $\mathbf{perp\_axis} = \mathbf{e}_1$.

2. **Compute Intrinsics:**
   - Convert `fov_y_deg` to radians: $\text{fov}_y = \text{fov\_y\_deg} \times \pi / 180$.
   - Compute focal length: $f = \frac{\text{height}}{2 \tan(\text{fov}_y / 2)}$.
   - Set intrinsics vector: $[f, f, \text{width}/2, \text{height}/2]$.

3. **Sample Stations:**
   - Calculate extent bounds along the principal axis:
     - $\text{extent}_{min} = \text{proj}_{min}[0] \times \text{margin\_factor}$
     - $\text{extent}_{max} = \text{proj}_{max}[0] \times \text{margin\_factor}$
   - For $i$ from 0 to `num_stations` - 1:
     - Compute parameter $t = \text{extent}_{min} + (\text{extent}_{max} - \text{extent}_{min}) \times \frac{i}{\max(\text{num\_stations} - 1, 1)}$.
     - Compute point on axis: $\mathbf{p}_{axis} = \mathbf{c}_{xy} + t \times \mathbf{main\_axis}$.
     - Clamp $p_{axis}[0]$ to $[x_{min}, x_{max}]$ and $p_{axis}[1]$ to $[y_{min}, y_{max}]$.
     - Store $(\text{clamped}_x, \text{clamped}_y)$ as a station.

4. **Build Density Grid:**
   - Determine `max_dist` = max(`distance`, `far_distance` if `far_distance` > 0 else 0).
   - Set grid resolution `grid_res` = 5.0 meters.
   - Set padding `grid_pad` = `max_dist` + $2 \times$ `grid_res`.
   - Calculate grid dimensions:
     - $grid\_nx = \max(3, \lfloor (x_{max} - x_{min} + 2 \times grid\_pad) / grid\_res \rfloor)$
     - $grid\_ny = \max(3, \lfloor (y_{max} - y_{min} + 2 \times grid\_pad) / grid\_res \rfloor)$
   - Define bin edges:
     - $x\_bins$ linearly spaced from $x_{min} - grid\_pad$ to $x_{min} - grid\_pad + grid\_nx \times grid\_res$.
     - $y\_bins$ linearly spaced from $y_{min} - grid\_pad$ to $y_{min} - grid\_pad + grid\_ny \times grid\_res$.
   - Map points to grid indices using bucketization.
   - Initialize `density` tensor of shape $[grid\_nx, grid\_ny]$ to zeros.
   - Accumulate counts: for each valid point index $(x_i, y_i)$, increment `density[x_i, y_i]`.

5. **Generate Camera Poses:**
   - Define `distances` list: `[distance]`. If `far_distance` > 0 and `far_distance` != `distance`, append `far_distance`.
   - Define `sides` list: `[1.0, -1.0]` if `view_both_sides` is True, else `[1.0]`.
   - For each `dist` in `distances`:
     - For each `side_sign` in `sides`:
       - For each station $(sx, sy)$:
         - Compute camera XY:
           - $cam\_x = sx + \mathbf{perp\_axis}[0] \times dist \times side\_sign$
           - $cam\_y = sy + \mathbf{perp\_axis}[1] \times dist \times side\_sign$
         - Set $cam\_z = z_{centre}$.
         - Compute radius for validation: $r = \max(15.0, dist \times 0.5)$.
         - Check if any points exist within radius $r$ of $(cam\_x, cam\_y)$ using the density grid. If not, skip this pose.
         - For each `pitch_deg` in `pitch_angles_deg`:
           - Convert `pitch_deg` to radians: $\text{pitch\_rad} = \text{pitch\_deg} \times \pi / 180$.
           - Compute target Z: $target\_z = cam\_z - \tan(\text{pitch\_rad}) \times dist$.
           - Compute target XY: The target lies along the camera's forward vector at distance `dist`. Since the camera is perpendicular to the wall axis (yaw offset 0), the target XY is the station XY $(sx, sy)$.
           - Construct camera-to-world rotation matrix $\mathbf{R}_{c2w}$ using `_look_at_cv(campos, target)`:
             - $\mathbf{forward} = \text{target} - \text{campos}$, normalized.
             - Handle singularity: if $|\mathbf{forward} \cdot \mathbf{up}| > 0.999$ (where $\mathbf{up}=[0,0,1]$), set reference axis to $\mathbf{x}=[1,0,0]$. Else, use $\mathbf{up}$.
             - $\mathbf{right} = \text{reference} \times \mathbf{forward}$, normalized.
             - $\mathbf{up}_{corrected} = \mathbf{forward} \times \mathbf{right}$, normalized.
             - $\mathbf{R}_{c2w} = [\mathbf{right}, \mathbf{up}_{corrected}, \mathbf{forward}]$.
           - Compute world-to-camera extrinsics $\mathbf{E}$:
             - $\mathbf{R}_{w2c} = \mathbf{R}_{c2w}^T$.
             - $\mathbf{t} = -\mathbf{R}_{w2c} \times \text{campos}$.
             - $\mathbf{E} = \begin{bmatrix} \mathbf{R}_{w2c} & \mathbf{t} \\ 0 & 1 \end{bmatrix}$.
           - Create `CameraPose` with `extrinsics`=$\mathbf{E}$, `intrinsics`, `position`=$(cam\_x, cam\_y, cam\_z)$, `target`=$(sx, sy, target\_z)$, and `pitch_deg`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch
from typing import List, Tuple, Optional

def generate_trajectory(xyz: torch.Tensor, config: 'TrajectoryConfig') -> List['CameraPose']:
    """
    Generate camera poses for elongated structures.
    """
    device = xyz.device
    
    # 1. Compute OBB and Bounds
    xy = xyz[:, :2]
    centre_xy = xy.mean(dim=0)
    cov = torch.cov(xy.T)
    eigenvalues, eigenvectors = torch.linalg.eigh(cov)
    # Sort descending
    eigenvalues = eigenvalues.flip(0)
    eigenvectors = eigenvectors.flip(1)
    
    main_axis = eigenvectors[:, 0] # [2]
    perp_axis = eigenvectors[:, 1] # [2]
    
    # Consistent orientation
    if perp_axis[0] < 0:
        perp_axis = -perp_axis
        main_axis = -main_axis
        
    x_min, x_max = float(xyz[:, 0].min()), float(xyz[:, 0].max())
    y_min, y_max = float(xyz[:, 1].min()), float(xyz[:, 1].max())
    z_min, z_max = float(xyz[:, 2].min()), float(xyz[:, 2].max())
    z_centre = (z_min + z_max) / 2.0
    
    # Project to find extents
    proj = xy @ main_axis
    proj_min = float(proj.min())
    proj_max = float(proj.max())
    
    # 2. Intrinsics
    fov_y = math.radians(config.fov_y_deg)
    focal = config.image_height / (2.0 * math.tan(fov_y / 2.0))
    intrinsics = torch.tensor([focal, focal, config.image_width / 2.0, config.image_height / 2.0], device=device)
    
    # 3. Stations
    margin = config.margin_factor
    extent_min = proj_min * margin
    extent_max = proj_max * margin
    
    stations_xy = []
    denom = max(config.num_stations - 1, 1)
    for i in range(config.num_stations):
        t = extent_min + (extent_max - extent_min) * i / denom
        on_wall = centre_xy + t * main_axis
        sx = float(torch.clamp(on_wall[0], x_min, x_max))
        sy = float(torch.clamp(on_wall[1], y_min, y_max))
        stations_xy.append((sx, sy))
        
    # 4. Density Grid
    all_dists = [config.distance]
    if config.far_distance > 0 and config.far_distance != config.distance:
        all_dists.append(config.far_distance)
    max_dist = max(all_dists)
    
    grid_res = 5.0
    grid_pad = max_dist + 2.0 * grid_res
    grid_nx = max(3, int((x_max - x_min + 2 * grid_pad) / grid_res))
    grid_ny = max(3, int((y_max - y_min + 2 * grid_pad) / grid_res))
    
    x_bins = torch.linspace(x_min - grid_pad, x_min - grid_pad + grid_nx * grid_res, grid_nx + 1, device=device)
    y_bins = torch.linspace(y_min - grid_pad, y_min - grid_pad + grid_ny * grid_res, grid_ny + 1, device=device)
    
    x_idx = torch.bucketize(xyz[:, 0], x_bins) - 1
    y_idx = torch.bucketize(xyz[:, 1], y_bins) - 1
    
    valid = (x_idx >= 0) & (x_idx < grid_nx) & (y_idx >= 0) & (y_idx < grid_ny)
    x_idx, y_idx = x_idx[valid], y_idx[valid]
    
    density = torch.zeros(grid_nx, grid_ny, device=device)
    density.index_put_((x_idx, y_idx), torch.ones(x_idx.shape[0], device=device), accumulate=True)
    
    # Helper to check density
    def has_points_near(cx: float, cy: float, radius: float) -> bool:
        # Map cx, cy to grid indices
        gx = int((cx - (x_min - grid_pad)) / grid_res)
        gy = int((cy - (y_min - grid_pad)) / grid_res)
        
        # Check a window around the center bin
        # Simplified: check if any bin in radius has points
        # For efficiency in implementation, one might query the grid directly.
        # Here we approximate by checking the center bin and neighbors if needed.
        # The source code uses a specific check, but logically it checks if density > 0 in range.
        # Given the grid resolution is 5m, and radius can be small, we check the bin containing the point.
        # If the point is on the edge, we might miss neighbors, but the source implies a simple check.
        # Let's implement a robust check:
        min_gx = max(0, gx - int(radius / grid_res) - 1)
        max_gx = min(grid_nx - 1, gx + int(radius / grid_res) + 1)
        min_gy = max(0, gy - int(radius / grid_res) - 1)
        max_gy = min(grid_ny - 1, gy + int(radius / grid_res) + 1)
        
        return density[min_gx:max_gx+1, min_gy:max_gy+1].sum() > 0

    poses = []
    sides = [1.0, -1.0] if config.view_both_sides else [1.0]
    
    for dist in all_dists:
        for side_sign in sides:
            for sx, sy in stations_xy:
                cam_x = sx + float(perp_axis[0]) * dist * side_sign
                cam_y = sy + float(perp_axis[1]) * dist * side_sign
                cam_z = z_centre
                
                radius = max(15.0, dist * 0.5)
                if not has_points_near(cam_x, cam_y, radius):
                    continue
                
                for pitch_deg in config.pitch_angles_deg:
                    pitch_rad = math.radians(pitch_deg)
                    target_z = cam_z - math.tan(pitch_rad) * dist
                    target_x, target_y = sx, sy
                    
                    campos = torch.tensor([cam_x, cam_y, cam_z], device=device)
                    tgt = torch.tensor([target_x, target_y, target_z], device=device)
                    
                    extrinsics = _make_extrinsics(campos, tgt)
                    
                    poses.append(CameraPose(
                        extrinsics=extrinsics,
                        intrinsics=intrinsics,
                        position=(cam_x, cam_y, cam_z),
                        target=(target_x, target_y, target_z),
                        pitch_deg=pitch_deg
                    ))
                    
    return poses

def _make_extrinsics(campos: torch.Tensor, target: torch.Tensor) -> torch.Tensor:
    up = torch.tensor([0.0, 0.0, 1.0], dtype=campos.dtype, device=campos.device)
    forward = target - campos
    forward = forward / (forward.norm() + 1e-12)
    
    cos_angle = torch.abs(torch.dot(forward, up))
    if cos_angle > 0.999:
        ref_axis = torch.tensor([1.0, 0.0, 0.0], dtype=campos.dtype, device=campos.device)
    else:
        ref_axis = up
        
    right = torch.cross(ref_axis, forward)
    right = right / (right.norm() + 1e-12)
    up_corrected = torch.cross(forward, right)
    up_corrected = up_corrected / (up_corrected.norm() + 1e-12)
    
    R_c2w = torch.stack([right, up_corrected, forward], dim=1)
    R_w2c = R_c2w.T
    extr = torch.eye(4, dtype=campos.dtype, device=campos.device)
    extr[:3, :3] = R_w2c
    extr[:3, 3] = -(R_w2c @ campos)
    return extr
```

**Edge Cases and Pitfalls:**
- **Singularity:** When the forward vector is nearly parallel to the world up vector (e.g., overhead cameras), the cross product for the right vector vanishes. This is handled by checking $|\mathbf{forward} \cdot \mathbf{up}| > 0.999$ and switching the reference axis to world X $[1, 0, 0]$ [S1, unofficial].
- **Division by Zero:** If `num_stations` is 1, the denominator in station interpolation becomes 0. This is handled by using $\max(\text{num\_stations} - 1, 1)$ [S1, unofficial].
- **Boundary Clamping:** Stations are clamped to the actual point cloud bounding box to ensure they remain within the data extent [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Dominated by PCA ($O(N)$ for covariance, constant for 2x2) and grid bucketization ($O(N)$). Pose generation is $O(S \times P)$ where $S$ is stations and $P$ is pitch angles.
- **Memory Complexity:** $O(N)$ for point cloud storage, $O(grid\_nx \times grid\_ny)$ for the density grid.
- **Strengths:** Adapts automatically to the structure's orientation; handles large-scale outdoor scenes; avoids invalid poses outside the point cloud.
- **Limitations:** Assumes the structure is elongated in one primary direction; less suitable for compact or spherical objects.
- **When to use:** Large-scale outdoor elongated structures (walls, roads, rivers) where a linear trajectory is appropriate.
- **When not to use:** Indoor rooms, spherical objects, or structures with complex 3D curvature not captured by a single principal axis.

## Background
This method was introduced in the context of the Point-Splatting Multi-View 2D Rendering Pipeline for rendering large-scale outdoor point clouds, specifically ancient city walls [S1, unofficial]. It replaces the hardcoded, room-centric camera poses used in earlier methods like Virtual-Multi-View-Fusion with a data-driven approach based on Principal Component Analysis [S1, unofficial].

## Sources
[S1] Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, https://github.com/Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, fetched 2026-10-28 (unofficial)
