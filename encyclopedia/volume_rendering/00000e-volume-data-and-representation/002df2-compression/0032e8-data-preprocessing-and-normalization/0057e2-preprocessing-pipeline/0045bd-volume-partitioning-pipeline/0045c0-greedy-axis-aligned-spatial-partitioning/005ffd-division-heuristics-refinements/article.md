# Division Heuristics Refinements

Also called: Binary Search Division, Recursive Load Recalculation, Multi-Array Workload Approximation

## Overview
Division Heuristics Refinements are minor algorithmic optimizations applied to the greedy axis-aligned spatial partitioning process to improve load-balancing accuracy and computational efficiency during volume data subdivision. These refinements address approximation errors inherent in linear division schemes by introducing binary search for cut location, recursive load recalculation, and multi-array workload tracking. Unlike the primary greedy heuristic which minimizes bounding box volume, these refinements focus on minimizing load imbalance errors introduced during the partitioning phase, though they have a lesser impact on overall runtime compared to earlier rasterization-stage refinements.

## How it works
The refinements modify the `FIND-DIVISION` procedure within the spatial partitioning pipeline. The goal is to determine the optimal split position along a chosen axis for a given region.

**Inputs:**
- A spatial region $R$ defined by its axis-aligned bounding box.
- A set of work items (e.g., triangles or volume primitives) $W$ contributing to the load within $R$.
- A target load balance metric (e.g., equal distribution between two resulting sub-regions).

**Outputs:**
- A split plane position $p$ along the chosen axis.
- Two new sub-regions $R_{left}$ and $R_{right}$.

**Data Structures:**
- **Workload Arrays:** Instead of a single scalar value for the region's total load, use multiple arrays to store workload contributions of individual items. This allows for finer-grained approximation of load distribution [S1, unofficial].
- **Region State:** Each region maintains a current estimated workload value.

**Steps:**
1. **Initialize Search Range:** Define the search interval $[L, R]$ for the split position along the chosen axis, bounded by the region's extent.
2. **Binary Search for Cut:** Perform a binary search within $[L, R]$ to find the optimal split position $p$.
   - At each step, calculate the midpoint $m = (L + R) / 2$.
   - Estimate the workload of the left sub-region $W_{left}$ and right sub-region $W_{right}$ if split at $m$.
   - Adjust $L$ or $R$ based on which side has higher load, aiming to minimize $|W_{left} - W_{right}|$.
   - This speeds up the division process compared to linear scanning [S1, unofficial].
3. **Recursive Load Recalculation:** After a division is performed, immediately re-calculate the workload for each resulting sub-region.
   - This reduces approximation errors that accumulate when using static or pre-computed load estimates [S1, unofficial].
   - The re-calculation uses the multi-array workload contributions for better precision.
4. **Recursive FIND-DIVISION:** Use recursion in the `FIND-DIVISION` procedure rather than a linear iterative approach.
   - This avoids cumulative errors introduced by linear division schemes, where small errors in one split propagate to subsequent splits [S1, unofficial].
   - The recursion ensures that each split is optimized based on the current, updated state of the sub-regions.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def find_division_recursive(region, items, axis):
    """
    Recursively find the optimal division for a region using binary search
    and updated workload estimates.
    """
    # Base case: if region is small enough or cannot be split further
    if region.is_leaf() or not region.can_split(axis):
        return None

    # Define search bounds along the axis
    low = region.min(axis)
    high = region.max(axis)
    
    # Binary search for optimal split
    while high - low > epsilon:
        mid = (low + high) / 2.0
        
        # Estimate workload for left and right parts if split at mid
        # This uses multiple arrays for item contributions for better approximation
        left_workload = estimate_workload(region, items, axis, mid, side='left')
        right_workload = estimate_workload(region, items, axis, mid, side='right')
        
        # Adjust search bounds to balance load
        if left_workload < right_workload:
            low = mid
        else:
            high = mid
            
    split_pos = (low + high) / 2.0
    
    # Perform the division
    left_region, right_region = region.split(axis, split_pos)
    
    # Re-calculate workload for each new region to reduce approximation errors
    left_region.workload = calculate_total_workload(left_region, items)
    right_region.workload = calculate_total_workload(right_region, items)
    
    # Return the split position and updated regions
    return split_pos, left_region, right_region

def estimate_workload(region, items, axis, split_pos, side):
    """
    Estimate workload for one side of the split using multi-array contributions.
    """
    total = 0.0
    for item in items:
        if item.intersects_side(region, axis, split_pos, side):
            # Sum contributions from multiple arrays for better approximation
            total += item.get_workload_contribution()
    return total
```

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point precision in binary search may lead to non-optimal splits if `epsilon` is too large or too small. [G] A typical `epsilon` value for spatial coordinates is $10^{-6}$ to $10^{-9}$ depending on the volume resolution.
- **Degenerate Input:** If all items are concentrated in one sub-region, the binary search may converge to the boundary. [G] Handle this by checking for zero-width regions and skipping division.
- **Cumulative Errors:** Linear division can accumulate errors; recursion mitigates this but increases call stack depth. [S1, unofficial]

## Subtopics

### [Recursive Heuristic](18040-recursive-heuristic.md)
The Recursive Heuristic is a load-balancing strategy for parallel volume rendering that recursively partitions the image plane into smaller rectangular regions until each processor manages a single region. It serves as a general superset of simpler 1D horizontal and vertical division schemes, dynamically choosing between horizontal and vertical cuts at each recursion level to optimize load balance. Unlike fixed-axis partitioning, it adapts the division orientation based on the computed load distribution, offering greater scalability flexibility at the cost of increased data exchange and loss of scanline coherency.

## Operating characteristics
- **Time Complexity:** The binary search reduces the time to find the optimal split from $O(N)$ (linear scan) to $O(\log N)$ per division step, where $N$ is the number of items in the region. However, the recursive nature and repeated workload calculations add overhead.
- **Memory Complexity:** Using multiple arrays for workload contributions increases memory usage compared to a single scalar value. The recursion depth adds stack space proportional to the number of divisions.
- **Performance Impact:** These refinements do not significantly affect the overall runtime or results of the algorithm compared to earlier refinements (e.g., rasterization-stage optimizations) [S1, unofficial]. Their primary benefit is improved load balance accuracy, which can lead to better parallel efficiency in distributed rendering.
- **Strengths:** Reduces approximation errors, speeds up division via binary search, avoids cumulative errors via recursion.
- **Limitations:** Increased complexity, higher memory usage for multi-array tracking, minimal impact on final rendering quality compared to other optimizations.

## Background
These refinements were proposed in the context of parallel volume rendering and spatial partitioning for load balancing. They represent minor but practical improvements to greedy axis-aligned spatial partitioning algorithms, focusing on reducing the error in workload estimation during the partitioning phase. The approach is documented in a thesis from Bilkent University, which discusses various refinements for parallel rendering systems [S1, unofficial].

## Sources
[S1] ;Г, ,1:. 'i f ,3 Ψ |'̂*i, ̂ t ѴУ t ̂ -.̂ 'j f'\ ' ‘'̂ '· îM. Vl ■I ·' !■' Í; 1 ! é .1 li ifi; .· ̂'■ '’(■H ' ■'“' ’"ΐ ΐ'-ΐ ...., https://repository.bilkent.edu.tr/server/api/core/bitstreams/a7dc03c6-3640-4028-b1b3-d6296aa2d23c/content, fetched 2026-10-30 (unofficial)
