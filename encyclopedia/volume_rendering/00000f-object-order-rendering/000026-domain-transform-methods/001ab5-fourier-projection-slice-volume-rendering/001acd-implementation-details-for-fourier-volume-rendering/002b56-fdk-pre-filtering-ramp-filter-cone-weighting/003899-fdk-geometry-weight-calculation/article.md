# FDK Geometry Weight Calculation

Also called: Cone Weighting, Feldkamp's cosine weight, Feldkamp-Davis-Kress (FDK) Geometry Weight Calculation

## Overview
Computes the geometric weighting factors required for the backprojection step of the Feldkamp-Davis-Kress (FDK) algorithm in explicit cone-beam geometry [S1, unofficial]. It solves the problem of correctly weighting each ray's contribution based on the angle between the ray and the detector panel normal, as well as the distance from the source to the detector [S1, unofficial]. It sets itself apart from parallel-beam FBP by accounting for the divergent cone-beam geometry through a `|dS · ν| / D'` weighting scheme, which simplifies to Feldkamp's cosine weight for circular orbits and upright panels [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:**
  - `sources`: Tensor of shape `[views, 3]`, source positions in 3D space.
  - `detectors`: Tensor of shape `[views, rows, columns, 3]`, detector pixel centers.
  - `center`: Tensor of shape `[3]`, center of rotation.
  - `rows`, `columns`: Integers, detector dimensions.
- **Outputs:**
  - `weights`: Tensor of shape `[views, rows, columns]`, the computed geometric weights.
  - `matrices`: Tensor of shape `[views, 12]`, projection matrices for mapping voxel coordinates to detector pixels.

**Data Structures:**
- `lattice`: Tensor of shape `[views, rows, columns, 3]`, representing the expected flat detector panel positions.
- `ViewColumn`: A helper structure (conceptual) containing the 12 floats defining the projection geometry for a single view, derived from `matrices`.

**Parameters:**
- `_FLATNESS`: Float `1e-2`, tolerance for panel flatness validation [S1, unofficial].
- `_FILTER_CHUNK`: Integer `1 << 25` (33,554,432), padded samples per filter call to bound FFT memory [S1, unofficial]. (Note: This parameter is used in the parent pre-filtering step, not directly in weight calculation, but is part of the context).

**Steps:**

1. **Validate Detector Panel:**
   If `rows < 2` or `columns < 2`, raise `ValueError` [S1, unofficial].

2. **Calculate Panel Geometry:**
   - `origin = detectors[:, 0, 0]` [S1, unofficial].
   - `row_step = (detectors[:, -1, 0] - origin) / (rows - 1)` [S1, unofficial].
   - `column_step = (detectors[:, 0, -1] - origin) / (columns - 1)` [S1, unofficial].
   - Construct lattice: `lattice = origin[:, None, None] + row * row_step[:, None, None] + column * column_step[:, None, None]` [S1, unofficial].
   - Calculate pitch: `pitch = torch.minimum(row_step.norm(dim=-1), column_step.norm(dim=-1)).min()` [S1, unofficial].

3. **Validate Flatness:**
   If `(detectors - lattice).norm(dim=-1).amax() > _FLATNESS * pitch`, raise `ValueError` [S1, unofficial].

4. **Calculate Source Steps:**
   - Pad sources: `before = 2 * sources[:1] - sources[1:2]`, `after = 2 * sources[-1:] - sources[-2:-1]` [S1, unofficial].
   - `padded = torch.cat([before, sources, after], dim=0)` [G].
   - `source_steps = (padded[2:] - padded[:-2]) / 2` [S1, unofficial].

5. **Calculate Redundancy Factor:**
   - `radial = (sources - center).detach().double()` [S1, unofficial].
   - `steps = F.cosine_similarity(radial[1:], radial[:-1], dim=-1).clamp(-1, 1).acos()` [S1, unofficial].
   - `turn = steps.sum() + steps.median()` [S1, unofficial].
   - `redundancy = 0.5` if `turn > 2 * math.pi - steps.median() / 2` else `1.0` [S1, unofficial].

6. **Determine Filter Axis:**
   - `along_columns = torch.sum(source_steps * column_step, dim=-1)` [G].
   - `along_rows = torch.sum(source_steps * row_step, dim=-1)` [G].
   - `axis = -1` if `along_columns >= along_rows` else `-2` [S1, unofficial].

7. **Calculate Panel Normal and Depth:**
   - `normal = _unit(torch.linalg.cross(column_step, row_step))` [S1, unofficial].
   - Adjust sign: if `torch.dot(normal, origin - sources) < 0`, `normal = -normal` [G].
   - `panel_depth = _dot(origin - sources, normal)` [S1, unofficial].

8. **Define Coordinate System:**
   - `filter_step = row_step` if `axis == -2` else `column_step` [G].
   - `along = _unit(filter_step)` [S1, unofficial].
   - `across = torch.linalg.cross(normal, along)` [S1, unofficial].
   - `principal = sources + panel_depth[:, None] * normal` [S1, unofficial].

9. **Compute Coordinates:**
   - Helper `coordinate(direction)`:
     - `diff = detectors - principal[:, None, None, None]` [G].
     - `coords = torch.sum(diff * direction[None, None, None, :], dim=-1)` [G].
   - `u = coordinate(along)` [S1, unofficial].
   - `v = coordinate(across)` [S1, unofficial].

10. **Calculate Weights:**
    - `depth = panel_depth` [G].
    - `line_depth = (depth.square() + v.square()).sqrt()` [S1, unofficial].
    - `motion_into_line = _dot(source_steps, normal)` [G].
    - `motion_into_line_normalized = motion_into_line / line_depth` [S1, unofficial].
    - `across_ray = _dot(source_steps, across)` [G].
    - `weights = redundancy * across_ray / (line_depth * pitch)` [S1, unofficial].

11. **Construct Projection Matrices:**
    - `gram = source_steps @ source_steps.transpose(1, 2)` [S1, unofficial].
    - Solve for `to_pixels` using `gram` and detector positions [S1, unofficial].
    - Calculate offset, inverse depth, and stack into `matrices` of shape `[views, 12]` [S1, unofficial].

**Formulas:**
- Backprojection weight for a voxel: $W = \frac{D^2}{l^2}$, where $l$ is the depth along the panel normal and $D$ is the panel's depth [S1, unofficial].
- For circular orbit and upright panel, this corresponds to Feldkamp's cosine weight: $W = \frac{1}{U^2}$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- Non-flat detector panels: Validated by `_FLATNESS` tolerance. If violated, raise `ValueError` [S1, unofficial].
- Degenerate detector dimensions: `rows < 2` or `columns < 2` raises `ValueError` [S1, unofficial].
- Precision: Use `double` for redundancy calculation to avoid numerical instability in cosine similarity [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math
import torch.nn.functional as F

def _unit(v):
    return v / v.norm(dim=-1, keepdim=True)

def _dot(a, b):
    return torch.sum(a * b, dim=-1)

def fdk_geometry_weight_calculation(sources, detectors, center, rows, columns, _FLATNESS=1e-2):
    """
    Computes FDK geometry weights and projection matrices.
    
    Args:
        sources: [views, 3]
        detectors: [views, rows, columns, 3]
        center: [3]
        rows: int
        columns: int
        _FLATNESS: float
        
    Returns:
        weights: [views, rows, columns]
        matrices: [views, 12]
    """
    if rows < 2 or columns < 2:
        raise ValueError("Detector dimensions must be at least 2x2")
        
    # Step 2: Panel Geometry
    origin = detectors[:, 0, 0]  # [views, 3]
    row_step = (detectors[:, -1, 0] - origin) / (rows - 1)  # [views, 3]
    column_step = (detectors[:, 0, -1] - origin) / (columns - 1)  # [views, 3]
    
    # Construct lattice
    row_indices = torch.arange(rows, device=sources.device).float()
    column_indices = torch.arange(columns, device=sources.device).float()
    row_grid, column_grid = torch.meshgrid(row_indices, column_indices, indexing='ij')
    row_grid = row_grid.unsqueeze(0).unsqueeze(-1).expand(sources.shape[0], -1, -1, -1)  # [views, rows, columns]
    column_grid = column_grid.unsqueeze(0).unsqueeze(-1).expand(sources.shape[0], -1, -1, -1)  # [views, rows, columns]
    
    lattice = origin[:, None, None, :] + row_grid[:, :, :, None] * row_step[:, None, None, :] + column_grid[:, :, :, None] * column_step[:, None, None, :]  # [views, rows, columns, 3]
    
    pitch = torch.minimum(row_step.norm(dim=-1), column_step.norm(dim=-1)).min()
    
    # Step 3: Flatness Validation
    flatness_error = (detectors - lattice).norm(dim=-1).amax()
    if flatness_error > _FLATNESS * pitch:
        raise ValueError("Detector panel is not flat enough")
        
    # Step 4: Source Steps
    before = 2 * sources[:1] - sources[1:2]
    after = 2 * sources[-1:] - sources[-2:-1]
    padded_sources = torch.cat([before, sources, after], dim=0)
    source_steps = (padded_sources[2:] - padded_sources[:-2]) / 2
    
    # Step 5: Redundancy Factor
    radial = (sources - center).detach().double()
    steps = F.cosine_similarity(radial[1:], radial[:-1], dim=-1).clamp(-1, 1).acos()
    turn = steps.sum() + steps.median()
    redundancy = 0.5 if turn > 2 * math.pi - steps.median() / 2 else 1.0
    
    # Step 6: Filter Axis
    along_columns = torch.sum(source_steps * column_step, dim=-1)
    along_rows = torch.sum(source_steps * row_step, dim=-1)
    axis = -1 if along_columns >= along_rows else -2
    
    # Step 7: Panel Normal and Depth
    normal = _unit(torch.linalg.cross(column_step, row_step))
    dot_product = _dot(normal, origin - sources)
    normal = torch.where(dot_product < 0, -normal, normal)
    panel_depth = _dot(origin - sources, normal)
    
    # Step 8: Coordinate System
    filter_step = row_step if axis == -2 else column_step
    along = _unit(filter_step)
    across = torch.linalg.cross(normal, along)
    principal = sources + panel_depth[:, None] * normal
    
    # Step 9: Coordinates
    diff = detectors - principal[:, None, None, :]  # [views, rows, columns, 3]
    u = torch.sum(diff * along[None, None, None, :], dim=-1)
    v = torch.sum(diff * across[None, None, None, :], dim=-1)
    
    # Step 10: Weights
    depth = panel_depth
    line_depth = (depth.square() + v.square()).sqrt()
    motion_into_line = _dot(source_steps, normal)
    motion_into_line_normalized = motion_into_line / line_depth
    across_ray = _dot(source_steps, across)
    weights = redundancy * across_ray / (line_depth * pitch)
    
    # Step 11: Projection Matrices
    # Simplified: Gram matrix and solving for projection parameters
    # This part is complex and depends on specific implementation details not fully provided in sources
    # For the purpose of this article, we assume a helper function exists
    matrices = _construct_projection_matrices(sources, detectors, rows, columns, axis)
    
    return weights, matrices

def _construct_projection_matrices(sources, detectors, rows, columns, axis):
    # Placeholder for the complex matrix construction logic
    # In a real implementation, this would involve solving linear systems
    # to map voxel coordinates to detector pixels
    views = sources.shape[0]
    matrices = torch.zeros(views, 12, device=sources.device)
    # ... detailed implementation ...
    return matrices
```

