# Brick LOD Selection

Also called: Choix du niveau de LOD des briques

## Overview
Brick LOD Selection determines the appropriate level of detail (LOD) for volumetric bricks based on the distance to the viewpoint, aiming to ensure that a projected voxel covers exactly one screen pixel [S1, unofficial]. It differs from sibling methods by calculating resolution bounds for entire brick volumes rather than individual samples, ensuring that all necessary LOD levels within the brick's spatial extent are loaded [S1, unofficial].

## How it works
**Inputs:** Viewpoint position, brick geometry (bounding box corners), projection parameters.
**Outputs:** A set of LOD levels required for the brick.

**Data Structures:**
*   **Brick Bounding Box:** Defined by its eight corners in world space.
*   **LOD Interval:** A pair of integers $[L_{min}, L_{max}]$ representing the required LOD levels.

**Steps:**
1.  **Assume Spherical/Circular Geometry:** To simplify resolution determination and make it independent of voxel orientation relative to the projection surface, assume spherical pixels and circular voxels [S1, unofficial].
2.  **Calculate Ideal Voxel Size:** The ideal voxel size is proportional to its distance from the viewpoint [S1, unofficial].
3.  **Determine Lower Bound ($L_{min}$):** Calculate the distance from the viewpoint to the farthest corner of the brick. Determine the minimum LOD level required such that the voxel size at this distance satisfies the projection constraint [S1, unofficial].
4.  **Determine Upper Bound ($L_{max}$):** Calculate the distance from the viewpoint to the nearest corner of the brick. Determine the maximum LOD level required such that the voxel size at this distance satisfies the projection constraint [S1, unofficial].
5.  **Load LODs:** Identify all LOD levels contained within the interval $[L_{min}, L_{max}]$. If these levels are not already present in storage, load them [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_brick_lod(brick_corners, viewpoint, projection_params):
    """
    brick_corners: List of 8 3D vectors representing the brick's corners in world space.
    viewpoint: 3D vector representing the camera position.
    projection_params: Object containing projection matrix and screen resolution info.
    """
    # Step 1 & 2: Ideal voxel size is proportional to distance.
    # We need to find LOD levels where projected voxel size == screen pixel size.
    
    # Step 3: Lower bound based on farthest corner
    distances = [dist(c, viewpoint) for c in brick_corners]
    max_dist = max(distances)
    lod_min = calculate_lod_for_distance(max_dist, projection_params)
    
    # Step 4: Upper bound based on nearest corner
    min_dist = min(distances)
    lod_max = calculate_lod_for_distance(min_dist, projection_params)
    
    # Step 5: Load all LODs in interval
    required_lods = range(lod_min, lod_max + 1)
    return required_lods

def calculate_lod_for_distance(distance, params):
    """
    Calculates the LOD level where a voxel projects to exactly one pixel.
    This is a placeholder for the specific formula derived from the spherical/circular assumption.
    The exact formula depends on the projection matrix and pixel aspect ratio.
    """
    # Not given in the sources. Implementation requires mapping distance to LOD index
    # such that voxel_size(distance) * projection_scale == pixel_size.
    pass

def dist(v1, v2):
    return ((v1.x - v2.x)**2 + (v1.y - v2.y)**2 + (v1.z - v2.z)**2)**0.5
```

## Subtopics

### Brick Misregistration Fix
Fixes geometric seams caused by multi-LOD bricks fetching boxes snapped to whole voxels of their pyramid level, which were previously treated as owned boxes, causing content shift and stretch [S2, unofficial]. The fix replaces the use of the fetched box for remap with `chunkOwnedTexBox` [S2, unofficial]. This change is applied as the single source for the remap in all three call sites: `chunkUniformsFor`, `chunkSampleTransform`, and the third implicit site [S2, unofficial]. Without this fix, neighboring bricks snap independently, leading to disagreement about level coordinates at shared faces, creating geometric seams that cannot be removed by brightness adjustments [S2, unofficial]. On the `hoa_heart` pyramid, this moves brick content by up to 3.4/6.3/8.7 common voxels at L2/L3/L4 and corrects stretch of up to 5.9/12.4/23.7 (approx 61 um displacement on an L4 brick) [S2, unofficial]. Single-level plans take an exact path and remain bit-identical to previous behavior [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The method originates from research on interactive exploration of extended and detailed procedural volumes [S1, unofficial]. It addresses the need to maintain visual quality while managing memory bandwidth and storage by selecting appropriate LODs for brick-based volume representations [S1, unofficial].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S2] feat(niivue): streamed multi-LOD rendering fixes, LOD compensation, budget plans, and block inspection, https://github.com/niivue/mono/pull/138, fetched 2026-10-27 (unofficial)
