# Temporal Sequence Segmentation
Also called: Greedy Temporal Segmentation, Sequence Segmentation, Time-step Clustering

## Overview
Temporal Sequence Segmentation splits a volumetric time sequence into multiple short segments of varying length to facilitate robust clustering and reduce the computational load of subsequent processing steps [S1]. Unlike video shot detection which identifies distinct boundaries, this method groups neighboring time steps based on high similarity and temporal continuity without seeking global distinct boundaries [S1]. It serves as a preprocessing step in the Volume Partitioning Pipeline, preparing data for clustering by reducing the number of units to process [S1].

## How it works
**Inputs:** A volumetric time sequence $S$ consisting of $T$ time steps. Each time step $t$ contains a 3D grid of data values $s_t(u)$, where $u$ is a 3D grid position.
**Outputs:** A set of segments $S_0 \subseteq S$, where each segment contains one or more contiguous time steps. For each segment, a representative time step $r(S_0)$ is identified.
**Parameters:**
- Maximum segment count ($N_{max}$): Determines the minimum number of segments. Set according to available computational resources [S1].
- Cost threshold: The minimum cost at which merging stops. Typically set to the average dissimilarity of the sequence [S1].

**Data Structures:**
- Dissimilarity array $D$: An array of size $T-1$, where $D[t]$ stores the dissimilarity $d_S(t)$ between time step $t$ and $t+1$.
- Segment list: A list of segments, initially containing $T$ single-time-step segments. Each segment tracks its constituent time steps and its current cost.

**Steps:**
1. **Compute Dissimilarity:** Calculate the dissimilarity $d_S(t)$ for all $t \in [1, T-1]$ using the sum of squared intensity differences between adjacent time steps [S1]:
   $$ d_S(t) = \sum_u (s_{t+1}(u) - s_t(u))^2 $$
   where $u$ iterates over all 3D grid positions [S1].

2. **Initialize Segments:** Initialize each time step as its own segment. Set the cost value associated with each segment to zero [S1].

3. **Iterative Merging:** While the number of segments is greater than $N_{max}$:
   a. Identify the pair of neighboring segments with the minimal combined cost. The cost of merging two neighboring segments is defined as the sum of the dissimilarity at their boundary and their individual costs [S1].
   b. Merge these two segments.
   c. Update the cost of the resulting merged segment to this sum [S1].

4. **Threshold Check:** If the number of segments is less than or equal to $N_{max}$, terminate the merging process when the minimum cost of any potential merge exceeds the cost threshold [S1].