## Subtopics

### FDK Pre-Filtering (Ramp Filter + Cone Weighting)
Applies the ramp filter and cone weighting to projections before backprojection. Choose this when you need to pre-process projections for FDK backprojection. [FDK Pre-Filtering (Ramp Filter + Cone Weighting)](5010-fdk-pre-filtering-ramp-filter-cone-weighting.md)

### Fourier Projection-Slice Volume Rendering
Uses Fourier transform properties for volume rendering. Choose this for frequency-domain based rendering. [Fourier Projection-Slice Volume Rendering](52-fourier-projection-slice-volume-rendering.md)

## Operating characteristics
- **Time Complexity:** $O(V \cdot R \cdot C)$ for weight calculation, where $V$ is views, $R$ is rows, $C$ is columns. Backprojection is $O(V \cdot R \cdot C \cdot X \cdot Y \cdot Z)$ [S1, unofficial].
- **Memory Complexity:** $O(V \cdot R \cdot C)$ for weights and matrices [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear with number of views and detector resolution [S1, unofficial].
- **Strengths:** Accurate geometric weighting for cone-beam geometry, handles non-flat panels with validation [S1, unofficial].
- **Limitations:** Requires flat detector panel validation, may be computationally intensive for large volumes [S1, unofficial].
- **When to Use:** When using FDK algorithm for cone-beam CT reconstruction [S1, unofficial].
- **When Not to Use:** For parallel-beam geometry or when high accuracy is not required [S1, unofficial].

## Background
The FDK algorithm was introduced by Feldkamp, Davis, and Kress in 1984 [S1, unofficial]. It is a widely used method for cone-beam CT reconstruction, extending the FBP algorithm to 3D cone-beam geometry [S1, unofficial]. The geometry weight calculation is a crucial part of the FDK algorithm, ensuring accurate reconstruction by accounting for the divergent beam geometry [S1, unofficial].

## Sources
[S1] itu-biai/torchtomo, https://github.com/itu-biai/torchtomo, fetched 2026-10-27 (unofficial)
