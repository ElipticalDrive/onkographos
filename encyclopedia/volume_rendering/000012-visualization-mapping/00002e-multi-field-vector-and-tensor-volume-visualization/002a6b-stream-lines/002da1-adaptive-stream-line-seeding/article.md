# Adaptive Stream Line Seeding

Also called: Adaptive seeding of stream lines

## Overview
Adaptive Stream Line Seeding is a GPU-based algorithm that dynamically increases the density of stream lines in regions of a vector field where flow diverges (such as turbulent areas or vortices) while keeping the density sparse in laminar regions [S1, unofficial]. It distinguishes itself from regular seeding by using an iterative subdivision process driven by a divergence test between neighboring lines, implemented via alternating CUDA kernels for control and evaluation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An initial set of stream lines defined by their vertex positions (`lineVertices`), their starting seeds (`seeds`), and a list of neighboring line pairs (`linePairs`) to be tested for divergence. Parameters include `maxAllowedLineDist` (the threshold for subdivision) and `linesMaxCount` (the maximum capacity of the output buffer).
*   **Output:** Updated lists of line pairs (`outLinePairs`) and a list of new seed positions (`seeds`) for newly generated lines. The algorithm uses static arrays allocated to maximum capacity, managed as dynamic arrays via atomic counters.

**Data Structures:**
*   `linePairs`: Array of `uint2` structures, where each element stores the indices of two neighboring stream lines.
*   `lineVertices`: Array of `float3` structures storing the 3D coordinates of vertices for all lines.
*   `seeds`: Array of `float3` structures storing the starting positions of stream lines.
*   `lineLengths`: Array of `uint` storing the number of vertices per line.
*   `outLinePairs`: Array of `uint2` for the output neighbor list.
*   `outPairsIndex`: Atomic counter tracking the number of valid entries in `outLinePairs`.
*   `outLinesIndex`: Atomic counter tracking the number of valid entries in `seeds`.

**Algorithm Steps:**
The algorithm operates in iterations. In each iteration, a CUDA kernel (`computeLineAdaptiveExtensionKernel`) processes the current set of line pairs to determine if subdivision is required.

1.  **Thread Initialization:** Each thread processes one line pair from the input `linePairs` array. The thread ID is calculated as `id = blockIdx.x * blockDim.x + threadIdx.x`. If `id >= linePairsCount`, the thread exits [S1, unofficial].
2.  **Pair Retrieval:** The thread retrieves the current pair indices `currPair = linePairs[id]` [S1, unofficial].
3.  **Output Pair Reservation:** The thread atomically increments `outPairsIndex` to reserve space in the output pair list. The original pair indices are copied to `outLinePairs[outPairIndex]` to preserve the topology [S1, unofficial].
4.  **Divergence Test:**
    *   Determine the minimum length between the two lines: `count = min(lineLengths[currPair.x], lineLengths[currPair.y])`. If `count < 2`, the thread exits (lines are too short to test) [S1, unofficial].
    *   Sample 32 points along the lines to check for divergence. The step between samples is `count / 32`.
    *   For each sample index `i` (incrementing by `count / 32`), retrieve the vertex positions `v1` and `v2` from `lineVertices`.
    *   Calculate the Euclidean distance `length(v1 - v2)` and track the maximum distance `maxDist` across all 32 samples [S1, unofficial].
5.  **Subdivision Decision:**
    *   If `maxDist < maxAllowedLineDist`, the lines are considered sufficiently close. The thread exits without creating new lines [S1, unofficial].
    *   If `maxDist >= maxAllowedLineDist`, subdivision is triggered. Calculate the number of new lines to insert using the formula:
        $$ newLines = \lfloor \sqrt{\frac{maxDist}{maxAllowedLineDist}} \rfloor $$
        [S1, unofficial]
6.  **Seed Allocation:**
    *   Atomically reserve space for the new seeds by adding `newLines` to `outLinesIndex`. Let `outLineIndex` be the returned value.
    *   Check if `outLineIndex + newLines >= linesMaxCount`. If true, the buffer is full. Atomically decrement `outLinesIndex` by `newLines` and exit [S1, unofficial].
7.  **Seed Generation and Pair Linking:**
    *   Calculate the step vector for new seeds: `seedStep = (seeds[currPair.y] - seeds[currPair.x]) / (newLines + 1)` [S1, unofficial].
    *   Initialize `lastOutPairIndex` to the current `outPairIndex` [S1, unofficial].
    *   Loop `i` from `0` to `newLines - 1`:
        *   Calculate the new seed position: `seeds[outLineIndex + i] = seeds[currPair.x] + (i + 1) * seedStep` [S1, unofficial].
        *   Atomically increment `outPairsIndex` to reserve space for the new neighbor links. Let this new index be `outPairIndex` [S1, unofficial].
        *   Update the neighbor links to connect the new line to its neighbors:
            *   Set `outLinePairs[lastOutPairIndex].y = outLineIndex + i` (link previous left neighbor to new line).
            *   Set `outLinePairs[outPairIndex].x = outLineIndex + i` (set new line as left neighbor).
            *   Set `outLinePairs[outPairIndex].y = currPair.y` (link new line to right neighbor) [S1, unofficial].
        *   Update `lastOutPairIndex = outPairIndex` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for computeLineAdaptiveExtensionKernel
