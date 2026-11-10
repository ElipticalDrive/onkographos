# Visual Hull Oracle Kernel


## Overview
The Visual Hull Oracle Kernel performs binary culling of voxels to determine if they intersect the visual hull surface $S(p) = 0.5$ [S1]. It identifies which voxels in a sparse voxel grid contribute to the final volume by leveraging hierarchical mip pyramids over silhouette images to compute conservative bounds on voxel coverage [S1]. It is distinguished from sibling methods by its specific use of component-wise multiplication of conservative bounds across multiple camera views to make a definitive KEEP or CULL decision, building on the silhouette acquisition and filtering techniques described in [Silhouette Image Acquisition and Filtering](3754-silhouette-image-acquisition-and-filtering.md) [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A list of voxels (`VoxelListIn`), $n$ filtered silhouette images, $n$ corresponding mip pyramids, and coordinate transforms from Novel View Camera (NVC) space to silhouette image space and screen space [S1].
*   **Output:** A decision array (`oracleOutput`) where each entry is either `CULL` or `KEEP` [S1].

**Data Structures:**
*   `VoxelListIn`: Array of voxel structures, indexed by `threadID` [S1].
*   `oracleOutput`: Array of integers, indexed by `threadID` [S1].
*   `bounds_x` and `bounds_y`: Scalars storing the lower and upper limits of the conservative bound on the silhouette coverage for a voxel [S1].

**Parameters:**
*   `numCameras`: The number of input camera views [S1].
*   `bounds_x`: Lower limit of the conservative bound on $S(p)$ [S1].
*   `bounds_y`: Upper limit of the conservative bound on $S(p)$ [S1].

**Algorithm Steps:**
1.  Initialize `DECISION` to `CULL` [S1].
2.  Read the current voxel $v$ from `VoxelListIn[threadID]` [S1].
3.  Check if the voxel is outside the viewing frustum of the novel view using `outsideFrustum(v)` [S1].
    *   If `outsideFrustum(v)` is true, skip further processing for this voxel and write `CULL` to `oracleOutput[threadID]` [S1].
    *   If `outsideFrustum(v)` is false, proceed to step 4 [S1].
4.  Initialize `bounds_x` and `bounds_y` to `1.0` [S1].
5.  Loop through each camera index $i$ from $0$ to `numCameras` - 1:
    *   Call `sampleSilhouette(v, i)` to retrieve the conservative bound of voxel $v$'s projection into silhouette image $i$ [S1].
    *   Update `bounds_x` and `bounds_y` by performing component-wise multiplication with the returned values [S1].
6.  After the loop, evaluate the final `bounds_x` and `bounds_y`:
    *   If `bounds_x < 0.5` AND `bounds_y > 0.5`, set `DECISION` to `KEEP` [S1].
    *   Otherwise, `DECISION` remains `CULL` [S1].
7.  Write `DECISION` to `oracleOutput[threadID]` [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
CULL = 0
KEEP = 1
# outsideFrustum(v) returns True if voxel v lies outside the novel view frustum [S1].
# sampleSilhouette(v, i) returns a tuple (lower_bound, upper_bound) for voxel v in camera i [S1].
def visual_hull_oracle_kernel(threadID, VoxelListIn, oracleOutput, numCameras, outsideFrustum, sampleSilhouette):
    DECISION = CULL
    v = VoxelListIn[threadID]
    if not outsideFrustum(v):
        bounds_x = 1.0
        bounds_y = 1.0
        for i in range(numCameras):
            cam_bounds = sampleSilhouette(v, i)
            bounds_x *= cam_bounds[0]
            bounds_y *= cam_bounds[1]
        if bounds_x < 0.5 and bounds_y > 0.5:
            DECISION = KEEP
    oracleOutput[threadID] = DECISION
```

**Edge Cases and Pitfalls:**
*   **Frustum Culling:** Voxels outside the viewing frustum of the novel view are automatically culled via the `outsideFrustum(v)` check to avoid unnecessary computation [S1].
*   **Precision:** The method relies on conservative bounds; if the mip pyramid levels are too coarse, the bounds may be inaccurate, potentially leading to false positives (keeping voxels that should be culled) or false negatives (culling voxels that should be kept) [S1].
*   **Surface Definition:** The decision is based on the intersection with the surface $S(p) = 0.5$, making the thresholds 0.5 specific to this definition [S1].

## Subtopics

### GPU-based Photo-hull Retrieval
A voxel-based approach that retrieves the photo-hull of a shape using a view-dependent plane-sweeping strategy [S2, unofficial]. It differs from the standard Visual Hull Oracle by combining rendering and reconstruction into a single step, producing no explicit volume [S2, unofficial]. A key pitfall is that it performs photo-consistency checks on each fragment regardless of rejection by silhouette/background tests due to a lack of branching support at the time [S2, unofficial]. Performance achieves rendering frame rates of 2-3 frames per second [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels processed, as each voxel culling decision is independent and assigned to a single thread [S1].
*   **Memory Complexity:** Requires storage for $n$ silhouette images and $n$ mip pyramids, where $n$ is the number of cameras [S1].
*   **Performance:** The hierarchical, coarse-to-fine manner of processing accelerates computation [S1]. Specific frame rates for the oracle kernel alone are not given, but related GPU-based photo-hull methods achieve 2-3 fps [S2, unofficial].
*   **Strengths:** Efficient culling of voxels outside the frustum and those not intersecting the visual hull surface [S1].
*   **Limitations:** Requires pre-computed mip pyramids and silhouette images [S1].

## Background
The Visual Hull Oracle Kernel was introduced in the context of real-time high-resolution sparse voxelization for image-based modeling [S1]. It builds upon hierarchical z-buffer concepts and uses mip pyramids to accelerate voxel/silhouette coverage queries [S1]. The approach is part of a broader shift towards GPU-based processing for volumetric reconstruction, following earlier work on GPU-based visual hull rendering [S2, unofficial].

## Sources
[S1] Real-time high-resolution sparse voxelization with application to image-based modeling, https://doi.org/10.1145/2492045.2492053, fetched 2026-11-01
[S2] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
