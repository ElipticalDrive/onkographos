# Fixed-Resolution Surface Rendering with View Bounds

Also called: 2 mm however long rendering, Surface Rendering with Adaptive Voxel Sampling and View Bounds, qc_surfaces surface rendering

## Overview
This method renders medical volumes (CT, MR, PET, RT Dose) as surfaces at a fixed spatial resolution of 2 mm, regardless of the volume's length, provided it stays within strict physical bounds [S1, unofficial]. It solves the problem of coarse sampling in long volumes (e.g., whole-body scans), where previous methods limited the voxel count per side to 256, resulting in resolutions as coarse as ~7 mm for a 1,800 mm long scan [S1, unofficial]. This method enforces a uniform 2 mm grid for visual consistency in medical review, specifically ensuring facial features are rendered with the same fidelity in whole-body scans as in head scans [S1, unofficial].

## How it works
**Inputs:** A medical volume dataset (voxel grid with physical spacing) [S1, unofficial].
**Outputs:** Six surface render views (Front, Back, Left, Right, Top, Bottom) and associated projection data [S1, unofficial].

**Data Structures:**
* `Volume`: Contains physical dimensions $(L_x, L_y, L_z)$ in mm and voxel spacing $(s_x, s_y, s_z)$ [S1, unofficial].
* `Surface`: Boolean flag `large`. If `true`, the volume is rejected [S1, unofficial].
* `Crossings`: 16-bit integers representing ray-surface intersections [S1, unofficial].
* `Batch`: A chunk of 64 slices processed simultaneously [S1, unofficial].

**Parameters:**
* `MINIMUM_VOXEL`: 2 mm. The target sampling resolution [S1, unofficial].
* `LONGEST`: 2,000 mm. Maximum allowed extent along the head-to-foot axis (Z) [S1, unofficial].
* `WIDEST`: 1,000 mm. Maximum allowed extent along the other two axes (X, Y) [S1, unofficial].
* `BATCH_SIZE`: 64 slices [S1, unofficial].

**Steps:**

1. **Bound Check:**
   Calculate the volume's physical extent along the head-to-foot axis ($E_z$) and the maximum extent of the other two axes ($E_{xy} = \max(E_x, E_y)$) [S1, unofficial].
   If $E_z > \text{LONGEST}$ OR $E_{xy} > \text{WIDEST}$:
   * Set `Surface.large = true` [S1, unofficial].
   * Return: No mask, no image, no projection [S1, unofficial].
   * Slices beyond the cine strip are held (not decoded) [S1, unofficial].
   * [G] Ensure the volume orientation is known to identify the head-to-foot axis correctly.

2. **Determine Sampling Resolution:**
   Calculate the effective voxel spacing $s_{eff}$:
   $$ s_{eff} = \min(s_{min}, \text{MINIMUM\_VOXEL}) $$
   where $s_{min} = \min(s_x, s_y, s_z)$ is the smallest physical spacing of the input volume [S1, unofficial].
   * [G] If the volume is already finer than 2 mm, use its native spacing to preserve detail, but do not upsample.

3. **Generate Views:**
   Define six viewing directions. Five are standard orthogonal views. The sixth is the "From Behind" view [S1, unofficial].
   * **From Behind Configuration:** Rotate the view by 180 degrees around the vertical axis [S1, unofficial]. The patient's left remains on the viewer's left [S1, unofficial]. Rays travel along the voxels in the same direction as the frontal view (head-to-foot or foot-to-head depending on convention, but consistent with frontal) [S1, unofficial].

4. **Render Crossings (Batch Processing):**
   For each of the six views:
   a. Initialize a row of crossings for the view width [S1, unofficial].
   b. Iterate through the volume slices in batches of `BATCH_SIZE` (64 slices) [S1, unofficial].
   c. For each batch:
      i. Step rays for the current slice batch alone [S1, unofficial].
      ii. Calculate surface intersections (crossings) for the rays [S1, unofficial].
      iii. Store crossings as 16-bit integers to save memory [S1, unofficial].
   d. Combine results from all batches into the final row [S1, unofficial].
   e. For the "From Behind" view, add a tile to each row but do not create new rows of crossings, as its rays are reversed frontal rays [S1, unofficial].

