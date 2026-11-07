# Ray Shuffling & Reordering

Also called: shuffleRays, shuffleToNewGroup

## Overview
Ray Shuffling & Reordering permutes ray data across GPU arrays using an index mapping array to improve training order or batch distribution [S1, unofficial].

## How it works
**Inputs:**
- Source ray arrays (`src`) containing device arrays for `cameraCenterX_Device`, `cameraCenterY_Device`, `cameraCenterZ_Device`, `directionX_Device`, `directionY_Device`, `directionZ_Device`, `near_t_Device`, `far_t_Device`, `gtMask_Device`, and color channels (`gtGray_Device` if grayscale, else `gtRed_Device`, `gtGreen_Device`, `gtBlue_Device`) [S1, unofficial].
- Destination ray arrays (`dest`) with an identical layout [S1, unofficial].
- `shuffledIndexArray`: A host or device array of integers mapping destination indices to source indices [S1, unofficial].
- `src.totalRays`: Total number of rays [S1, unofficial].
- `src.graylized`: Boolean flag indicating grayscale mode [S1, unofficial].

**Outputs:**
- Destination ray arrays (`dest`) with rays reordered according to `shuffledIndexArray` [S1, unofficial].

**Data Structures:**
Rays are stored in Structure of Arrays (SoA) format on the GPU, with each attribute as a separate device array of size `totalRays` [G].

**Steps:**
1. Host function allocates destination memory for ray attributes [S1, unofficial].
2. Host function calls `ShuffleRank` to generate `shuffledIndexArray`. (Implementation not provided in the sources.) [G]
3. Host function copies `shuffledIndexArray` to GPU [S1, unofficial].
4. Host function launches the `shuffleToNewGroup` kernel with grid/block dimensions sized to `src.totalRays` [S1, unofficial].
5. Each thread computes its global `destRayIndex` from block and thread indices [S1, unofficial].
6. If `destRayIndex < src.totalRays`, the thread reads the source index from `shuffledIndexArray[destRayIndex]` [S1, unofficial].
7. The thread copies all ray attributes from the source arrays to the destination arrays using the mapped source index [S1, unofficial].
8. If `src.graylized` is true, copy `gtGray_Device`; otherwise, copy `gtRed_Device`, `gtGreen_Device`, and `gtBlue_Device` [S1, unofficial].

**Formulas:**
Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def shuffle_to_new_group(src, dest, shuffled_index_array, total_rays, graylized):
    """
    Per-thread kernel logic for shuffling ray attributes.
    In CUDA, dest_ray_index is computed as:
    (blockIdx.y * gridDim.x + blockIdx.x) * blockDim.x * blockDim.y + threadIdx.y * blockDim.x + threadIdx.x
    """
    for dest_ray_index in range(total_rays):
        src_index = shuffled_index_array[dest_ray_index]
        
        dest['cameraCenterX_Device'][dest_ray_index] = src['cameraCenterX_Device'][src_index]
        dest['cameraCenterY_Device'][dest_ray_index] = src['cameraCenterY_Device'][src_index]
        dest['cameraCenterZ_Device'][dest_ray_index] = src['cameraCenterZ_Device'][src_index]
        
        dest['directionX_Device'][dest_ray_index] = src['directionX_Device'][src_index]
        dest['directionY_Device'][dest_ray_index] = src['directionY_Device'][src_index]
        dest['directionZ_Device'][dest_ray_index] = src['directionZ_Device'][src_index]
        
        dest['near_t_Device'][dest_ray_index] = src['near_t_Device'][src_index]
        dest['far_t_Device'][dest_ray_index] = src['far_t_Device'][src_index]
        
        dest['gtMask_Device'][dest_ray_index] = src['gtMask_Device'][src_index]
        
        if graylized:
            dest['gtGray_Device'][dest_ray_index] = src['gtGray_Device'][src_index]
        else:
            dest['gtRed_Device'][dest_ray_index] = src['gtRed_Device'][src_index]
            dest['gtGreen_Device'][dest_ray_index] = src['gtGreen_Device'][src_index]
            dest['gtBlue_Device'][dest_ray_index] = src['gtBlue_Device'][src_index]
```

**Parameters:**
- `shuffledIndexArray`: Host or device array of integers mapping destination indices to source indices [S1, unofficial].
- `maxIndex`: Total number of rays to shuffle (equivalent to `src.totalRays`) [S1, unofficial].
- `initialized`: Boolean controlling host memory allocation for indices [S1, unofficial].

**Edge Cases and Pitfalls:**
- If `destRayIndex >= src.totalRays`, the thread returns immediately [S1, unofficial].
- Not given in the sources.

## Subtopics

### ShuffleRank
Generates the permutation indices. Not given in the sources.

### Batch Ray Sampling
Uses shuffled rays for training. Links to [Batch Ray Sampling](5447-batch-ray-sampling.md).

## Operating characteristics
- Time and memory complexity: Not given in the sources.
- Measured performance, scaling, strengths, limitations, and usage conditions: Not given in the sources.

## Background
The method is implemented in the `nortonBryan/Clear-plenoxels` repository [S1, unofficial]. Origin details, broader history, and specific usage contexts beyond the provided code: Not given in the sources.

## Sources
[S1] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
