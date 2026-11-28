# Ray Sample Generation

Also called: Frustum Generation

## Overview
Ray Sample Generation constructs `RaySamples` from a `RayBundle` and bin boundaries, projecting points along ray directions to create frustums for volume rendering [S1, unofficial]. It discretizes the ray path into segments with associated geometric properties, unlike [Camera Ray Generation](7456-camera-ray-generation.md) which defines origins and directions [G].

## How it works
**Inputs and Outputs**
*   **Input:** A `RayBundle` containing ray origins, directions, and camera parameters; bin boundaries defining the sampling intervals along the ray [S1, unofficial].
*   **Output:** A `RaySamples` object containing the geometric representation of the sampled frustums [S1, unofficial].

**Data Structures**
The `RaySamples` structure contains the following fields [S1, unofficial]:
*   `frustums`: The geometric representation of the frustums.
*   `camera_indices`: Integer tensor of shape `(*bs, 1)` indicating the camera index for each sample.
*   `deltas`: Float tensor of shape `(*bs, 1)` representing the distance between consecutive sample points.
*   `spacing_starts`: Float tensor of shape `(*bs, num_samples, 1)` indicating the start distance of each interval.
*   `spacing_ends`: Float tensor of shape `(*bs, num_samples, 1)` indicating the end distance of each interval.
*   `spacing_to_euclidean_fn`: A callable function to convert spacing to Euclidean distance.
*   `metadata`: A dictionary for additional context.
*   `times`: Float tensor of shape `(*batch, 1)` for temporal information.

**Steps**
1.  **Extract Ray Parameters:** Retrieve ray origins $\mathbf{o}$ and directions $\mathbf{d}$ from the `RayBundle` [S1, unofficial].
2.  **Compute Interval Boundaries:** Derive `spacing_starts` and `spacing_ends` from the provided bin boundaries [S1, unofficial].
3.  **Calculate Deltas:** Compute the delta $\Delta^{(i)}$ for each interval from the spacing boundaries [S1, unofficial].
4.  **Generate Frustums:** Project points along ray directions to create frustums for volume rendering [S1, unofficial].
5.  **Assemble RaySamples:** Populate the `RaySamples` object with the computed frustums, deltas, spacing starts/ends, and metadata [S1, unofficial].

**Formulas**
*   Interval Delta: $\Delta^{(i)} = t_{end}^{(i)} - t_{start}^{(i)}$ [S1, unofficial].
*   Frustum Geometry: The frustum is defined by the ray direction $\mathbf{d}$, origin $\mathbf{o}$, and the projected radii at $t_{start}^{(i)}$ and $t_{end}^{(i)}$ [S1, unofficial].

