# Adaptive Stream Line Seeding

Also called: Adaptive Line Extension, Adaptive seeding of stream lines, Streamline Subdivision

## Overview
Adaptive Stream Line Seeding is a GPU-based algorithm that dynamically refines the density of stream lines in 3D vector fields by subdividing regions where neighboring lines diverge beyond a threshold [S1, unofficial]. Unlike static seeding methods that distribute seeds uniformly or based on pre-computed scalar fields, this method iteratively inserts new seeds between existing diverging line pairs to capture turbulent or vortex structures while maintaining sparsity in laminar flow areas [S1, unofficial]. It builds on [Stream Lines](4796-stream-lines.md) by adding a control phase that manages seed placement based on geometric divergence.

## How it works
The algorithm operates in an iterative loop managed by the CPU, alternating between a control kernel (which identifies diverging line pairs and generates new seeds) and an evaluation kernel (which integrates the stream lines) [S1, unofficial]. The control phase uses atomic operations to manage dynamic arrays within fixed-size GPU buffers, allowing the number of lines to grow until a capacity limit is reached [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- `linePairs`: Array of `uint2` structures, where each element contains indices of two neighboring stream lines to be tested for divergence [S2, unofficial].
- `lineVertices`: Array of `float3` structures containing the vertex positions of all current stream lines, packed sequentially [S2, unofficial].
- `lineLengths`: Array of `uint` values, storing the number of vertices in each stream line [S2, unofficial].
- `seeds`: Array of `float3` structures, storing the initial seed position for each stream line [S2, unofficial].
- `maxAllowedLineDist`: Scalar threshold for the maximum distance between corresponding points of two lines before subdivision is triggered [S1, unofficial].
- `verticesPerLine`: Constant, the number of vertices per stream line [S2, unofficial].
- `verticesPerSample`: Constant, the stride (in vertices) between sampled points along a line for the divergence test [S2, unofficial].
- `linesMaxCount`: Constant, the maximum capacity of the output line buffer [S2, unofficial].

**Outputs:**
- `outLinePairs`: Array of `uint2` structures, storing the updated list of neighboring line pairs for the next iteration [S2, unofficial].
- `outPairsIndex`: Atomic counter, tracking the number of pairs written to `outLinePairs` [S2, unofficial].
- `outLinesIndex`: Atomic counter, tracking the number of lines written to the output buffer [S2, unofficial].
- `seeds`: Updated array of `float3` structures, containing both original and newly inserted seeds [S2, unofficial].

### Data Structures
The implementation uses static arrays allocated to a maximum capacity to avoid dynamic memory allocation overhead during kernel execution [S1, unofficial].
- `linePairs` and `outLinePairs`: `uint2*` arrays. Each `uint2` holds two indices into the `lineVertices` buffer.
- `lineVertices`: `float3*` array. Layout is contiguous: `[Line0_V0, Line0_V1, ..., Line1_V0, ...]`.
- `lineLengths`: `uint*` array. Index corresponds to line index.
- `seeds`: `float3*` array. Index corresponds to line index.
- `outLinesIndex`, `outPairsIndex`: `uint*` device pointers used as atomic counters.

### Algorithm Steps
The core logic resides in the `computeLineAdaptiveExtensionKernel`. Each thread processes one pair of lines from `linePairs`.

1.  **Thread Identification**: Calculate global thread ID `id`. If `id >= linePairsCount`, terminate [S2, unofficial].
2.  **Pair Retrieval**: Get the current pair indices `currPair = linePairs[id]` [S2, unofficial].
3.  **Pair Preservation**: Atomically allocate an index in `outLinePairs` using `atomicAdd(outPairsIndex, 1)`. Store the original pair indices in `outLinePairs[outPairIndex]` to preserve the input structure for the next iteration [S2, unofficial].
4.  **Length Check**: Calculate `count = min(lineLengths[currPair.x], lineLengths[currPair.y])`. If `count < 2`, terminate (lines are too short to sample) [S2, unofficial].
5.  **Divergence Sampling**: Initialize `maxDist = 0`. Iterate through the lines using a stride of `count / 32` to sample exactly 32 points along the shorter line [S1, unofficial].
    -   For each sample index `i`, retrieve vertices:
        $$v_1 = \text{lineVertices}[currPair.x \times \text{verticesPerLine} + i \times \text{verticesPerSample}]$$
        $$v_2 = \text{lineVertices}[currPair.y \times \text{verticesPerLine} + i \times \text{verticesPerSample}]$$
    -   Update `maxDist`:
        $$\text{maxDist} = \max(\text{maxDist}, \|v_1 - v_2\|)$$
6.  **Threshold Test**: If `maxDist < maxAllowedLineDist`, terminate (lines are sufficiently close) [S2, unofficial].
7.  **Calculate New Lines**: Compute the number of new lines to insert using the square root formula [S1, unofficial]:
    $$\text{newLines} = \left\lfloor \sqrt{\frac{\text{maxDist}}{\text{maxAllowedLineDist}}} \right\rfloor$$
8.  **Atomic Allocation for Lines**: Atomically allocate space for new seeds in the output buffer:
    $$\text{outLineIndex} = \text{atomicAdd}(\text{outLinesIndex}, \text{newLines})$$
    If `outLineIndex + newLines >= linesMaxCount`, rollback the allocation (`atomicAdd(outLinesIndex, -newLines)`) and terminate (buffer full) [S2, unofficial].
9.  **Seed Interpolation**: Calculate the step vector between the seeds of the diverging pair:
    $$\text{seedStep} = \frac{\text{seeds}[currPair.y] - \text{seeds}[currPair.x]}{\text{newLines} + 1}$$
10. **Insert New Seeds and Pairs**: Iterate `i` from `0` to `newLines - 1`:
    -   Set new seed position:
        $$\text{seeds}[\text{outLineIndex} + i] = \text{seeds}[currPair.x] + (i + 1) \times \text{seedStep}$$
    -   Atomically allocate space in `outLinePairs` for the new neighbor relationships:
        $$\text{outPairIndex} = \text{atomicAdd}(\text{outPairsIndex}, 1)$$
    -   Link the previous pair's end to the new seed:
        $$\text{outLinePairs}[\text{lastOutPairIndex}].y = \text{outLineIndex} + i$$
    -   Link the new seed to the current pair's end:
        $$\text{outLinePairs}[\text{outPairIndex}].x = \text{outLineIndex} + i$$
        $$\text{outLinePairs}[\text{outPairIndex}].y = currPair.y$$
    -   Update `lastOutPairIndex = outPairIndex` [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# CUDA Kernel Logic (Python-like representation)
def computeLineAdaptiveExtensionKernel(
    maxAllowedLineDist, linePairs, linePairsCount,
    lineVertices, verticesPerLine, verticesPerSample,
    lineLengths, seeds, outLinePairs,
    outPairsIndex, outLinesIndex, linesMaxCount
):
    id = blockIdx.x * blockDim.x + threadIdx.x
    if id >= linePairsCount:
        return

    currPair = linePairs[id]
    
    # Preserve original pair
    outPairIndex = atomicAdd(outPairsIndex, 1)
    outLinePairs[outPairIndex] = currPair

    count = min(lineLengths[currPair.x], lineLengths[currPair.y])
    if count < 2:
        return

    # Sample 32 points
    maxDist = 0
    step = count // 32
    for i in range(0, count, step):
        v1 = lineVertices[currPair.x * verticesPerLine + i * verticesPerSample]
        v2 = lineVertices[currPair.y * verticesPerLine + i * verticesPerSample]
        dist = length(v1 - v2)
        if dist > maxDist:
            maxDist = dist

    if maxDist < maxAllowedLineDist:
        return

    # Calculate number of new lines
    newLines = floor(sqrt(maxDist / maxAllowedLineDist))

    # Allocate space for new seeds
    outLineIndex = atomicAdd(outLinesIndex, newLines)
    if outLineIndex + newLines >= linesMaxCount:
        atomicAdd(outLinesIndex, -newLines)
        return

    # Calculate seed step
    seedStep = (seeds[currPair.y] - seeds[currPair.x]) / (newLines + 1)

    lastOutPairIndex = outPairIndex
    for i in range(newLines):
        # Set new seed
        seeds[outLineIndex + i] = seeds[currPair.x] + (i + 1) * seedStep
        
        # Allocate pair index
        outPairIndex = atomicAdd(outPairsIndex, 1)
        
        # Link previous pair end to new seed
        outLinePairs[lastOutPairIndex].y = outLineIndex + i
        # Link new seed to current pair end
        outLinePairs[outPairIndex].x = outLineIndex + i
        outLinePairs[outPairIndex].y = currPair.y
        
        lastOutPairIndex = outPairIndex
```

### Parameters
- `maxAllowedLineDist`: The maximum Euclidean distance between corresponding points of two neighboring stream lines. If the maximum sampled distance exceeds this value, subdivision is triggered [S1, unofficial]. Typical values depend on the scale of the vector field and the desired visual density [S1, unofficial].
- `verticesPerSample`: Stride between sampled points in the `lineVertices` buffer. Determined by `count / 32` to ensure exactly 32 samples are taken along the line [S2, unofficial].

### Edge Cases and Pitfalls
- **Buffer Overflow**: The algorithm uses atomic operations to manage a fixed-size buffer. If `linesMaxCount` is reached, new lines are discarded, and the counter is rolled back [S2, unofficial]. This may result in incomplete refinement in highly turbulent regions if the capacity is too low.
- **Short Lines**: Lines with fewer than 2 vertices are skipped to prevent division by zero or invalid sampling [S2, unofficial].
- **Performance**: The algorithm is slower than non-adaptive versions due to the overhead of atomic operations and the iterative CPU-GPU synchronization [S1, unofficial]. The number of new lines grows exponentially in the early iterations ($2^n$), which can cause initial slowdowns [S1, unofficial].
- **Precision**: Floating-point precision in distance calculations and seed interpolation may affect the accuracy of subdivision in very fine-scale features [G].

## Subtopics

### [Adaptive Streamline Generation](7692-adaptive-streamline-generation.md)
Adaptive Streamline Generation is an iterative CUDA-based algorithm that generates streamlines by recursively extending existing lines and seeding new ones based on flow divergence. It differs from static seeding by dynamically refining the visualization in turbulent or vortex regions while maintaining sparsity in laminar areas, building on the principles of [Adaptive Stream Line Seeding](5604-adaptive-stream-line-seeding.md).

### [Entropy-based Seeding](14471-entropy-based-seeding.md)
Entropy-based Seeding is a method for selecting streamline seed locations in 3D vector fields by using Shannon’s entropy to measure local information content. It places seeds proportionally to entropy values to quantitatively capture critical flow features, such as turbulent cores, ensuring important regions are densely sampled. It differs from sibling methods like [Adaptive Stream Line Seeding](5604-adaptive-stream-line-seeding.md) by deriving seed density directly from an information-theoretic scalar field rather than geometric divergence or control kernels.

### Multi-Seed Streamline Tracking
Multi-Seed Streamline Tracking involves tracking multiple seed points simultaneously to overcome ambiguities in depth when setting seed positions from a Maximum Intensity Projection (MIP) alone [S3, unofficial]. Seeds can be placed in a 26-neighbourhood of voxels around a central seed point, or all points in the data volume can be seeded at a specified step interval [S3, unofficial]. While seeding the entire volume is slow, it provides a better impression of flow connectivity than static tensor displays [S3, unofficial]. Arrow heads can be added to the start and end of individual stream lines for display purposes [S3, unofficial]. For finer discrimination, start points can be seeded with greater precision on the cut-view display window [S3, unofficial].

## Operating characteristics
- **Time Complexity**: The control kernel runs in $O(N)$ time, where $N$ is the number of line pairs processed per iteration [S1, unofficial]. The number of iterations depends on the convergence of the divergence metric, which can be unpredictable in highly turbulent flows [S1, unofficial].
- **Memory Complexity**: Memory usage is dominated by the pre-allocated buffers for `lineVertices`, `seeds`, and `linePairs`, sized to `linesMaxCount` [S1, unofficial]. This static allocation avoids dynamic memory overhead but requires careful estimation of the maximum number of lines needed [S1, unofficial].
- **Performance**: The algorithm is performance-costly compared to non-adaptive versions due to the need for CPU-GPU synchronization and atomic operations [S1, unofficial]. However, the overall slowdown is described as "not very significant" relative to the improvement in visual quality [S1, unofficial].
- **Strengths**: Effectively captures complex flow features like vortices and turbulence by concentrating lines where they are needed most [S1, unofficial].
- **Limitations**: Requires prior knowledge of the maximum number of lines to allocate memory [S1, unofficial]. The iterative nature and atomic operations can lead to contention and reduced GPU utilization [S1, unofficial].

## Background
The method was developed for real-time visualization of 3D vector fields using CUDA [S1, unofficial]. The primary challenge addressed was the difficulty of implementing adaptive algorithms on GPUs due to the cost of thread communication and dynamic memory management [S1, unofficial]. The solution uses a split-kernel approach (control and evaluation) invoked alternately by the CPU to manage the adaptive process [S1, unofficial]. The technique was demonstrated in the context of visualizing MRI vector and tensor fields, where accurate representation of flow connectivity is critical [S3, unofficial].

## Sources
[S1] Realtime visualization of 3D vector fields using CUDA | Marek's blog, https://marekfiser.com/projects/realtime-visualization-of-3D-vector-fields-using-CUDA/, fetched 2026-10-28 (unofficial)
[S2] NightElfik/Vector-field-visualization-using-cuda NightElfik/Vector-field-visualization-using-cuda on GitHub, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
[S3] Angiotool : a tool for interactive visualization of MRI vector and tensor fields, https://doi.org/10.82444/warw.33642268, fetched 2026-10-26 (unofficial)
