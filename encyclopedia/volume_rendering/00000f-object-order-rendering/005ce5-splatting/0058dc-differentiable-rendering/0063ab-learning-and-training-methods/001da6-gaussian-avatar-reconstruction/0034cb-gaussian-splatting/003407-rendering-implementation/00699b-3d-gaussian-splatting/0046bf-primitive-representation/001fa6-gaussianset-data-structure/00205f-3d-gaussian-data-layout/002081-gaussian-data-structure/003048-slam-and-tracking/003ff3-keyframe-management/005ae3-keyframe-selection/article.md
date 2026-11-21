# Keyframe Selection

Also called: Keyframe Selection Strategy

## Overview
Keyframe selection determines when to add a new frame to the optimization window to maintain long-term map consistency and prevent catastrophic forgetting in online SLAM systems [S1]. It solves the problem of balancing the need for new geometric information against the computational cost of joint optimization [S1]. Unlike fixed-interval strategies, this method uses an explicit voxel structure to test tracked frames against the existing map, inserting keyframes only when they contribute significant new allocatable voxels or when a maximum time interval has elapsed [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A successfully tracked RGB-D frame, the current voxel map (implicitly or explicitly defined), the current set of keyframes, and the maximum interval parameter $N$.
*   **Output:** A boolean flag indicating whether the current frame should be added to the keyframe set $W_k$.

**Data Structures:**
*   **Voxel Map:** A 3D grid representing the scene. The specific layout depends on the underlying representation (e.g., dense voxel grid for Vox-Fusion [S1] or sampled rays for INR-SLAM [S2]).
*   **Keyframe Set ($W_k$):** A list or window of indices of previously selected keyframes.

**Parameters:**
*   $N_c$: The number of voxels that would be newly allocated (or significantly updated) if the current frame were chosen as a keyframe [S1].
*   $N_o$: The number of currently observed voxels in the existing map [S1].
*   $N$: The maximum number of frames allowed between keyframe insertions [S1].
*   $\tau$: The threshold for the ratio $p_{kf}$ (Not given in the sources. [G]: Typically a small value like 0.1–0.3, but exact value not specified in cited texts).

**Steps:**
1.  **Track Frame:** Perform tracking to obtain the pose of the current frame relative to the map.
2.  **Intersection Test:** For the current frame, perform an intersection test against the existing voxel map to determine $N_c$, the count of voxels that would be allocated if this frame were added as a keyframe [S1].
3.  **Calculate Ratio:** Compute the ratio $p_{kf} = N_c / N_o$ [S1].
4.  **Check Novelty Condition:** If $p_{kf} > \tau$, mark the frame as a candidate keyframe [S1].
5.  **Check Interval Condition:** If no keyframe has been created for the past $N$ frames, mark the current frame as a candidate keyframe to prevent missing parts in loopy camera motions [S1].
6.  **Insert Keyframe:** If either condition is met, add the current frame to $W_k$ and update the map [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_keyframe(current_frame, voxel_map, keyframe_window, N, tau):
    """
    Determines if current_frame should be added to the keyframe set.
    
    Args:
        current_frame: The tracked RGB-D frame with pose.
        voxel_map: The current voxel representation of the scene.
        keyframe_window: List of indices of recent keyframes.
        N: Maximum interval between keyframes.
        tau: Threshold for novel voxel ratio.
        
    Returns:
        bool: True if the frame should be added as a keyframe.
    """
    # Step 1: Count currently observed voxels in the map
    N_o = voxel_map.count_observed_voxels()
    
    # Step 2: Perform intersection test to find allocatable voxels
    # This simulates adding the frame to the map and counting new allocations
    N_c = voxel_map.count_new_allocations(current_frame)
    
    # Step 3: Calculate ratio
    if N_o == 0:
        p_kf = 1.0 # Avoid division by zero; first frame is always keyframe
    else:
        p_kf = N_c / N_o
        
    # Step 4: Check novelty threshold
    is_novel = p_kf > tau
    
    # Step 5: Check maximum interval
    # If no keyframe has been added in the last N frames
    frames_since_last_kf = len(keyframe_window) # Assuming window tracks sequence
    # Note: In practice, this checks the index difference or time difference
    # If the source implies a sliding window of size N, we check if the window is full
    # or if the last keyframe was too long ago.
    # Simplified interpretation based on "past N frames":
    is_interval_expired = (frames_since_last_kf >= N)
    
    # Step 6: Decision
    if is_novel or is_interval_expired:
        return True
    else:
        return False
```

**Edge Cases and Pitfalls:**
*   **Loopy Motions:** Without the interval constraint ($N$), the system may fail to allocate new keyframes in long-term loops, resulting in missing parts of the model due to lack of multi-view constraints [S1].
*   **Exploratory Movements:** In exploratory movements, new voxels are constantly allocated, making the ratio strategy adequate [S1].
*   **Catastrophic Forgetting:** Online incremental learning is prone to forgetting; joint optimization with keyframes mitigates this [S1].

## Subtopics

### Keyframe Management and Optimization
This variant focuses on the optimization strategy associated with the selected keyframes rather than just the selection criterion. It employs a global bundle adjustment strategy for enhanced pose estimation and consistent global reconstruction, following approaches in [42, 56]. To improve storage and retrieval efficiency, it selectively caches only key sampled rays from keyframes for optimization purposes [S2]. Training is continuously conducted in real-time to optimize objectives including photometric and geometric losses (Eq. 4), SDF loss, and free space suppressing loss [S2].

## Operating characteristics
*   **Time Complexity:** The intersection test requires iterating over the voxel map or ray samples, which is linear with respect to the number of voxels or samples per frame [S1, S2].
*   **Memory Complexity:** Depends on the voxel map size and the number of cached rays/keyframes [S1, S2].
*   **Strengths:** Simple yet effective at creating consistent scene maps; prevents catastrophic forgetting; handles loopy motions via interval constraint [S1].
*   **Limitations:** Requires an explicit voxel structure or ray sampling for the intersection test; threshold $\tau$ is not specified in the sources [S1].
*   **When to Use:** Online SLAM systems requiring long-term consistency and joint optimization of geometry and poses [S1, S2].

## Background
The method originates from Vox-Fusion, which introduced dense tracking and mapping with a voxel-based neural implicit representation [S1]. It addresses the limitation of previous works that used heuristically chosen metrics or fixed intervals for keyframe insertion [S1]. The approach is designed to ensure long-term map consistency and prevent catastrophic forgetting in online incremental learning [S1].

## Sources
[S1] Vox-Fusion: Dense Tracking and Mapping with Voxel-based Neural Implicit Representation, https://doi.org/10.1109/ismar55827.2022.00066, fetched 2026-10-26
[S2] Benchmarking Implicit Neural Representation and Geometric Rendering in Real-Time RGB-D SLAM, https://doi.org/10.1109/cvpr52733.2024.02017, fetched 2026-10-26