**Parameters**
*   `num_samples`: The number of intervals per ray [S1, unofficial].
*   `bin_boundaries`: The specific distances along the ray where samples are taken [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Precision:** Floating-point precision errors can accumulate when computing large distances or small deltas [G].
*   **Boundaries:** Ensure that bin boundaries are strictly increasing and within the valid viewing range [G].
*   **Degenerate Input:** Handle cases where ray directions are zero or origins are undefined [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from dataclasses import dataclass
import torch

@dataclass
class Frustum:
    origins: torch.Tensor
    directions: torch.Tensor
    radii: torch.Tensor

@dataclass
class RaySamples:
    frustums: Frustum
    camera_indices: torch.Tensor
    deltas: torch.Tensor
    spacing_starts: torch.Tensor
    spacing_ends: torch.Tensor
    spacing_to_euclidean_fn: callable
    metadata: dict
    times: torch.Tensor

def generate_ray_samples(ray_bundle, bin_boundaries):
    """
    Generates RaySamples from a RayBundle and bin boundaries.
    Projects points along ray directions to create frustums for volume rendering [S1, unofficial].
    """
    o = ray_bundle.origins  # Shape: (*batch, 3)
    d = ray_bundle.directions  # Shape: (*batch, 3)
    
    # Derive spacing boundaries from input bin boundaries [S1, unofficial]
    spacing_starts = bin_boundaries[..., :-1]  # Shape: (*batch, num_samples, 1)
    spacing_ends = bin_boundaries[..., 1:]     # Shape: (*batch, num_samples, 1)
    
    # Compute deltas from spacing boundaries [S1, unofficial]
    deltas = spacing_ends - spacing_starts     # Shape: (*batch, num_samples, 1)
    
    # Project points along ray directions to create frustums [S1, unofficial]
    # Frustum geometry is defined by origin, direction, and interval boundaries.
    frustums = Frustum(
        origins=o.unsqueeze(-2),
        directions=d.unsqueeze(-2),
        radii=spacing_starts  # Radius computation depends on camera intrinsics [G]
    )
    
    # Assemble RaySamples with all required fields [S1, unofficial]
    ray_samples = RaySamples(
        frustums=frustums,
        camera_indices=ray_bundle.camera_indices,
        deltas=deltas,
        spacing_starts=spacing_starts,
        spacing_ends=spacing_ends,
        spacing_to_euclidean_fn=ray_bundle.spacing_to_euclidean_fn,
        metadata=ray_bundle.metadata,
        times=ray_bundle.times
    )
    
    return ray_samples
```

## Subtopics

### Ray Featurization with Gaussians
Approximates a frustum as a Gaussian distribution (mean + covariance) to enable efficient featurization of ray sections [S2, unofficial]. It assumes the ray originates from the origin and casts cone- or cylinder-shaped rays [S2, unofficial]. Choose this variant when using Gaussian-based rendering or when smooth, differentiable approximations of ray segments are required [S2, unofficial]. It sets itself apart by converting distance intervals into Gaussian parameters rather than keeping them as geometric frustums [S2, unofficial].

**Steps:**
1.  Extract interval boundaries $t_0$ and $t_1$ from the ray samples [S2, unofficial].
2.  Compute the mean distance $t_{\text{mean}} = (t_0 + t_1) / 2$ [S2, unofficial].
3.  Compute the variance of the distance $t_{\text{var}} = (t_1 - t_0)^2 / 12$ [S2, unofficial].
4.  Compute the radius variance $r_{\text{var}} = \text{radius}^2 / 4$ [S2, unofficial].
5.  Select the Gaussian conversion function based on `ray_shape`: 'cone' uses `conical_frustum_to_gaussian`, 'cylinder' uses `cylinder_to_gaussian` [S2, unofficial].
6.  Call the selected function with directions, $t_0$, $t_1$, radii, and variances to get means and covariances [S2, unofficial].
7.  Add the ray origins to the means: $\text{means} = \text{means} + \text{origins}[\dots, \text{None}, :]$ [S2, unofficial].
8.  Return the means and covariances [S2, unofficial].

**Formulas:**
*   $t_{\text{mean}} = \frac{t_0 + t_1}{2}$ [S2, unofficial].
*   $t_{\text{var}} = \frac{(t_1 - t_0)^2}{12}$ [S2, unofficial].
*   $r_{\text{var}} = \frac{\text{radius}^2}{4}$ [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear in the number of samples $N$ and batch size $B$, i.e., $O(B \cdot N)$ [G].
*   **Memory Complexity:** Linear in $B \cdot N$ for storing frustum parameters and deltas [G].
*   **Strengths:** Provides a structured representation of ray segments suitable for volume rendering and integration [S1, unofficial].
*   **Limitations:** Memory usage scales with the number of samples [G]. Precision issues may arise with very small or large distances [G].
*   **When to Use:** When volume rendering requires explicit discretization of rays into segments [S1, unofficial].
*   **When Not to Use:** When continuous ray integration is preferred or when memory is constrained [G].

## Background
Not given in the sources.

## Sources
[S1] Gaussian-Splatting-Toolkit/Gaussian-Splatting-Toolkit, https://github.com/Gaussian-Splatting-Toolkit/Gaussian-Splatting-Toolkit, fetched 2026-10-27 (unofficial)
[S2] https://github.com/CPREgroup/CaSE-NeRF, https://github.com/CPREgroup/CaSE-NeRF, fetched 2026-10-30 (unofficial)
