# RayBatch container

Also called: RayBatch

## Overview
The RayBatch container is a NumPy-style data structure for storing and sampling ray batches during training, supporting efficient indexing, slicing, and conversion to Mitsuba tensor types [S1, unofficial]. It provides batch-level operations and explicit conversion logic to Mitsuba 3, distinguishing it from the generic [Ray Structure](4026-ray-structure.md) and the [Ray Bundle](2179-ray-bundle.md) [G].

## How it works
**Inputs and Outputs:**
*   **Input:** NumPy arrays for ray origins, directions, and colors. Optional arrays for masks, sensor indices, and pixel coordinates [S1, unofficial].
*   **Output:** A `RayBatch` object supporting indexing/slicing and a method to convert contents into Mitsuba `mi.Ray3f` and `mi.Color3f` objects [S1, unofficial].

**Data Structures:**
The container holds the following fields, all sharing a common batch dimension $N$:
*   `origins`: `np.ndarray`, shape $(N, 3)$ [S1, unofficial].
*   `directions`: `np.ndarray`, shape $(N, 3)$, normalized [S1, unofficial].
*   `colors`: `np.ndarray`, shape $(N, 3)$ [S1, unofficial].
*   `masks`: `np.ndarray`, shape $(N, 1)$ or $(N,)$, optional [S1, unofficial].
*   `sensor_indices`: `np.ndarray`, shape $(N,)$, optional [S1, unofficial].
*   `pixel_coords`: `np.ndarray`, shape $(N, 2)$, optional [S1, unofficial].

**Numbered Steps:**
1.  **Initialization:** Construct the container by passing the NumPy arrays for origins, directions, and colors [S1, unofficial].
2.  **Validation:** Verify that `origins` and `directions` have shape $(N, 3)$, `colors` has shape $(N, 3)$, and `masks` (if present) have shape $(N, 1)$ or $(N,)$ [S1, unofficial].
3.  **Sampling:** To sample a subset of rays, generate random indices using `rng.choice(self.num_rays, size=num_rays, replace=False)` and index the internal arrays [S1, unofficial].
4.  **Conversion:** Convert the batch to Mitsuba types by mapping `origins` to `mi.Point3f`, `directions` to `mi.Vector3f`, and `colors` to `mi.Color3f` [S1, unofficial].

**Formulas:**
*   Ray count: $N = \text{origins.shape}[0]$ [S1, unofficial].

**Parameters:**
*   `num_rays`: Implicitly defined by the batch dimension $N$ [S1, unofficial].
*   `replace`: Boolean for sampling (default `False`) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Shape Mismatches:** If `origins`, `directions`, and `colors` do not share the same batch dimension $N$, assertion errors will be raised [S1, unofficial].
*   **Mask Reshaping:** Masks provided as 1D arrays $(N,)$ must be explicitly reshaped to $(N, 1)$ before assignment to avoid broadcasting issues [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from typing import Tuple, Optional
import mitsuba as mi

class RayBatch:
    def __init__(
        self,
        origins: np.ndarray,
        directions: np.ndarray,
        colors: np.ndarray,
        masks: Optional[np.ndarray] = None,
        sensor_indices: Optional[np.ndarray] = None,
        pixel_coords: Optional[np.ndarray] = None
    ):
        assert origins.shape == (origins.shape[0], 3), "Origins must be (N, 3)"
        assert directions.shape == (directions.shape[0], 3), "Directions must be (N, 3)"
        assert colors.shape == (colors.shape[0], 3), "Colors must be (N, 3)"
        assert origins.shape[0] == directions.shape[0] == colors.shape[0], "Batch dimensions must match"
        
        self.num_rays = origins.shape[0]
        self.origins = origins
        self.directions = directions
        self.colors = colors
        
        if masks is not None:
            if masks.ndim == 1:
                masks = masks.reshape(-1, 1)
            assert masks.shape[0] == self.num_rays, "Mask batch dimension must match"
        self.masks = masks
        
        self.sensor_indices = sensor_indices
        self.pixel_coords = pixel_coords

    def sample(self, num_rays: int, rng: np.random.Generator) -> 'RayBatch':
        indices = rng.choice(self.num_rays, size=num_rays, replace=False)
        
        new_origins = self.origins[indices]
        new_directions = self.directions[indices]
        new_colors = self.colors[indices]
        
        new_masks = self.masks[indices] if self.masks is not None else None
        new_sensor_indices = self.sensor_indices[indices] if self.sensor_indices is not None else None
        new_pixel_coords = self.pixel_coords[indices] if self.pixel_coords is not None else None
        
        return RayBatch(
            origins=new_origins,
            directions=new_directions,
            colors=new_colors,
            masks=new_masks,
            sensor_indices=new_sensor_indices,
            pixel_coords=new_pixel_coords
        )

    def to_mitsuba(self) -> Tuple[mi.Ray3f, mi.Color3f]:
        origins_mi = mi.TensorXf(self.origins)
        directions_mi = mi.TensorXf(self.directions)
        
        rays = mi.Ray3f(
            o=mi.Point3f(origins_mi[:, 0], origins_mi[:, 1], origins_mi[:, 2]),
            d=mi.Vector3f(directions_mi[:, 0], directions_mi[:, 1], directions_mi[:, 2])
        )
        
        colors_mi = mi.Color3f(
            mi.Float(self.colors[:, 0]),
            mi.Float(self.colors[:, 1]),
            mi.Float(self.colors[:, 2])
        )
        
        return rays, colors_mi
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] tlir, https://github.com/JackZhouSz/tlir, fetched 2026-10-27 (unofficial)
