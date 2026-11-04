# Partitioned Distance Map Combination


## Overview
Partitioned Distance Map Combination is the runtime assembly phase that merges a selected subset of precomputed partitioned distance maps into a single global distance map for empty space skipping [S1, unofficial]. It uses a ping-pong buffer strategy with two swap textures to iteratively combine maps, allowing more maps to be processed than can be bound simultaneously as shader inputs [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of precomputed partitioned distance maps (`texturePartitionedDistanceMaps`), a boolean array indicating which maps are active for the current transfer function (`partitionedDistanceMapsToUse`), and the occupancy map dimensions (`sizeOccupancyMap`) [S1, unofficial].
*   **Output:** A single combined distance map stored in `textureIntermediateDistanceMapOutput` [S1, unofficial].

**Data Structures:**
*   `textureSwapArray`: An array of two textures with format `GL_R8UI` (8-bit unsigned integer, single channel) and `GL_READ_WRITE` access, acting as ping-pong buffers [S1, unofficial].
*   `texturePartitionedDistanceMaps`: An array of textures, each representing a distance map for a specific intensity partition, accessed with `GL_READ_ONLY` [S1, unofficial].
*   `partitionedDistanceMapsToUse`: A boolean array indicating which partitioned maps are currently relevant [S1, unofficial].
*   `UBO`: An 8-byte Uniform Buffer Object used to pass control flags to the compute shader [S1, unofficial].

**Parameters:**
*   `numMapsToCombinePerLaunch`: The maximum number of partitioned maps to combine in a single compute dispatch. The source specifies a constant value of 6 [S1, unofficial].

**Numbered Steps:**
1.  **Count Active Maps:** Iterate through `partitionedDistanceMapsToUse` to calculate `numMapsToCombineTotal`, the total number of active partitioned maps [S1, unofficial].
2.  **Calculate Iterations:** Compute `numIterations` as $\lceil \text{numMapsToCombineTotal} / \text{numMapsToCombinePerLaunch} \rceil$ [S1, unofficial].
3.  **Iterative Combination Loop:** For each iteration `j` from 0 to `numIterations - 1`:
    a.  **Bind Swap Buffers:** Bind `textureSwapArray[j % 2]` to image unit 0 and `textureSwapArray[(j + 1) % 2]` to image unit 1 with `GL_READ_WRITE` access [S1, unofficial].
    b.  **Determine Maps to Combine:** Calculate `numMapsToUse` as $\min(\text{numMapsToCombinePerLaunch}, \text{numMapsToCombineTotal} - j \times \text{numMapsToCombinePerLaunch})$ [S1, unofficial].
    c.  **Bind Partitioned Maps:** Loop through `i` from 0 to `numMapsToUse - 1`. Skip inactive maps in `partitionedDistanceMapsToUse` by incrementing `idxMapProcessed` until a `true` value is found. Bind the corresponding `texturePartitionedDistanceMaps[idxMapProcessed]` to image unit `2 + i` with `GL_READ_ONLY` access. Increment `idxMapProcessed` [S1, unofficial].
    d.  **Setup UBO:** Create an 8-byte UBO. Set the first 4 bytes to `useDistanceValueFromPreviousPass` (1 if `j > 0`, else 0). Set the next 4 bytes to `numMapsToUse`. Bind this UBO to binding point 0 [S1, unofficial].
    e.  **Dispatch Compute:** Calculate work groups based on `sizeOccupancyMap` dimensions divided by 8. Dispatch the compute shader with dimensions `(numWorkGroups[0], numWorkGroups[1], numWorkGroups[2])`. Issue a memory barrier to ensure completion [S1, unofficial].
4.  **Final Copy:** After the loop, copy the final result from the active swap buffer to `textureIntermediateDistanceMapOutput` using `glCopyImageSubData` with dimensions from `sizeOccupancyMap` [S1, unofficial].
5.  **Cleanup:** Delete the UBO [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct

# Inputs: partitionedDistanceMapsToUse (list of bool), sizeOccupancyMap (tuple of 3 ints)
# Outputs: combined distance map (conceptual)
# Constants: numMapsToCombinePerLaunch = 6

numMapsToCombineTotal = sum(1 for active in partitionedDistanceMapsToUse if active)
numIterations = (numMapsToCombineTotal + numMapsToCombinePerLaunch - 1) // numMapsToCombinePerLaunch

idxMapProcessed = 0
for j in range(numIterations):
    read_buffer_idx = j % 2
    write_buffer_idx = (j + 1) % 2

    remaining_maps = numMapsToCombineTotal - j * numMapsToCombinePerLaunch
    numMapsToUse = min(numMapsToCombinePerLaunch, remaining_maps)

    for i in range(numMapsToUse):
        while not partitionedDistanceMapsToUse[idxMapProcessed]:
            idxMapProcessed += 1
        # Bind texturePartitionedDistanceMaps[idxMapProcessed] to image unit 2 + i
        idxMapProcessed += 1

    # Setup UBO (8 bytes)
    use_prev = 1 if j > 0 else 0
    ubo_data = struct.pack('II', use_prev, numMapsToUse)
    # Bind UBO to binding point 0

    # Calculate work groups
    numWorkGroups = tuple((sizeOccupancyMap[k] + 7) // 8 for k in range(3))

    # Dispatch compute shader with numWorkGroups
    # Issue memory barrier
    pass

# Copy final result from textureSwapArray[(numIterations - 1) % 2] to output
# Cleanup UBO
```

**Edge Cases and Pitfalls:**
*   **Zero Active Maps:** If `numMapsToCombineTotal` is 0, `numIterations` is 0, and the loop is skipped. The output distance map may need to be initialized to a maximum value to indicate no occupied space [G].
*   **Precision:** The use of `GL_R8UI` implies 8-bit unsigned integer distance values, which may lead to quantization errors or overflow if distances exceed 255 [S1, unofficial].
*   **Boundary Conditions:** The compute shader must handle boundary checks correctly when reading from partitioned maps and writing to the swap buffers [G].

## Subtopics

### [Partitioned Distance Map (PDM) Acceleration](1286-partitioned-distance-map-pdm-acceleration.md)
Direct Volume Rendering (DVR) with Partitioned Distance Maps (PDM) based Empty Space Skipping (ESS) is an acceleration technique for distance-field ray marching that subdivides the volume's intensity range into $n$ partitions to precompute individual distance maps. When a transfer function (TF) changes, only the relevant PDMs are merged to form the final distance map $D'$, avoiding a full recomputation of the distance transform. Unlike the base method [Partitioned Distance Map Combination](1191-partitioned-distance-map-combination.md) which recomputes the global distance map on every TF change, PDM-based ESS trades higher memory usage and one-time initialization cost for significantly faster TF update times.

## Operating characteristics
*   **Time Complexity:** The combination process scales with the number of active partitioned maps and the volume size, with iterations bounded by $\lceil N / 6 \rceil$ where $N$ is the number of active maps [S1, unofficial].
*   **Memory Complexity:** Requires storing $n$ partitioned distance maps plus two swap buffers, resulting in $O(n \cdot V)$ memory usage where $V$ is the number of voxels [S1, unofficial].
*   **Strengths:** Avoids full distance transform recomputation on transfer function changes [S1, unofficial].
*   **Limitations:** Higher memory usage due to storing multiple partitioned maps and precision limitations due to 8-bit distance maps [S1, unofficial].

## Background
The method originates from an unofficial implementation of Partitioned Distance Map based Empty Space Skipping for Direct Volume Rendering [S1, unofficial]. It builds on the principles of Distance Map Acceleration [972-distance-map-acceleration.md] and Occupancy Map Generation [1197-occupancy-map-generation.md].

## Sources
[S1] CalamityMichL/DVR_ESS_PDM, https://github.com/CalamityMichL/DVR_ESS_PDM, fetched 2026-10-26 (unofficial)