# Note: This is a conceptual translation of the CUDA kernel logic.
# In practice, this runs on GPU with atomic operations.

def compute_line_adaptive_extension(
    max_allowed_line_dist,
    line_pairs,
    line_pairs_count,
    line_vertices,
    vertices_per_line,
    vertices_per_sample,
    line_lengths,
    seeds,
    out_line_pairs,
    out_pairs_index,
    out_lines_index,
    lines_max_count
):
    # Thread ID calculation (conceptual)
    thread_id = get_thread_id()
    
    if thread_id >= line_pairs_count:
        return

    curr_pair = line_pairs[thread_id]
    
    # Reserve space in output pair list
    out_pair_index = atomic_add(out_pairs_index, 1)
    
    # Preserve original pair
    out_line_pairs[out_pair_index].x = curr_pair.x
    out_line_pairs[out_pair_index].y = curr_pair.y
    
    # Determine minimum line length
    count = min(line_lengths[curr_pair.x], line_lengths[curr_pair.y])
    
    if count < 2:
        return

    # Divergence test with 32 samples
    max_dist = 0.0
    step = count // 32
    
    for i in range(0, count, step):
        v1 = line_vertices[curr_pair.x * vertices_per_line + i * vertices_per_sample]
        v2 = line_vertices[curr_pair.y * vertices_per_line + i * vertices_per_sample]
        dist = length(v1 - v2)
        if dist > max_dist:
            max_dist = dist

    if max_dist < max_allowed_line_dist:
        return

    # Calculate number of new lines
    new_lines = floor(sqrt(max_dist / max_allowed_line_dist))

    # Reserve space for new seeds
    out_line_index = atomic_add(out_lines_index, new_lines)
    
    if out_line_index + new_lines >= lines_max_count:
        atomic_add(out_lines_index, -new_lines)
        return

    # Calculate seed step
    seed_step = (seeds[curr_pair.y] - seeds[curr_pair.x]) / (new_lines + 1)
    
    last_out_pair_index = out_pair_index
    
    for i in range(new_lines):
        # Generate new seed
        seeds[out_line_index + i] = seeds[curr_pair.x] + (i + 1) * seed_step
        
        # Reserve space for new pair links
        out_pair_index = atomic_add(out_pairs_index, 1)
        
        # Link new line to neighbors
        out_line_pairs[last_out_pair_index].y = out_line_index + i
        out_line_pairs[out_pair_index].x = out_line_index + i
        out_line_pairs[out_pair_index].y = curr_pair.y
        
        last_out_pair_index = out_pair_index
```

**Parameters:**
*   `maxAllowedLineDist`: The maximum Euclidean distance between corresponding points on two neighboring stream lines before subdivision is triggered. [S1, unofficial]
*   `linesMaxCount`: The maximum number of stream lines that can be stored in the output buffer. [S1, unofficial]

**Edge Cases and Pitfalls:**
*   **Buffer Overflow:** If the pre-allocated buffer for lines (`linesMaxCount`) is insufficient, the algorithm atomically rolls back the seed index and skips the subdivision for that pair [S1, unofficial].
*   **Short Lines:** Lines with fewer than 2 vertices are skipped to prevent division by zero or invalid sampling [S1, unofficial].
*   **Performance:** The algorithm can be slow at the beginning of iterations because the number of new lines grows exponentially ($2^n$) in highly divergent regions. The square-root formula helps mitigate this by limiting the number of inserted lines relative to the distance [S1, unofficial].
*   **CPU-GPU Synchronization:** The algorithm requires alternating invocation of the control kernel (this one) and the evaluation kernel (stream line integration) by the CPU, which adds overhead compared to fully GPU-resident non-adaptive methods [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The divergence test is $O(N)$ per pair where $N$ is the number of samples (fixed at 32). The subdivision step is $O(K)$ where $K$ is the number of new lines. The overall complexity depends on the number of iterations and the growth rate of lines in turbulent regions.
*   **Memory Complexity:** Requires pre-allocation of static arrays for the maximum possible number of lines and pairs. This can lead to significant memory usage if the scene has large turbulent regions, as the buffer must be sized for the worst-case scenario [S1, unofficial].
*   **Performance:** The use of atomic operations for dynamic array management on GPU is performance-costly due to contention. The alternating CPU-GPU kernel invocation further reduces efficiency compared to non-adaptive versions, though the visual quality improvement is significant [S1, unofficial].
*   **Strengths:** Effectively reveals flow features in turbulent/vortical regions without wasting resources on laminar areas.
*   **Limitations:** High memory overhead due to pre-allocation; slower performance due to atomic operations and CPU synchronization; complexity in implementation [S1, unofficial].

## Background
The method was described by Marek Fiser in a blog post detailing the implementation of realtime 3D vector field visualization using CUDA [S1, unofficial]. The approach addresses the challenge of implementing adaptive algorithms on GPUs, where thread communication and dynamic memory management are difficult. It was designed to improve visual quality over regular seeding by focusing computational effort on areas of high flow divergence [S1, unofficial].

## Sources
[S1] Realtime visualization of 3D vector fields using CUDA, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
