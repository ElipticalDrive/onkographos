# Random C-Arm View Sampling


## Overview
Random C-Arm View Sampling generates random camera-to-world view matrices for C-arm X-ray projections by uniformly sampling geometric parameters (SID, AP angle, lateral angle, table SI translation) and converting them to camera coordinates [S1, unofficial].

## How it works
**Inputs:**
- `n_views`: Integer, number of views to generate.
- `sid_range`: Tuple `(min_sid, max_sid)` in mm.
- `ap_range`: Tuple `(min_ap, max_ap)` in degrees.
- `lat_range`: Tuple `(min_lat, max_lat)` in degrees.
- `si_range`: Tuple `(min_si, max_si)` in mm.
- `center`: Tuple `(cx, cy, cz)` in RAS coordinates (mm), representing the volume center [S1, unofficial].

**Outputs:**
- `view_matrices`: A tensor of shape `(n_views, 4, 4)` containing the camera-to-world view matrices, stored on the CPU [S1, unofficial].

**Data Structures:**
- The output is a tensor where each slice `view_matrices[i]` is a $4 \times 4$ matrix [S1, unofficial].

**Parameters:**
- `sid_range`: Typical range `(900, 1200)` [S1, unofficial].
- `ap_range`: Typical range `(-40, 40)` for cranial/caudal angles [S1, unofficial].
- `lat_range`: Typical range `(-90, 90)` for LAO/RAO angles [S1, unofficial].
- `si_range`: Typical range `(-100, 100)` for table translation [S1, unofficial].

**Steps:**
1. Initialize an output tensor of shape `(n_views, 4, 4)`.
2. For each sample index $i$ from $0$ to $n\_views - 1$:
   a. Draw `sid` $\sim U(\text{sid\_range}[0], \text{sid\_range}[1])$.
   b. Draw `ap_angle` $\sim U(\text{ap\_range}[0], \text{ap\_range}[1])$.
   c. Draw `lat_angle` $\sim U(\text{lat\_range}[0], \text{lat\_range}[1])$.
   d. Draw `table_si` $\sim U(\text{si\_range}[0], \text{si\_range}[1])$.
   e. Compute `pos`, `focal`, `up` via `carm_to_camera_params(sid, ap_angle, lat_angle, center, table_si)` [S1, unofficial].
   f. Compute $V_i = \text{get\_vtk\_view\_mat}(pos, focal, up)$ [S1, unofficial].
   g. Assign `view_matrices[i] = V_i`.
3. Return `view_matrices` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def random_carm_view_sampling(n_views, sid_range, ap_range, lat_range, si_range, center):
    # 1. Sample parameters uniformly
    sids = torch.rand(n_views) * (sid_range[1] - sid_range[0]) + sid_range[0]
    ap_angles = torch.rand(n_views) * (ap_range[1] - ap_range[0]) + ap_range[0]
    lat_angles = torch.rand(n_views) * (lat_range[1] - lat_range[0]) + lat_range[0]
    table_sis = torch.rand(n_views) * (si_range[1] - si_range[0]) + si_range[0]
    
    # 2. Convert to camera parameters
    # Exact kinematic formulas not provided in the sources [G].
    pos, focal, up = carm_to_camera_params(sids, ap_angles, lat_angles, center, table_sis)
    
    # 3. Build view matrices
    view_matrices = torch.zeros(n_views, 4, 4)
    for i in range(n_views):
        view_matrices[i] = get_vtk_view_mat(pos[i], focal[i], up[i])
    return view_matrices

def carm_to_camera_params(sid, ap_angle, lat_angle, center, table_si):
    # Exact kinematic formulas not provided in the sources [G].
    # Returns pos, focal, up tensors of shape (..., 3) in RAS coordinates.
    return torch.zeros_like(sid), torch.zeros_like(sid), torch.zeros_like(sid)

def get_vtk_view_mat(pos, focal, up):
    # Standard VTK view matrix construction [G]
    # Camera looks from pos to focal. VTK camera looks down -Z.
    z = (pos - focal) / torch.norm(pos - focal)
    x = torch.cross(up, z)
    x = x / torch.norm(x)
    y = torch.cross(z, x)
    R = torch.stack([x, y, z], dim=1)
    t = -R @ pos
    V = torch.eye(4)
    V[0:3, 0:3] = R
    V[0:3, 3] = t
    return V
```

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point errors in rotation matrices can accumulate; normalize vectors after cross products [G].
- **Degenerate Input:** If `pos` equals `focal`, the view matrix is undefined; add a small epsilon or check for zero distance [G].
- **Boundaries:** Not given in the sources.

## Subtopics

### [Low Discrepancy View Sampling](8328-low-discrepancy-view-sampling.md)
Uses low discrepancy sequences to distribute projection angles uniformly over time, ensuring all $N_p$ angles are unique to maximize information extraction while decomposing into linear scan rounds. Choose this for structured dynamic CT acquisition where static parts benefit from increasing frames.

## Operating characteristics
- **Time Complexity:** $O(n\_views)$ for sampling and matrix construction [G].
- **Memory Complexity:** $O(n\_views)$ to store the view matrices [G].
- **Performance:** Not given in the sources.
- **Strengths:** Simple to implement, provides diverse views for data augmentation [G].
- **Limitations:** Does not guarantee uniform coverage of angular space, which may lead to gaps in reconstruction if used for tomography [G].

## Background
Part of the `flashdrr` library ecosystem for synthetic DRR generation and C-arm simulation [S1, unofficial]. Builds on [C-Arm to Camera Parameters Conversion](2589-c-arm-to-camera-parameters-conversion.md) to translate geometric parameters into camera poses [S1, unofficial].

## Sources
[S1] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