5. **Handle Edge Cases:**
   * **Edge Marks:** The "From Behind" view draws edges facing the back [S1, unofficial]. This ensures that a mark within two voxels of the slice edge is shown face-on rather than edge-on, correcting a previous pitfall where such marks were only visible as edges from front/side views [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_surface(volume, min_voxel=2.0, longest=2000.0, widest=1000.0, batch_size=64):
    # 1. Bound Check
    if volume.extent_z > longest or max(volume.extent_x, volume.extent_y) > widest:
        return None, None, None  # Large volume, hold slices

    # 2. Determine Resolution
    min_spacing = min(volume.spacing_x, volume.spacing_y, volume.spacing_z)
    step_size = min(min_spacing, min_voxel)

    # 3. Generate Views
    views = [
        {"name": "Front", "dir": (0, 0, 1), "rot": 0},
        {"name": "Back", "dir": (0, 0, -1), "rot": 0},
        {"name": "Left", "dir": (1, 0, 0), "rot": 0},
        {"name": "Right", "dir": (-1, 0, 0), "rot": 0},
        {"name": "Top", "dir": (0, 1, 0), "rot": 0},
        {"name": "Bottom", "dir": (0, -1, 0), "rot": 0},
        {"name": "FromBehind", "dir": (0, 0, 1), "rot": 180}
    ]

    results = {}

    for view in views:
        crossings = []
        num_slices = volume.num_slices

        # 4. Batch Processing
        for start in range(0, num_slices, batch_size):
            end = min(start + batch_size, num_slices)
            batch = volume.get_slices(start, end)

            # Step rays for the current slice batch alone
            for ray in view.rays:
                for slice_data in batch:
                    # Calculate surface intersection for this ray and slice
                    intersection = compute_intersection(ray, slice_data, step_size)
                    if intersection is not None:
                        # Build crossings as 16-bit integers from the start to save memory
                        crossings.append(intersection & 0xFFFF)

        # For the "From Behind" view, add a tile to each row but do not create new rows
        if view["rot"] == 180:
            for i in range(len(crossings)):
                crossings[i] = (crossings[i] << 16) | 0x8000

        results[view["name"]] = crossings

    return results

def compute_intersection(ray, slice_data, step_size):
    # Standard ray-slice intersection calculation
    # Returns integer index or None
    return None  # Not given in the sources.
```

## Subtopics

### Standard Orthogonal Surface Rendering
The five standard views (Front, Back, Left, Right, Top). Used for general anatomical overview. Choose when standard DICOM viewer compatibility is required. [G]

### From-Behind Surface Rendering
The sixth view, rotated 180 degrees. Sets itself apart by showing posterior surfaces face-on, resolving edge-case visibility issues. Choose when reviewing posterior features or ensuring consistent facial preview in whole-body scans. [S1, unofficial]

## Operating characteristics
* **Time Complexity:** Linear with respect to the number of voxels in the volume, $O(N)$, where $N$ is the total voxel count within the bounds. The fixed resolution means the number of samples per ray is constant relative to the volume's physical depth, not its voxel count. [G]
* **Memory Complexity:** Dominated by the volume data itself. The largest rendered volume (1,000x1,000x2,000 mm) peaks at approximately 1.4 GB memory usage [S1, unofficial]. Using 16-bit integers for crossings reduces intermediate memory overhead compared to 64-bit floats. [S1, unofficial]
* **Measured Performance:** In a whole-body MR Otsu threshold calculation, memory usage dropped from 911 MB (with float64 copy) to 355 MB (direct comparison/chunked) [S1, unofficial].
* **Strengths:** Consistent visual detail for medical review regardless of volume length; prevents coarse sampling artifacts in long volumes; memory efficient through batching and 16-bit integer storage. [S1, unofficial]
* **Limitations:** Volumes exceeding 2,000 mm in length or 1,000 mm in width are not rendered at all; fixed resolution may be overkill for small volumes or under-resolving for very high-resolution small volumes (though native spacing is used if finer). [S1, unofficial]
* **When to use:** Medical QC packs, contour sheets, and previews where consistent facial/anatomical detail is critical across varying scan lengths. [S1, unofficial]
* **When not to use:** Volumes exceeding the physical bounds; real-time interactive rendering where memory is extremely constrained and the volume is very large. [G]

## Background
This method was introduced in the `pymedphys` project to address gaps in the QC pack's surface rendering capabilities. Prior to this change, volumes longer than 510 mm were coarsened to maintain a 256-voxel limit per side, leading to resolutions as coarse as ~7 mm for whole-body CTs. The change, documented in pull request #2434, added a "from behind" view and removed the voxel-side limit, enforcing a 2 mm sampling resolution for volumes within strict bounds [S1, unofficial]. It is used in medical physics QC tools for verifying contour accuracy and dose distributions. [S1, unofficial]

## Sources
[S1] Render surfaces from behind too, and at 2 mm however long the volume, https://github.com/pymedphys/pymedphys/pull/2434, fetched 2026-10-26 (unofficial)
