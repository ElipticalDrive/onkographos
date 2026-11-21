# Voxel Value Counting
Also called: RegularGrid::countValues
## Overview
Voxel Value Counting iterates over a voxel grid to tally the occurrences of each unique voxel value (fragment ID), explicitly ignoring empty and free voxels. It differs from [Voxel Material Classification](3645-voxel-material-classification.md) by producing raw frequency histograms rather than semantic labels, and from [Voxel Solidity Check](3758-voxel-solidity-check.md) by quantifying density rather than binary presence. This method is essential for statistical analysis of voxelized data, such as estimating object volume or material distribution.
## How it works
**Inputs and Outputs:**
- Input: A voxel grid where each cell stores a value (fragment ID). Values are typically masked integers.
- Output: A map or histogram data structure mapping unique unmasked fragment IDs to their integer counts.

**Data Structures:**
- Voxel Grid: A 3D array or flattened vector of integers. Each element represents a voxel.
- Count Map: A hash map or dense array where keys are fragment IDs and values are counts.

**Steps:**
1. Initialize an empty count map.
2. Iterate through every voxel in the grid.
3. For each voxel, retrieve its raw value.
4. Check if the value is greater than `VOXEL_FREE`. If not, skip to the next voxel.
5. Apply the `unmask()` function to the raw value to extract the actual fragment ID. [S1, unofficial]
6. Increment the count for this fragment ID in the map.
7. After iterating all voxels, return the populated map.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def count_voxel_values(grid, VOXEL_FREE):
    counts = {}
    for voxel in grid:
        if voxel > VOXEL_FREE:
            fragment_id = unmask(voxel)
            if fragment_id in counts:
                counts[fragment_id] += 1
            else:
                counts[fragment_id] = 1
    return counts
```

**Parameters:**
- `VOXEL_FREE`: A constant representing the value for empty/free space. [S1, unofficial]
- `unmask()`: A function that extracts the relevant bits from the voxel value. [S1, unofficial]

**Edge Cases and Pitfalls:**
- Masked Values: Voxel values may be masked, storing metadata alongside the ID. Failing to call `unmask()` before counting will result in incorrect fragment IDs and skewed counts. [S1, unofficial]
- Empty Voxels: Voxels with values $\le$ `VOXEL_FREE` must be excluded to avoid counting background noise. [S1, unofficial]
## Subtopics

### Volume Calculation via Histogramming
This variant uses the output of Voxel Value Counting to estimate the physical volume of objects. By multiplying the count of each fragment ID by the volume of a single voxel, the total volume for each material or fragment can be calculated. It is chosen when quantitative size analysis is required rather than just frequency data. It relies on the same underlying counting mechanism but adds a post-processing step.

## Operating characteristics
**Time Complexity:** $O(N)$, where $N$ is the total number of voxels in the grid. Each voxel is visited exactly once.
**Memory Complexity:** $O(K)$, where $K$ is the number of unique fragment IDs present in the grid. The count map grows with the diversity of values, not the grid size.
**Performance:** Performance is dominated by memory access patterns in the voxel grid. Cache-friendly traversal (e.g., linear scan) is preferred.
**Strengths:** Simple to implement; provides direct statistical data; handles sparse data efficiently if the grid is sparse.
**Limitations:** Does not provide spatial information; sensitive to masking errors; requires a full grid scan.
**When to Use:** When analyzing the distribution of fragment IDs or calculating volumes.
**When Not to Use:** When spatial locality is needed (use [Voxel Filtering](4035-voxel-filtering.md) or ray casting instead).
## Background
This method is part of the voxel processing pipeline in tools like VoxelFragmentML, used for machine learning applications involving voxelized data. It serves as a preprocessing step to extract statistical features from 3D shapes.
## Sources
[S1] AlfonsoLRz/VoxelFragmentML, https://github.com/AlfonsoLRz/VoxelFragmentML, fetched 2026-10-26 (unofficial)
