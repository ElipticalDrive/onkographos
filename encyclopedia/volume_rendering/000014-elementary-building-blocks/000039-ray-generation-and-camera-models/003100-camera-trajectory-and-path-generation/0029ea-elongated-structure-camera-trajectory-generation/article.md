# Elongated Structure Camera Trajectory Generation

Also called: Elongated-structure trajectory generator, PCA-based Camera Path Generation, PCA-based multi-view trajectory generation, Principal Axis Camera Trajectory

## Overview
This method generates multi-view camera poses for large-scale outdoor elongated structures (e.g., ancient city walls) from a 3D point cloud by fitting a principal axis via Principal Component Analysis (PCA), sampling stations along it, and validating positions against a density grid [S1, unofficial]. It replaces indoor room-centric hardcoded poses with data-driven trajectories that align with the structure's geometry [S1, unofficial]. Unlike [Spiral Camera Path Generation](1635-spiral-camera-path-generation.md) which assumes rotational symmetry, this method assumes linear elongation and places cameras on parallel tracks perpendicular to the principal axis [S1, unofficial].

## How it works
**Inputs:**
- `xyz`: A tensor of shape $[N, 3]$ containing 3D point cloud coordinates.
- `config`: A `TrajectoryConfig` object containing parameters (see Parameters).

**Outputs:**
- A list of `CameraPose` objects, each containing:
  - `extrinsics`: A $4 \times 4$ world-to-camera matrix.
  - `intrinsics`: A vector $[f_x, f_y, c_x, c_y]$.
  - `position`: $(x, y, z)$ camera center.
  - `target`: $(x, y, z)$ look-at target.
  - `pitch_deg`: The pitch angle used.

**Data Structures:**
- **Density Grid**: A 2D tensor of shape $[grid\_nx, grid\_ny]$. It is populated by bucketizing point XY coordinates into bins defined by `x_bins` and `y_bins`, then accumulating ones via `index_put_` with accumulation enabled [S1, unofficial].
- **CameraPose**: A dataclass holding the pose components [S1, unofficial].

**Parameters:**
- `num_stations`: int, default 10. Number of camera stations along the principal axis [S1, unofficial].
- `distance`: float, default 15.0. Primary distance from the wall surface in metres [S1, unofficial].
- `far_distance`: float, default 0.0. Optional farther distance for wide-coverage views. If set, generates an additional set of views [S1, unofficial].
- `pitch_angles_deg`: Tuple[float], default (-30.0, 0.0, 30.0). List of pitch angles in degrees (0 = horizontal, positive = camera looks downward) [S1, unofficial].
- `fov_y_deg`: float, default 60.0. Vertical field of view in degrees [S1, unofficial].
- `image_width`/`image_height`: int, default 640/480. Output image resolution in pixels [S1, unofficial].
- `view_both_sides`: bool, default True. Whether to place cameras on both sides of the wall [S1, unofficial].
- `margin_factor`: float, default 1.1. Extra margin beyond actual point cloud bounds (1.0 = exact bounds, 1.1 = 10% extra) [S1, unofficial].
- `yaw_offset_deg`: float, default 0.0. Additional yaw offset added to each station [S1, unofficial].

**Steps:**

1. **Compute OBB and Bounds:**
   Compute the 2D Oriented Bounding Box (OBB) in the XY plane via PCA on the point cloud's XY coordinates. Extract the centre, principal axes (sorted by eigenvalue descending), and actual projected min/max ranges along each axis [S1, unofficial].
   Determine the bounding box limits $(x_{min}, x_{max}, y_{min}, y_{max}, z_{min}, z_{max})$ and compute $z_{centre}$ as the mean of $z_{min}$ and $z_{max}$ [S1, unofficial].

2. **Compute Intrinsics:**
   Compute pinhole intrinsics vector $[f_x, f_y, c_x, c_y]$ using the vertical FOV and image dimensions [S1, unofficial].
   $$ f = \frac{height}{2.0 \cdot \tan(\frac{fov_y}{2.0})} $$
   $$ intr = [f, f, \frac{width}{2.0}, \frac{height}{2.0}] $$
   where $fov_y$ is in radians [S1, unofficial].

3. **Sample Stations:**
   Sample `num_stations` evenly along the principal axis. The parameter $t$ ranges from $extent_{min}$ to $extent_{max}$ (projected bounds scaled by `margin_factor`). Station XY is computed as $centre_{xy} + t \cdot main\_axis$, then clamped to the actual point cloud XY bounds $[x_{min}, x_{max}]$ and $[y_{min}, y_{max}]$ [S1, unofficial].

4. **Build Density Grid:**
   Build a 2D density grid over the XY plane with resolution `grid_res` = 5.0 metres per bin. Grid dimensions are calculated from the bounding box expanded by `grid_pad` = `max_dist` + 2.0 * `grid_res`. Points are bucketized into grid cells and accumulated [S1, unofficial].