5. **Select Representatives:** For each resulting segment $S_0$, determine the representative time step $r(S_0)$ [S1]:
   $$ r(S_0) = \arg \min_{s_i \in S_0} \left| \sum_{s_j \in S_0, j<i} d_S(j) - \sum_{s_j \in S_0, j>i} d_S(j-1) \right| $$
   This minimizes the absolute difference between the cumulative dissimilarity of predecessors and successors within the segment [S1].
   - **Edge Case:** For segments with only two members, the lower time step is chosen as the representative [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def temporal_sequence_segmentation(time_steps, max_segments, cost_threshold):
    """
    time_steps: List of 3D arrays, where time_steps[t] is the volume at time t.
    max_segments: Integer, maximum number of segments to produce.
    cost_threshold: Float, average dissimilarity of the sequence.
    Returns: List of segments, where each segment is a list of time step indices.
    """
    T = len(time_steps)
    if T == 0:
        return []

    # Step 1: Compute dissimilarity
    # d_S(t) is dissimilarity between time step t and t+1
    # Indices for d_S are 0 to T-2, corresponding to boundaries between t and t+1
    dissimilarities = []
    for t in range(T - 1):
        diff_sq_sum = 0.0
        # Sum of squared intensity differences over all grid points u
        # Assuming time_steps[t] and time_steps[t+1] are numpy arrays or similar
        diff = time_steps[t+1] - time_steps[t]
        diff_sq_sum = np.sum(diff ** 2)
        dissimilarities.append(diff_sq_sum)

    # Step 2: Initialize segments
    # Each segment is a list of time step indices
    segments = [[t] for t in range(T)]
    
    # Helper to get boundary dissimilarity between two adjacent segments
    def get_merge_cost(seg1, seg2):
        # seg1 and seg2 are adjacent in time
        # The boundary is between the last time step of seg1 and first of seg2
        last_t = seg1[-1]
        # dissimilarities index: last_t corresponds to boundary between last_t and last_t+1
        boundary_dissim = dissimilarities[last_t]
        cost1 = sum(dissimilarities[t] for t in range(seg1[0], seg1[-1]))
        cost2 = sum(dissimilarities[t] for t in range(seg2[0], seg2[-1]))
        return boundary_dissim + cost1 + cost2

    # Step 3 & 4: Iterative merging
    while len(segments) > max_segments:
        min_cost = float('inf')
        merge_idx = -1
        
        # Find pair of neighboring segments with minimal merge cost
        for i in range(len(segments) - 1):
            cost = get_merge_cost(segments[i], segments[i+1])
            if cost < min_cost:
                min_cost = cost
                merge_idx = i
        
        # If min cost exceeds threshold, stop
        if min_cost > cost_threshold:
            break
            
        # Merge segments at merge_idx and merge_idx+1
        seg1 = segments[merge_idx]
        seg2 = segments[merge_idx+1]
        merged_seg = seg1 + seg2
        segments[merge_idx] = merged_seg
        segments.pop(merge_idx + 1)

    return segments

def get_representative(segment_indices, dissimilarities):
    """
    segment_indices: List of time step indices in the segment.
    dissimilarities: List of d_S(t) values.
    Returns: The index of the representative time step.
    """
    if len(segment_indices) == 2:
        # Edge case: lower time step is chosen
        return segment_indices[0]
    
    # Calculate cumulative dissimilarities
    # Predecessors: d_S(j) for j < i
    # Successors: d_S(j-1) for j > i
    # Note: d_S(j) is boundary between j and j+1
    
    best_idx = -1
    min_diff = float('inf')
    
    for i in segment_indices:
        # Sum of dissimilarities of predecessors within segment
        # Predecessors are time steps j in segment such that j < i
        # The dissimilarity associated with predecessor j is d_S(j)
        pred_sum = 0.0
        for j in segment_indices:
            if j < i:
                pred_sum += dissimilarities[j]
        
        # Sum of dissimilarities of successors within segment
        # Successors are time steps j in segment such that j > i
        # The dissimilarity associated with successor j is d_S(j-1)
        succ_sum = 0.0
        for j in segment_indices:
            if j > i:
                succ_sum += dissimilarities[j-1]
        
        diff = abs(pred_sum - succ_sum)
        if diff < min_diff:
            min_diff = diff
            best_idx = i
            
    return best_idx
```

## Operating characteristics
- **Time Complexity:** Computing dissimilarities is $O(T \cdot N)$, where $N$ is the number of grid points. Merging is $O(T^2)$ in the worst case if implemented naively, but can be optimized with a priority queue to $O(T \log T)$.
- **Memory Complexity:** $O(T \cdot N)$ to store the input volumes and dissimilarities.
- **Strengths:** Reduces computational load for subsequent clustering by grouping similar time steps. Handles continuous temporal data without requiring distinct boundaries.
- **Limitations:** Performance depends on the choice of $N_{max}$ and cost threshold. High dissimilarity between adjacent steps may lead to many small segments.
- **When to use:** When preprocessing volumetric time sequences for clustering, especially when computational resources are limited.
- **When not to use:** When distinct temporal boundaries are required, or when the sequence is very short.

## Background
This method was introduced in the context of result-driven exploration of simulation parameter spaces for visual effects design [S1]. It draws inspiration from video processing methods for generating overviews by dividing input into shots, but adapts the approach for simulation sequences which do not exhibit distinct boundaries [S1]. The method is part of a pipeline that includes spatial partitioning and clustering to facilitate robust analysis of large volumetric datasets [S1].

## Sources
[S1] Result-Driven Exploration of Simulation Parameter Spaces for Visual Effects Design, https://doi.org/10.1109/TVCG.2010.190, fetched 2026-10-31
