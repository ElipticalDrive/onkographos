# Keyframe Selection

Also called: Keyframe Selection Strategy

## Overview
Keyframe selection determines when to add a new frame to the optimization window to maintain long-term map consistency and prevent catastrophic forgetting in online SLAM systems [S1]. Unlike fixed-interval strategies, this method uses an explicit voxel structure to test tracked frames against the existing map, inserting keyframes only when they contribute significant new allocatable voxels or when a maximum interval has elapsed [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A successfully tracked frame and the current voxel map [S1].
*   **Output:** A boolean decision indicating whether the frame should be added to the keyframe set [S1].

**Data Structures:**
*   **Voxel Map:** An explicit 3D grid representing the scene geometry [S1].
*   **Keyframe Set:** A collection of previously selected keyframes used for joint optimization [S1].
*   **Frame Counter:** A value tracking the number of frames processed since the last keyframe insertion [G].

**Parameters:**
*   $N_c$: The number of voxels that would be newly allocated if the current frame were chosen as a keyframe [S1].
*   $N_o$: The number of currently observed voxels in the existing map [S1].
*   $N$: The maximum number of frames allowed between keyframe insertions [S1].
*   $\tau$: The threshold for the ratio $p_{kf}$ (Not given in the sources.)

**Steps:**
1.  **Track Frame:** Obtain the pose of the current frame relative to the map [S1].
2.  **Intersection Test:** Test the current frame against the existing voxel map to count $N_c$, the number of voxels that would be allocated [S1].
3.  **Calculate Ratio:** Compute the novelty ratio $p_{kf} = N_c / N_o$ [S1].
4.  **Check Novelty Condition:** If $p_{kf} > \tau$, mark the frame as a candidate keyframe [S1].
5.  **Check Interval Condition:** If no keyframe has been created for the past $N$ frames, mark the current frame as a candidate keyframe to prevent missing parts in loopy camera motions [S1].
6.  **Insert Keyframe:** If either condition is met, add the current frame to the keyframe set and update the map [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_keyframe(current_frame, voxel_grid, frames_since_last_kf, N, threshold):
    # Count currently observed voxels in the map
    N_o = 0
    for voxel in voxel_grid:
        if voxel.is_observed():
            N_o += 1
    # Perform intersection test to find allocatable voxels for current_frame
    N_c = 0
    for voxel in voxel_grid:
        if voxel.is_unallocated() and voxel.intersects(current_frame):
            N_c += 1
    # Calculate ratio
    if N_o == 0:
        p_kf = 1.0
    else:
        p_kf = N_c / N_o
    # Check novelty condition
    is_novel = p_kf > threshold
    # Check interval condition
    is_interval_expired = frames_since_last_kf >= N
    # Decision
    if is_novel or is_interval_expired:
        return True, 0
    return False, frames_since_last_kf + 1
```

**Edge Cases and Pitfalls:**
*   **Loopy Motions:** Without the interval constraint ($N$), the system may fail to allocate new keyframes in long-term loops, resulting in missing parts of the model due to lack of multi-view constraints [S1].
*   **Exploratory Movements:** In exploratory movements, new voxels are constantly allocated, making the ratio strategy adequate [S1].
*   **Catastrophic Forgetting:** Online incremental learning is prone to forgetting; joint optimization with keyframes mitigates this [S1].

## Subtopics

### Keyframe Management and Optimization
This variant extends the selection process by integrating the selected keyframes into a global optimization framework rather than relying solely on incremental updates [S2]. It differs from standard selection by employing a global bundle adjustment strategy for enhanced pose estimation and consistent global reconstruction [S2]. It is chosen when long-term geometric consistency and efficient ray caching are prioritized over purely local incremental mapping [S2]. Training continuously optimizes photometric, geometric, SDF, and free space suppressing losses using the cached keyframe rays [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Simple yet effective at creating consistent scene maps; prevents catastrophic forgetting; handles loopy motions via interval constraint [S1].
*   **Limitations:** Requires an explicit voxel structure or ray sampling for the intersection test; threshold $\tau$ is not specified in the sources [S1].
*   **When to Use:** Online SLAM systems requiring long-term consistency and joint optimization of geometry and poses [S1].

## Background
The method is described in Vox-Fusion, which introduced dense tracking and mapping with a voxel-based neural implicit representation [S1]. It addresses the limitation of previous works that used heuristically chosen metrics or fixed intervals for keyframe insertion [S1]. The approach is designed to ensure long-term map consistency and prevent catastrophic forgetting in online incremental learning [S1].

## Sources
[S1] Vox-Fusion: Dense Tracking and Mapping with Voxel-based Neural Implicit Representation, https://doi.org/10.1109/ismar55827.2022.00066, fetched 2026-10-26
[S2] Benchmarking Implicit Neural Representation and Geometric Rendering in Real-Time RGB-D SLAM, https://doi.org/10.1109/cvpr52733.2024.02017, fetched 2026-10-26