5. **Generate Poses:**
   For each distance in `all_distances` (containing `distance` and optionally `far_distance`), each `side_sign` in $[1.0, -1.0]$ (or $[1.0]$ if `view_both_sides` is False), and each station $(sx, sy)$:
   - Compute camera position: $cam_{xy} = (sx, sy) + perp\_axis \cdot dist \cdot side\_sign$, and $cam_z = z_{centre}$ [S1, unofficial].
   - **Validation**: Skip the camera pose if no points exist within $radius = \max(15.0, dist \cdot 0.5)$ metres of the camera position, checked via the density grid [S1, unofficial].
   - For each pitch angle, compute target Z: $target_z = cam_z - \tan(pitch_{rad}) \cdot dist$. Construct the camera-to-target vector and build the $4 \times 4$ world-to-camera extrinsics matrix [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch
from typing import List, Tuple

def generate_trajectory(xyz: torch.Tensor, config) -> List[dict]:
    """
    Generate camera poses for elongated structures.
    xyz: [N, 3] tensor of point cloud coordinates.
    config: TrajectoryConfig object.
    Returns: List of CameraPose dicts.
    """
    device = xyz.device
    xyz = xyz.to(device)
    
    # 1. Compute OBB and Bounds
    # PCA on XY plane
    xy = xyz[:, :2]
    mean_xy = xy.mean(dim=0)
    cov = torch.cov(xy.T)
    eigenvalues, eigenvectors = torch.linalg.eigh(cov)
    # Sort descending
    eigenvalues = eigenvalues.flip(0)
    eigenvectors = eigenvectors.flip(1)
    axes = eigenvectors.T  # [2, 2], rows are axes
    main_axis = axes[0]
    perp_axis = axes[1]
    
    # Project to find ranges
    proj = xy @ axes.T
    proj_min = proj.min(dim=0).values
    proj_max = proj.max(dim=0).values
    
    # Global bounds
    x_min, x_max = float(xyz[:, 0].min()), float(xyz[:, 0].max())
    y_min, y_max = float(xyz[:, 1].min()), float(xyz[:, 1].max())
    z_min, z_max = float(xyz[:, 2].min()), float(xyz[:, 2].max())
    z_centre = (z_min + z_max) / 2.0
    
    # Consistent axis orientation
    if perp_axis[0] < 0:
        perp_axis = -perp_axis
        main_axis = -main_axis
        
    # 2. Intrinsics
    fov_y = math.radians(config.fov_y_deg)
    focal = config.image_height / (2.0 * math.tan(fov_y / 2.0))
    intrinsics = torch.tensor([focal, focal, config.image_width / 2.0, config.image_height / 2.0], device=device)
    
    # 3. Sample Stations
    margin = config.margin_factor
    extent_min = float(proj_min[0]) * margin
    extent_max = float(proj_max[0]) * margin
    
    stations_xy = []
    denom = max(config.num_stations - 1, 1) # Handle num_stations=1
    for i in range(config.num_stations):
        t = extent_min + (extent_max - extent_min) * i / denom
        on_wall = mean_xy + t * main_axis
        sx = float(torch.clamp(on_wall[0], x_min, x_max))
        sy = float(torch.clamp(on_wall[1], y_min, y_max))
        stations_xy.append((sx, sy))
        
    # 4. Build Density Grid
    all_distances = [config.distance]
    if config.far_distance > 0 and config.far_distance != config.distance:
        all_distances.append(config.far_distance)
    max_dist = max(all_distances)
    
    grid_res = 5.0
    grid_pad = max_dist + 2.0 * grid_res
    grid_nx = max(3, int((x_max - x_min + 2 * grid_pad) / grid_res))
    grid_ny = max(3, int((y_max - y_min + 2 * grid_pad) / grid_res))
    
    x_bins = torch.linspace(x_min - grid_pad, x_min - grid_pad + grid_nx * grid_res, grid_nx + 1, device=device)
    y_bins = torch.linspace(y_min - grid_pad, y_min - grid_pad + grid_ny * grid_res, grid_ny + 1, device=device)
    
    x_idx = torch.bucketize(xyz[:, 0].contiguous(), x_bins) - 1
    y_idx = torch.bucketize(xyz[:, 1].contiguous(), y_bins) - 1
    valid_xy = (x_idx >= 0) & (x_idx < grid_nx) & (y_idx >= 0) & (y_idx < grid_ny)
    x_idx, y_idx = x_idx[valid_xy], y_idx[valid_xy]
    
    density = torch.zeros(grid_nx, grid_ny, device=device)
    density.index_put_((x_idx, y_idx), torch.ones(x_idx.shape[0], device=device), accumulate=True)
    
    def has_points_near(cx: float, cy: float, radius: float) -> bool:
        # Check grid cells within radius
        # Map cx, cy to grid indices
        gx = int((cx - (x_min - grid_pad)) / grid_res)
        gy = int((cy - (y_min - grid_pad)) / grid_res)
        
        # Check surrounding cells
        r_cells = int(math.ceil(radius / grid_res))
        for dx in range(-r_cells, r_cells + 1):
            for dy in range(-r_cells, r_cells + 1):
                nx, ny = gx + dx, gy + dy
                if 0 <= nx < grid_nx and 0 <= ny < grid_ny:
                    if density[nx, ny] > 0:
                        return True
        return False

    poses = []
    sides = [1.0, -1.0] if config.view_both_sides else [1.0]
    
    for dist in all_distances:
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
                    
                    campos = torch.tensor([cam_x, cam_y, cam_z], device=device)
                    tgt_t = torch.tensor([cam_x, cam_y, target_z], device=device)
                    
                    extrinsics = _make_extrinsics(campos, tgt_t)
                    
                    poses.append({
                        "extrinsics": extrinsics,
                        "intrinsics": intrinsics,
                        "position": (cam_x, cam_y, cam_z),
                        "target": (cam_x, cam_y, target_z),
                        "pitch_deg": pitch_deg
                    })
    return poses

def _make_extrinsics(campos: torch.Tensor, target: torch.Tensor) -> torch.Tensor:
    """Build 4x4 world-to-camera matrix."""
    R_c2w = _look_at_cv(campos, target)
    R_w2c = R_c2w.T
    extr = torch.eye(4, dtype=torch.float32, device=campos.device)
    extr[:3, :3] = R_w2c
    extr[:3, 3] = -(R_w2c @ campos)
    return extr

def _look_at_cv(campos: torch.Tensor, target: torch.Tensor) -> torch.Tensor:
    """Build camera-to-world rotation matrix [3,3]."""
    up = torch.tensor([0.0, 0.0, 1.0], dtype=torch.float32, device=campos.device)
    forward = target - campos
    forward = forward / (forward.norm() + 1e-12)
    
    cos_angle = torch.abs(torch.dot(forward, up))
    if cos_angle > 0.999:
        # Singularity: forward parallel to up
        ref_axis = torch.tensor([1.0, 0.0, 0.0], dtype=torch.float32, device=campos.device)
    else:
        ref_axis = up
        
    right = torch.linalg.cross(ref_axis, forward)
    right = right / (right.norm() + 1e-12)
    up_corrected = torch.linalg.cross(forward, right)
    up_corrected = up_corrected / (up_corrected.norm() + 1e-12)
    
    R = torch.stack([right, up_corrected, forward], dim=1)
    return R
```

**Edge Cases and Pitfalls:**
- **Singularity**: When the forward vector is nearly parallel to the world up vector (e.g., overhead cameras looking straight down), the cross product for the right vector vanishes. This is handled by checking $cos\_angle = |dot(forward, up)| > 0.999$. If true, the reference axis is switched to world X $[1, 0, 0]$ to compute `right` and `up_corrected` via cross products [S1, unofficial].
- **Division by Zero**: When `num_stations` = 1, division by zero occurs in station interpolation. This is handled by using $\max(num\_stations - 1, 1)$ in the denominator [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Dominated by PCA ($O(N)$ for covariance, constant for $2 \times 2$) and grid population ($O(N)$). Pose generation is $O(S \cdot D \cdot P)$ where $S$ is stations, $D$ is distances, and $P$ is pitch angles. Density validation is $O(1)$ per pose with grid lookup.
- **Memory Complexity**: $O(N)$ for point cloud storage. Density grid is $O(grid\_nx \cdot grid\_ny)$, which scales with the bounding box size and `grid_res`.
- **Strengths**: Adapts automatically to the geometry of elongated structures; avoids invalid poses outside the point cloud via density validation; handles large-scale outdoor scenes better than hardcoded indoor poses.
- **Limitations**: Assumes linear elongation; may fail for complex 3D structures without a clear principal axis; density grid resolution is fixed (5.0m), which may be too coarse or fine depending on scene scale.
- **When to use**: Large-scale outdoor elongated structures (walls, roads, rivers) where a principal axis exists.
- **When not to use**: Indoor scenes, spherical objects, or structures without a dominant linear direction.

## Background
This method was introduced in the context of the Point-Splatting-Multi-View-2D-Rendering-Pipeline to generate multi-view camera poses for large-scale outdoor point clouds, specifically ancient city walls [S1, unofficial]. It replaces the indoor room-centric hardcoded poses used in Virtual-Multi-View-Fusion with a data-driven approach based on PCA [S1, unofficial].

## Sources
[S1] Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, https://github.com/Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, fetched 2026-10-28 (unofficial)
