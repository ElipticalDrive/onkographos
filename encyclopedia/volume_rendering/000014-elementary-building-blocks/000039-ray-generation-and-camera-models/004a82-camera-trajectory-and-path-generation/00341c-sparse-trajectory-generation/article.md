# Sparse Trajectory Generation
Also called: Mapping-based Trajectory Selection

## Overview
Sparse Trajectory Generation reduces a dense camera path to a subset of frames by filtering based on a predefined mapping file [S1, unofficial]. It differs from [Parametric Path Generation](11476-parametric-path-generation.md) by relying on discrete frame indices rather than continuous mathematical functions [G]. It differs from [Synthetic Camera Trajectory Generation](7942-synthetic-camera-trajectory-generation.md) by acting as a post-processing filter rather than a generative model [G].

## How it works
**Inputs and Outputs**
- Input: `f_trajectory`, a list of camera frames; `mapping`, a list of entries where each entry contains a frame index [S1, unofficial].
- Output: `sparse_traj`, a list of camera frames corresponding to the selected indices [S1, unofficial].

**Data Structures**
- `f_trajectory`: A sequence of frame objects [S1, unofficial].
- `mapping`: A sequence of entries. Each entry `m` is structured such that `m[1]` holds the 1-based frame index [S1, unofficial].
- `sparse_traj`: A dynamic list of frame objects [S1, unofficial].

**Steps**
1. Initialize an empty list `sparse_traj` [S1, unofficial].
2. Iterate through each entry `m` in `mapping` [S1, unofficial].
3. Convert `m[1]` to an integer, then subtract 1 to adjust for 0-based indexing [S1, unofficial].
4. Append `f_trajectory[index]` to `sparse_traj` [S1, unofficial].
5. Return `sparse_traj` [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_sparse_trajectory(f_trajectory, mapping):
    sparse_traj = []
    for m in mapping:
        frame_index_1_based = int(m[1])
        index = frame_index_1_based - 1
        sparse_traj.append(f_trajectory[index])
    return sparse_traj
```

**Parameters and Edge Cases**
- If `mapping` is empty, the result is an empty list [G].
- If `index` is out of bounds for `f_trajectory`, a robust implementation should check bounds before access [G].
- The 1600-frame threshold for optional sparsification applies only to the alignment variant, not the general generation method [S1, unofficial].

## Subtopics

### [Trajectory Alignment (RANSAC)](7876-trajectory-alignment-ransac.md)
Aligns a sparse camera trajectory to a ground truth trajectory using RANSAC-based point cloud registration. It differs from standard global alignment by handling sparse, potentially noisy inputs through iterative sampling, and optionally sparsifies the input trajectory if a mapping file is provided and the trajectory length exceeds 1600 frames.

### Extrapolated View Partitioning Procedure
For the Waymo dataset, generates testing views by translating the camera by 2 to 3 meters or rotating it by 10 to 15 degrees horizontally [S2]. For LLFF and Mip-NeRF 360, partitions the full trajectory into training and testing sets to ensure ground truth images exist for extrapolated views [S2]. For scenes lacking obvious extrapolated trajectories, sparsifies the training views to create relative extrapolated trajectories [S2].

## Operating characteristics
- **Time Complexity**: Linear with respect to the size of the `mapping` list, $O(N)$, where $N$ is the number of entries [G].
- **Memory Complexity**: Linear with respect to the number of selected frames, $O(K)$, where $K$ is the length of `sparse_traj` [G].
- **Strengths**: Simple to implement; computationally inexpensive; deterministic based on the mapping file [G].
- **Limitations**: Does not optimize for view coverage or redundancy; relies entirely on the quality of the provided mapping file [G].
- **When to use**: When a specific subset of frames is known a priori to be sufficient or required [G].

## Background
Not given in the sources.

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] FreeFix: Boosting 3D Gaussian Splatting via Fine-Tuning-Free Diffusion Models, https://doi.org/10.1109/3dv69130.2026.00109, fetched 2026-10-30
