# Random C-Arm View Sampling
## Overview
A method to generate random camera-to-world view matrices for C-arm X-ray projections by uniformly sampling geometric parameters (Source-to-Image Distance, AP angle, lateral angle, table SI translation) and converting them to camera coordinates [S1, unofficial].
## How it works
**Inputs:**
- `n_views`: Integer, number of views to generate [S1, unofficial].
- `sid_range`: Tuple of 2 floats `(min_sid, max_sid)` in mm [S1, unofficial].
- `ap_range`: Tuple of 2 floats `(min_ap, max_ap)` in degrees [S1, unofficial].
- `lat_range`: Tuple of 2 floats `(min_lat, max_lat)` in degrees [S1, unofficial].
- `si_range`: Tuple of 2 floats `(min_si, max_si)` in mm [S1, unofficial].
- `center`: Tuple or list of 3 floats representing the volume center in RAS coordinates (mm) [S1, unofficial].

**Outputs:**
- `view_matrices`: Tensor of shape `(n_views, 4, 4)` on CPU, containing the camera-to-world view matrices [S1, unofficial].

**Data Structures:**
- The output is a contiguous tensor where each `4x4` slice corresponds to one view matrix [S1, unofficial].

**Steps:**
1. Initialize an empty list to store the generated view matrices [S1, unofficial].
2. For each sample index $i$ from $0$ to $n\_views - 1$:
   a. Draw `sid` uniformly from `sid_range` [S1, unofficial].
   b. Draw `ap_angle` uniformly from `ap_range` [S1, unofficial].
   c. Draw `lat_angle` uniformly from `lat_range` [S1, unofficial].
   d. Draw `table_si` uniformly from `si_range` [S1, unofficial].
   e. Convert the sampled parameters to camera position, focal point, and up vector using `carm_to_camera_params(sid, ap_angle, lat_angle, center, table_si)` [S1, unofficial].
   f. Build a $4 \times 4$ camera-to-world view matrix using `get_vtk_view_mat(pos, focal, up)` [S1, unofficial].
   g. Append the matrix to the list [S1, unofficial].
3. Stack all matrices along a new leading dimension to form the final tensor of shape `(n_views, 4, 4)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch

def random_carm_view_sampling(n_views, sid_range, ap_range, lat_range, si_range, center):
    """Generate random C-arm view matrices."""
    view_matrices = []
    for _ in range(n_views):
        # 1. Sample parameters uniformly
        sid = np.random.uniform(sid_range[0], sid_range[1])
        ap_angle = np.random.uniform(ap_range[0], ap_range[1])
        lat_angle = np.random.uniform(lat_range[0], lat_range[1])
        table_si = np.random.uniform(si_range[0], si_range[1])
        
        # 2. Convert to camera parameters [G]
        pos, focal, up = carm_to_camera_params(sid, ap_angle, lat_angle, center, table_si)
        
        # 3. Build view matrix [G]
        view_mat = get_vtk_view_mat(pos, focal, up)
        view_matrices.append(view_mat)
        
    # Stack into tensor
    return torch.tensor(np.stack(view_matrices), dtype=torch.float32)

def carm_to_camera_params(sid, ap_angle, lat_angle, center, table_si):
    # [G] Standard C-arm geometry conversion.
    ap_rad = np.radians(ap_angle)
    lat_rad = np.radians(lat_angle)
    beam_dir = np.array([0.0, 0.0, -1.0])
    rot_ap = np.array([[1, 0, 0], [0, np.cos(ap_rad), -np.sin(ap_rad)], [0, np.sin(ap_rad), np.cos(ap_rad)]])
    beam_dir = rot_ap @ beam_dir
    rot_lat = np.array([[np.cos(lat_rad), 0, np.sin(lat_rad)], [0, 1, 0], [-np.sin(lat_rad), 0, np.cos(lat_rad)]])
    beam_dir = rot_lat @ beam_dir
    beam_dir = beam_dir / np.linalg.norm(beam_dir)
    table_si_vec = np.array([0.0, 0.0, table_si])
    pos = center - sid * beam_dir + table_si_vec
    focal = center + table_si_vec
    up = np.array([0.0, 0.0, 1.0])
    return pos, focal, up

def get_vtk_view_mat(pos, focal, up):
    # [G] Standard camera-to-world matrix construction.
    fwd = np.subtract(focal, pos)
    fwd_len = np.linalg.norm(fwd)
    if fwd_len == 0:
        raise ValueError("pos and focal must not be identical.")
    fwd = fwd / fwd_len
    right = np.cross(up, fwd)
    right_len = np.linalg.norm(right)
    if right_len == 0:
        right = np.array([1.0, 0.0, 0.0])
    else:
        right = right / right_len
    up = np.cross(fwd, right)
    mat = np.eye(4)
    mat[:3, 0] = right
    mat[:3, 1] = up
    mat[:3, 2] = fwd
    mat[:3, 3] = pos
    return mat
```

**Parameters:**
- `sid_range`: Typical range `(900, 1200)` mm [S1, unofficial].
- `ap_range`: Typical range `(-40, 40)` degrees for cranial/caudal tilt [S1, unofficial].
- `lat_range`: Typical range `(-90, 90)` degrees for LAO/RAO rotation [S1, unofficial].
- `si_range`: Typical range `(-100, 100)` mm for table translation [S1, unofficial].
- `center`: Volume center in RAS coordinates [S1, unofficial].

**Edge Cases and Pitfalls:**
- Precision: Ensure floating-point precision is sufficient for coordinate transformations [G].
- Boundaries: Ensure sampled angles and distances are within physically possible ranges for the C-arm [G].
- Degenerate Input: If `pos` equals `focal`, the view matrix is undefined; guard against this case [G].

## Operating characteristics
- **Time Complexity:** $O(n\_views)$, as each view is generated independently [G].
- **Memory Complexity:** $O(n\_views)$ to store the view matrices [G].
- **Performance:** Not given in the sources. [S1, unofficial]
- **Strengths:** Not given in the sources. [S1, unofficial]
- **Limitations:** Not given in the sources. [S1, unofficial]
- **When to Use:** Not given in the sources. [S1, unofficial]
- **When Not to Use:** Not given in the sources. [S1, unofficial]

## Background
The method is part of the `flashdrr` project, which provides fast DRR (Digitally Reconstructed Radiograph) rendering [S1, unofficial].

## Sources
[S1] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
