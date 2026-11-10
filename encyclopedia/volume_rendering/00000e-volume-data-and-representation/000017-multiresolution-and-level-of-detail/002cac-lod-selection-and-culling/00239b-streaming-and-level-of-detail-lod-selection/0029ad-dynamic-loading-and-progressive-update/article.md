# Dynamic Loading and Progressive Update

Also called: Chargement dynamique et mise à jour progressive

## Overview
Dynamic Loading and Progressive Update is a strategy for visualizing massive scenes by dynamically and adaptively loading data based on viewpoint position, bounding memory usage to visible data and required resolution [S1, unofficial]. It solves the problem of navigating arbitrarily large datasets without exceeding memory limits or stalling the frame rate [S1, unofficial]. It sets itself apart from its sibling [Streaming and Level-of-Detail (LOD) selection](3087-streaming-and-level-of-detail-lod-selection.md) by explicitly addressing the temporal aspect of data updates: while the parent selects the coarsest substitutive level based on projected splat footprint, this method focuses on the incremental update of the data structure (brick resolution and N3-tree subdivision) and spreading heavy loads across multiple frames to maintain real-time fluidity [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Viewpoint position, camera parameters, global scene bounds, data variation frequency, and the current state of the N3-tree structure [S1, unofficial].
- Output: An updated in-memory data structure (N3-tree with multi-resolution bricks) containing only the necessary data for the current view, rendered at the "useful resolution" [S1, unofficial].

**Data Structures:**
- **N3-tree:** A hierarchical structure where leaves support bricks [S1, unofficial].
- **Bricks:** Store voxel data. Sizes range from 16KB to 1MB [S1, unofficial].
- **Tree Nodes:** Store metadata. Updates involve sparse, small data (few bytes per node) [S1, unofficial].
- **Camera Parameter Request Bitmask:** Tracks pending reload states for point cloud variants (see Subtopics) [S3, unofficial].

**Parameters:**
- **LOD Bounds:** Lower and upper LOD bounds for each brick. Determined by global scene bounds and data variation frequency [S1, unofficial].
- **Minimum Frame Rate:** 10Hz, required to maintain correct refresh frequency during heavy loads [S1, unofficial].
- **Reload Delays:** `WVS_RELOAD_DELAY[0]` and `WVS_RELOAD_DELAY[1]` (milliseconds) for triggering 1/4 and full resolution point cloud reloads respectively [S3, unofficial].

**Steps:**
1. **Determine Useful Resolution:** Calculate the required resolution based on viewpoint position and data variation frequency [S1, unofficial].
2. **Apply LOD Strategies:**
   - **Brick Level:** Select the appropriate multi-resolution level for each brick. If the useful LOD is higher than the upper bound, merge bricks. If lower than the lower bound, subdivide the brick into N sub-bricks [S1, unofficial].
   - **Tree Level:** Subdivide or merge N3-tree leaves to adjust the spatial coverage of the bricks [S1, unofficial].
3. **Incremental Update:** Update the data structure to include new data. Avoid reloading all data at once to maintain navigation fluidity [S1, unofficial].
4. **Time-Sliced Loading (if needed):** If the amount of data to load exceeds the time available per frame (e.g., during sudden camera movements or entering unvisited zones), spread the loading operations across multiple frames, limiting the time available for loading in each frame to maintain at least 10Hz [S1, unofficial].
5. **Progressive Refinement (Wavelet Variant):** Load the lowest detail level (highest compression) first from the beginning of the brick file. Display the dataset at the current resolution level $i$. Triggered by a timer, switch to level $i-1$, $i-2$, etc., loading subsequent detail coefficients [S2].
6. **Adaptive Point Cloud Reload (Point Cloud Variant):**
   - Track camera parameter changes via a timestamp [S3, unofficial].
   - If elapsed time $> WVS\_RELOAD\_DELAY[0]$ and bit 0 of `currentCameraParameterRequest` is unset, request a 1/4 resolution point cloud update and set bit 0 [S3, unofficial].
   - If elapsed time $> WVS\_RELOAD\_DELAY[1]$ and bit 1 is unset, request a full-resolution update and set bits 0 and 1 to prevent redundant reloads [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Loading Cost:** Loading large bricks (16KB to 1MB) takes 60 microseconds to 4 milliseconds [S1, unofficial].
- **Sparse Update Cost:** Updating sparse tree nodes individually incurs high transfer latency costs due to small data size and memory sparsity [S1, unofficial].
- **Sudden Visibility:** Sudden camera movements or entering unvisited zones can cause too much data to become visible simultaneously, exceeding the time available per image for real-time refresh [S1, unofficial].
- **Rapid Movement:** Rapid camera movement could cause repeated full-resolution requests in point cloud variants. The bitmask flags combined with timestamp checks ensure each resolution tier is requested only once per movement phase [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_loading_update(viewpoint, camera_params, scene_bounds, data_variation_freq, current_tree, current_time):
    # 1. Determine useful resolution based on viewpoint position and data variation frequency [S1, unofficial]
    useful_lod = ... # Not given in the sources.

    # 2. Apply LOD strategies to visible bricks [S1, unofficial]
    for brick in current_tree.get_visible_bricks():
        lod_lower, lod_upper = ... # Not given in the sources.
        if useful_lod > lod_upper:
            merge_bricks(brick)
        elif useful_lod < lod_lower:
            subdivide_bricks(brick)

    # 3. Incremental Update [S1, unofficial]
    bricks_to_load = ... # Not given in the sources.

    # 4. Time-Sliced Loading [S1, unofficial]
    load_budget = 1.0 / 10.0  # 10Hz minimum frame rate
    current_load_time = 0.0
    for brick in bricks_to_load:
        brick_cost = ... # Not given in the sources.
        if current_load_time + brick_cost <= load_budget:
            load_brick(brick)
            current_load_time += brick_cost
        else:
            break

    # 5. Progressive Wavelet Refinement [S2]
    current_wavelet_level = ... # Not given in the sources.
    if timer_triggered():
        current_wavelet_level -= 1
        load_wavelet_coefficients(current_wavelet_level)

    # 6. Adaptive Point Cloud Reload [S3, unofficial]
    elapsed_time = current_time - last_camera_change_time
    if elapsed_time > WVS_RELOAD_DELAY[0] and not (currentCameraParameterRequest & 1):
        request_point_cloud_update(resolution=1/4)
        currentCameraParameterRequest |= 1
    if elapsed_time > WVS_RELOAD_DELAY[1] and not (currentCameraParameterRequest & 2):
        request_point_cloud_update(resolution=1)
        currentCameraParameterRequest |= 3

    return current_tree

# Helper definitions for steps not detailed in the sources
def merge_bricks(brick): pass
def subdivide_bricks(brick): pass
def load_brick(brick): pass
def load_wavelet_coefficients(level): pass
def request_point_cloud_update(resolution): pass
def timer_triggered(): return False
```

## Subtopics

### Time-Sliced Loading
A strategy to maintain real-time frame rates (minimum 10Hz) during heavy data loads by spreading necessary loading operations across multiple frames, limiting the time available for loading in each frame [S1, unofficial]. It is chosen when sudden camera movements or entering unvisited zones cause a spike in visible data that would otherwise stall the renderer [S1, unofficial]. Pitfall: If the load exceeds the time budget, visual feedback may lag, but the 10Hz floor prevents complete freeze [S1, unofficial].

### Progressive Wavelet Refinement
A rendering technique where data is loaded and displayed incrementally by wavelet detail levels to provide immediate visual feedback while refining resolution over time [S2]. It loads the lowest detail level (highest compression) first, then switches to lower levels ($i-1, i-2$) triggered by a timer [S2]. It is chosen when immediate visual feedback is critical and wavelet-compressed data is available [S2]. It differs from LaMar et al. [20] by keeping the wavelet level constant within each tile, whereas their algorithm generates regions of varying resolution within the final image [S2].

### Adaptive View-Dependent Point Cloud Reload
A method for reloading point cloud data based on camera movement thresholds and state flags [S3, unofficial]. It uses `WVS_RELOAD_DELAY[0]` and `WVS_RELOAD_DELAY[1]` to trigger 1/4 and full-resolution updates, respectively, using a bitmask (`currentCameraParameterRequest`) to prevent redundant reloads [S3, unofficial]. It is chosen for point cloud-based volume exploration where resolution adaptation is needed based on camera dynamics [S3, unofficial]. Pitfall: Rapid camera movement could cause repeated requests, but the bitmask and timestamp checks mitigate this [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of visible bricks and the cost of loading/subdividing them. Loading large bricks takes 60 microseconds to 4 milliseconds [S1, unofficial]. Sparse tree node updates incur high transfer latency costs [S1, unofficial].
- **Memory Complexity:** Bounded by visible data and required resolution ("useful resolution"), not the total scene size [S1, unofficial].
- **Measured Performance:** Maintains a minimum frame rate of 10Hz during heavy loads [S1, unofficial].
- **Scaling:** Scales to arbitrarily large scenes by bounding memory usage to visible data [S1, unofficial].
- **Strengths:** Allows navigation of massive scenes without memory overflow; maintains fluidity through incremental updates and time-slicing [S1, unofficial].
- **Limitations:** Sudden camera movements can cause load spikes; sparse tree updates are latency-heavy [S1, unofficial].
- **When to Use:** When visualizing massive, multi-resolution scenes with dynamic viewpoint changes [S1, unofficial].
- **When Not to Use:** Not given in the sources.

## Background
Originated from research on interactive exploration of procedural volumes, specifically the work described in "Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés" [S1, unofficial]. It builds on producer/consumer models [S1, unofficial] and wavelet-based rendering techniques [S2].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S2] Giga-Scale Multiresolution Volume Rendering on Distributed Display Clusters, https://doi.org/10.1007/978-3-642-19641-6_10, fetched 2026-10-26
[S3] larsxschneider/iVoxel, https://github.com/larsxschneider/iVoxel, fetched 2026-10-27 (unofficial)
