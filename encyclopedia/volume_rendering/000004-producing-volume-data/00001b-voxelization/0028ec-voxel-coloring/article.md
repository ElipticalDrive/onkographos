# Voxel Coloring


## Overview
Voxel Coloring is a volumetric reconstruction algorithm that uses color consistency across multiple views to distinguish object surface voxels from background [S1, unofficial]. It solves the problem of 3D shape recovery from uncalibrated or calibrated multi-view images by starting with a fully opaque volume and iteratively carving away voxels that exhibit inconsistent colors across input images [S1, unofficial]. Unlike Space Carving, which requires multiple scans and arbitrary camera placement, Voxel Coloring relies on an ordinal visibility constraint to allow reconstruction in a single monotonic scan, simplifying visibility handling at the cost of restricting camera placement [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of $N$ calibrated images with known camera poses; a 3D reconstruction volume (bounding box) discretized into a grid of voxels.
- **Outputs:** A set of "surface" voxels (opaque) and "background" voxels (transparent).

**Data Structures:**
- **Voxel Grid:** A 3D array of voxels. Each voxel has an occupancy state: `Opaque` (potentially surface) or `Transparent` (carved/background).
- **Image Data:** For each camera $i$, the color $C_i(x)$ at any projected pixel location $x$.

**Algorithm Steps:**
1.  **Initialization:** Initialize all voxels within the bounding box as `Opaque`.
2.  **Visibility Constraint:** Ensure camera locations satisfy the ordinal visibility constraint. This requires that voxels can be visited in a single near-to-far scan relative to every camera. This implies cameras cannot surround the scene [S1, unofficial].
3.  **Traversal:** Traverse the voxels in a monotonic order (e.g., increasing distance from a reference point or along a specific axis) that respects the ordinal visibility constraint for all cameras [S1, unofficial].
4.  **Color Consistency Test:** For each `Opaque` voxel $v$:
    a. Project $v$ into each input image $i$ to obtain the pixel location $p_i$.
    b. Sample the color $C_i(p_i)$ from image $i$.
    c. Check if the colors $\{C_i(p_i)\}$ are consistent. Consistency is typically defined by a threshold on the color difference (e.g., Euclidean distance in RGB space) between the colors observed across all views where the voxel is visible.
    d. If the colors are inconsistent, mark $v$ as `Transparent` (carve).
    e. If the colors are consistent, mark $v$ as `Surface` (keep opaque).
5.  **Termination:** Stop when all voxels have been processed. The remaining `Opaque` voxels constitute the reconstructed surface.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_coloring(images, cameras, bounding_box, resolution, color_threshold):
    # Initialize volume: 1 = Opaque, 0 = Transparent
    volume = np.ones(resolution, dtype=np.uint8)
    
    # Define traversal order based on ordinal visibility constraint
    # This order must be monotonic relative to all camera centers
    voxel_indices = get_monotonic_voxel_order(bounding_box, resolution, cameras)
    
    for idx in voxel_indices:
        voxel_center = index_to_world(idx, bounding_box, resolution)
        
        # Check consistency across all views
        consistent = True
        colors = []
        
        for i, camera in enumerate(cameras):
            # Project voxel center to image plane
            pixel_uv = project(voxel_center, camera)
            
            # Check if projection is within image bounds
            if not is_inside_image(pixel_uv, images[i].shape):
                continue # Skip views where voxel is not visible
            
            # Sample color
            color = images[i].sample(pixel_uv)
            colors.append(color)
        
        # If voxel is visible in at least one view, check consistency
        if len(colors) > 0:
            # Compute max pairwise color difference
            max_diff = 0.0
            for c1 in colors:
                for c2 in colors:
                    diff = np.linalg.norm(c1 - c2)
                    if diff > max_diff:
                        max_diff = diff
            
            # Carve if inconsistent
            if max_diff > color_threshold:
                volume[idx] = 0
            else:
                volume[idx] = 1 # Keep as surface
        else:
            # Not visible in any view; treat as background
            volume[idx] = 0
            
    return volume
```

**Parameters:**
- **Color Threshold:** Maximum allowed color difference for consistency. Not given in the sources. [G] Typical values depend on image bit depth and noise levels.
- **Ordinal Visibility Constraint:** Cameras must be arranged such that a single monotonic scan order works for all. This prevents cameras from surrounding the scene [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Surrounding Cameras:** If cameras surround the scene, the ordinal visibility constraint is violated, and the single-scan approach fails. Use Space Carving instead [S1, unofficial].
- **Non-Lambertian Surfaces:** Specularities or view-dependent effects can cause color inconsistency, leading to incorrect carving. This is addressed by post-processing methods like Multi-view Color Consistency and Smoothing [S2, unofficial].
- **Calibration Errors:** Inaccurate camera poses lead to misprojection and false inconsistency.

## Subtopics

### [Generalized Voxel Coloring](5526-generalized-voxel-coloring.md)
Computes visibility exactly to determine color consistency, addressing approximation errors in earlier methods. Ensures voxels are retained only if consistent across all strictly visible cameras.

### [Multi-view Color Consistency and Smoothing](6445-multi-view-color-consistency-and-smoothing.md)
Post-processing method to handle noise, non-Lambertian reflectance, and quantization artifacts by refining colors of identified surface voxels.

## Operating characteristics
- **Time Complexity:** $O(V \cdot N)$, where $V$ is the number of voxels and $N$ is the number of cameras. Each voxel is tested against all views.
- **Memory Complexity:** $O(V)$ for the volume grid.
- **Strengths:** Simple implementation; single-scan efficiency due to ordinal constraint; robust to noise if threshold is tuned.
- **Limitations:** Cannot reconstruct surfaces not visible in any image; requires ordinal visibility constraint (no surrounding cameras); sensitive to calibration errors and non-Lambertian surfaces.
- **When to Use:** When cameras are arranged in a non-surrounding configuration (e.g., a subset of views around an object) and computational efficiency is prioritized.
- **When Not to Use:** When cameras surround the scene (use Space Carving) or when high precision on complex lighting is required (use Generalized Voxel Coloring or smoothing).

## Background
Voxel Coloring was introduced by Seitz and Dyer as an improvement over Space Carving [S1, unofficial]. It simplifies the visibility computation by imposing an ordinal visibility constraint, allowing for a single monotonic scan of the voxel volume. This contrasts with Space Carving, which evaluates planes of voxels at a time using multiple scans to handle arbitrary camera placements [S1, unofficial]. The method is foundational in multi-view stereo volumetric reconstruction.

## Sources
[S1] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
