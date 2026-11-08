# Shuffled Morton Order

Also called: Morton-shuffled vertex order, Optimized vertex order, Shuffled Morton, Shuffled Morton Order Vertex Reordering

## Overview
Shuffled Morton Order is a vertex reordering policy for point clouds that combines Morton code sorting with batch shuffling to improve GPU cache locality and load balancing across Graphics Processing Clusters (GPCs) [S1]. It solves the problem of performance instability and contention inherent in pure spatial sorting by ensuring uniform distribution of localized groups, providing high rendering performance with low sensitivity to viewpoint changes [S1]. Unlike pure Morton order which may cause excessive locality, or pure shuffled order which causes cache thrashing, this hybrid approach balances spatial coherence with hardware utilization [S1].

## How it works
**Inputs and Outputs**
- Input: A list of $N$ points, each with 3D coordinates $(x, y, z)$ and associated attributes (e.g., color, size).
- Output: A reordered list of points in a vertex buffer layout optimized for `GL_POINTS` rendering.

**Data Structures**
- The vertex buffer is a contiguous array of point structures.
- Each point structure contains at least position $(x, y, z)$ and color $(r, g, b, a)$.
- The layout is created by sorting points according to Morton code, then shuffling the resulting sequence in batches.

**Steps**
1. **Morton Code Calculation**: For each point $(x, y, z)$, compute its Morton code (Z-order curve index). This maps the 3D coordinate to a 1D integer that preserves spatial locality.
2. **Morton Sorting**: Sort all $N$ points in the point cloud according to their computed Morton codes. This groups spatially close points together in the buffer.
3. **Batch Division**: Divide the sorted sequence into contiguous batches of size $B$. The standard batch size is $B = 128$ points [S1].
4. **Batch Shuffling**: Shuffle the positions of these batches within the vertex buffer. The internal order of points within each batch is preserved. This moves spatially close groups to random locations in memory, preventing excessive locality that leads to contention or imbalanced workload across GPCs [S1].

**Parameters**
- Batch size $B$: 128 points [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random
import math

def shuffle_morton_order(points):
    """
    Reorders points using Shuffled Morton Order.
    
    Args:
        points: List of tuples (x, y, z, r, g, b, a)
        
    Returns:
        List of tuples in shuffled Morton order
    """
    if not points:
        return []
        
    # Step 1 & 2: Sort by Morton Code
    # Morton code interleaves bits of x, y, z coordinates.
    # Assuming coordinates are normalized or quantized to integers.
    def morton_code(x, y, z):
        # Placeholder for actual Morton encoding logic
        # Interleave bits of x, y, z
        return (x | (y << 1) | (z << 2)) # Simplified for illustration; real Morton code uses bit interleaving
        
    # Sort points by Morton code
    sorted_points = sorted(points, key=lambda p: morton_code(p[0], p[1], p[2]))
    
    # Step 3: Divide into batches
    batch_size = 128
    batches = []
    for i in range(0, len(sorted_points), batch_size):
        batches.append(sorted_points[i:i + batch_size])
        
    # Step 4: Shuffle batches
    random.shuffle(batches)
    
    # Flatten the shuffled batches
    shuffled_points = []
    for batch in batches:
        shuffled_points.extend(batch)
        
    return shuffled_points
```

**Edge Cases and Pitfalls**
- **Precision**: Morton code calculation requires integer coordinates or quantized floating-point values. Precision loss during quantization can affect spatial locality.
- **Boundary Conditions**: If $N$ is not a multiple of $B$, the last batch will be smaller. This is handled naturally by the slicing operation.
- **Degenerate Input**: Empty point clouds return empty lists. Single-point clouds remain unchanged.

## Subtopics

### [Morton Code Spatial Sorting](4818-morton-code-spatial-sorting.md)
Sorts Gaussians by Morton code to improve spatial locality for compression. Shuffled Morton Order builds on this by adding batch shuffling to mitigate hardware contention.

### [Shuffled Order](52-shuffled-order.md)
Pure random shuffling of points. Shuffled Morton Order improves upon this by preserving local spatial coherence within batches.

## Operating characteristics
- **Time Complexity**: Sorting takes $O(N \log N)$ time. Shuffling batches takes $O(N/B)$ time. Overall complexity is dominated by sorting, $O(N \log N)$.
- **Memory Complexity**: $O(N)$ to store the reordered points.
- **Performance**: Boosts the efficiency of `GL_POINTS` by a factor of 5x compared to other orderings in cases where hardware rendering is compulsory [S1]. Achieves rates of 62-64 fps for 796 million points on an RTX 3090 [S1].
- **Strengths**: High rendering performance, low sensitivity to viewpoint changes, reduced fluctuations across different compute-based methods [S1].
- **Limitations**: Requires pre-processing to reorder points. Not suitable for dynamic point clouds where points move frequently, as reordering overhead may outweigh benefits.
- **When to Use**: Static or slowly changing point clouds rendered with `GL_POINTS` on hardware pipelines.
- **When Not to Use**: Dynamic point clouds with frequent updates, or when using compute-based rendering where other optimizations may be more effective.

## Background
Shuffled Morton Order was introduced in "Rendering Point Clouds with Compute Shaders and Vertex Order Optimization" [S1]. It addresses the limitations of pure Morton sorting, which can cause contention or imbalanced workload due to excessive locality, and pure shuffled order, which causes cache thrashing. The method was developed to optimize hardware rendering of large point clouds, particularly for `GL_POINTS` primitives. It has been evaluated on NVIDIA RTX 3090 and RTX 2070 Super GPUs, demonstrating significant performance improvements over baseline orderings.

## Sources
[S1] Rendering Point Clouds with Compute Shaders and Vertex Order Optimization, https://doi.org/10.1111/cgf.14345, fetched 2026-10-26
