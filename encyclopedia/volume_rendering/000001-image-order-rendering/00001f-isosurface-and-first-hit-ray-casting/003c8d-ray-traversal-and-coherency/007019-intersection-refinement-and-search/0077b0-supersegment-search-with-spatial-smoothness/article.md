# Supersegment Search with Spatial Smoothness

Also called: Adjacency Search, Finding the first supersegment on a ray, Optimized supersegment search

## Overview
Supersegment Search with Spatial Smoothness is an algorithm for finding the first supersegment intersected by a ray in a sorted list $\mathbb{L}_i$ by leveraging the spatial coherence of volume data [S1]. It minimizes memory accesses and thread divergence by using the index $p$ of the last intersected supersegment in the previous list $\mathbb{L}_h$ as an initial guess [S1]. Unlike standard binary search which seeds at the middle, or linear search which starts at the beginning, this method prunes the search space based on the depth boundaries of the candidate supersegment and its predecessor [S1]. It yields up to 40% better frame rates compared to the best-performing baseline (linear or binary search) for finding the first supersegment [S1].

## How it works
**Inputs:**
*   $d_{entry}$: Entry intercept of the ray in $\mathbb{L}_i$ (in NDC) [S1].
*   $d_{exit}$: Exit intercept of the ray in $\mathbb{L}_i$ [S1].
*   $p$: Index of the last supersegment intersected in the previous list $\mathbb{L}_h$ [S1]. If no supersegment was intersected in $\mathbb{L}_h$, $p$ is the index of the closest supersegment from the previous exit point [S1].
*   $maxSupersegments$: Total number of supersegments in the current list $\mathbb{L}_i$ [S1].
*   $b(S_j)$: Function returning the back-depth (exit boundary) of supersegment $S_j$ [S1].
*   $f(S_j)$: Function returning the front-depth (entry boundary) of supersegment $S_j$ [S1].

**Outputs:**
*   $index$: The index $j$ of the first supersegment $S_j$ intersected in $\mathbb{L}_i$. Returns -1 if no such supersegment exists [S1].
*   $supseg\_found$: Boolean flag indicating if a valid supersegment was found [S1].

**Data Structures:**
The supersegments in $\mathbb{L}_i$ are sorted by position [S1]. The algorithm assumes access to the depth boundaries $b(S_j)$ and front depths $f(S_j)$ for any index $j$ within $[0, maxSupersegments - 1]$ [S1].

**Algorithm Steps:**
1.  If $\mathbb{L}_i$ is the first list intersected by the ray, perform a binary search from index 0 to $maxSupersegments - 1$ to find the first intersected supersegment [S1].
2.  If not the first list, retrieve $p$ [S1].
3.  Initialize $bin\_search\_end = -1$ and $bin\_search\_start = -1$ [S1].
4.  If $p$ is negative (invalid):
    *   Set $bin\_search\_end = maxSupersegments - 1$ [S1].
    *   Set $bin\_search\_start = 0$ [S1].
5.  Else ($p$ is valid):
    *   Pre-fetch $b_1 = b(S_p^i)$ if $S_p^i$ exists, else $\infty$ [S1].
    *   Pre-fetch $b_0 = b(S_{p-1}^i)$ if $S_{p-1}^i$ exists, else $-\infty$ [S1].
    *   Calculate $interval = (b_1 \ge d_{entry}) + (b_0 \ge d_{entry})$ [S1].
    *   If $interval$ is 0:
        *   The intersecting supersegment is behind $S_p^i$ [S1].
        *   Set $bin\_search\_end = maxSupersegments - 1$ [S1].
        *   Set $bin\_search\_start = p + 1$ [S1].
    *   Else if $interval$ is 2:
        *   The intersecting supersegment is in front of $S_{p-1}^i$ [S1].
        *   Set $bin\_search\_end = p - 1$ [S1].
        *   Set $bin\_search\_start = 0$ [S1].
    *   Else ($interval$ is 1):
        *   The entry point $d_{entry}$ lies between $b_0$ and $b_1$ [S1].
        *   If $b_1 < \infty$ (i.e., $S_p^i$ exists):
            *   Set $supseg\_found = true$ [S1].
            *   Set $index = p$ [S1].
            *   Set $depthEnd = b_1$ [S1].
            *   Return $index$ [S1].
        *   Else ($S_p^i$ does not exist):
            *   Set $bin\_search\_end = p - 1$ [S1].
            *   Set $bin\_search\_start = 0$ [S1].
6.  If $bin\_search\_end \neq -1$:
    *   Call $BinS(d_{entry}, bin\_search\_start, bin\_search\_end)$ to get $index$ [S1].
    *   $BinS(d, start, stop)$ is a binary search function returning an index $j$ such that $b(S_{j-1}) < d < b(S_j)$, limited to $start \le j \le stop$ [S1].
7.  Get $depthStart = f(S_{index})$ [S1].
8.  If $depthStart > d_{exit}$:
    *   The ray passes between supersegments without intersecting any [S1].
    *   Set $supseg\_found = false$ [S1].
    *   Set $index = -1$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def find_first_supersegment(d_entry, d_exit, p, max_supersegments, b_func, f_func):
    """
    Finds the first supersegment intersected in list L_i.
    
    Args:
        d_entry: Entry intercept of the ray in L_i (in NDC).
        d_exit: Exit intercept of the ray in L_i.
        p: Index of the last supersegment intersected in the previous list L_h.
           If no supersegment was intersected in L_h, p is the index of the 
           closest supersegment from the previous exit point.
        max_supersegments: Total number of supersegments in the current list L_i.
        b_func: Function returning the back-depth (exit boundary) of a supersegment.
        f_func: Function returning the front-depth (entry boundary) of a supersegment.
        
    Returns:
        index: The index of the first intersected supersegment, or -1 if none.
        supseg_found: Boolean indicating if a valid supersegment was found.
    """
    bin_search_end = -1
    bin_search_start = -1
    
    if p < 0:
        # Invalid p, search full range
        bin_search_end = max_supersegments - 1
        bin_search_start = 0
    else:
        # Pre-fetch boundaries for S_p and S_{p-1}
        # Handle boundary conditions where indices might be out of bounds
        if 0 <= p < max_supersegments:
            b1 = b_func(p)
        else:
            b1 = float('inf')
            
        if 0 <= p - 1 < max_supersegments:
            b0 = b_func(p - 1)
        else:
            b0 = float('-inf')
            
        # Calculate interval based on where d_entry lies relative to b0 and b1
        # interval = 0: d_entry < b0 and d_entry < b1 (behind S_p)
        # interval = 2: d_entry >= b0 and d_entry >= b1 (in front of S_{p-1})
        # interval = 1: d_entry is between b0 and b1
        interval = (1 if b1 >= d_entry else 0) + (1 if b0 >= d_entry else 0)
        
        if interval == 0:
            # Ray is behind S_p
            bin_search_end = max_supersegments - 1
            bin_search_start = p + 1
        elif interval == 2:
            # Ray is in front of S_{p-1}
            bin_search_end = p - 1
            bin_search_start = 0
        else:
            # Ray is between b0 and b1
            if b1 < float('inf'):
                # S_p exists and contains the entry point
                return p, True
            else:
                # S_p does not exist, search in front of p
                bin_search_end = p - 1
                bin_search_start = 0
                
    index = -1
    supseg_found = False
    
    if bin_search_end != -1:
        # Perform binary search in the pruned range
        index = binary_search(d_entry, bin_search_start, bin_search_end, b_func)
        
    # Verify the found supersegment actually intersects the ray
    if 0 <= index < max_supersegments:
        depth_start = f_func(index)
        if depth_start > d_exit:
            # Ray passes between supersegments
            supseg_found = False
            index = -1
        else:
            supseg_found = True
    else:
        supseg_found = False
        
    return index, supseg_found

def binary_search(d, start, stop, b_func):
    """
    Binary search for the first supersegment S_j such that b(S_{j-1}) < d < b(S_j).
    Limited to start <= j <= stop.
    """
    low = start
    high = stop
    result = -1
    
    while low <= high:
        mid = (low + high) // 2
        
        # Check if S_mid is the candidate
        # We need b(S_{mid-1}) < d < b(S_mid)
        
        # Get b(S_{mid-1})
        if mid - 1 >= 0:
            b_prev = b_func(mid - 1)
        else:
            b_prev = float('-inf')
            
        # Get b(S_mid)
        b_curr = b_func(mid)
        
        if b_prev < d < b_curr:
            result = mid
            break
        elif d <= b_prev:
            # d is in front of S_mid, search lower indices
            high = mid - 1
        else:
            # d is behind S_mid, search higher indices
            low = mid + 1
            
    return result
```

**Edge Cases and Pitfalls:**
*   If $p$ is invalid (negative), the search defaults to the full list range [S1].
*   If the determined supersegment's front-depth is greater than the ray's exit distance ($d_{exit}$), the ray passes between supersegments; index is set to -1 and $supseg\_found$ to false [S1].
*   If $S_p^i$ does not exist, $b_1$ is set to $\infty$ and a binary search is performed [S1].
*   If $S_{p-1}^i$ does not exist, $b_0$ is set to $-\infty$ [S1].
*   The algorithm assumes the angle between the intersecting ray and the original ray is less than $90^\circ$, but is applicable in both cases [S1].

## Operating characteristics
*   **Time Complexity:** The algorithm reduces memory accesses compared to a simple binary search seeded at the middle element [S1]. In the best case (when $S_p^i$ is the correct supersegment), it is $O(1)$ [S1]. In the worst case, it performs a binary search, which is $O(\log N_S)$ [S1].
*   **Measured Performance:** Algorithm 1 yields up to 40% better frame rates compared to the best-performing baseline (linear or binary search) for finding the first supersegment [S1].
*   **Strengths:** Minimizes memory accesses and thread divergence by leveraging spatial smoothness [S1]. Reduces the search space for binary search by using the previous list's result as a seed [S1].
*   **Limitations:** Relies on the assumption that neighboring lists have similar supersegment sizes (spatial smoothness) [S1]. If this assumption is violated, the performance gain may be reduced.
*   **When to use:** When rendering volumetric depth images (VDI) or similar volume data where rays are processed in lists and spatial coherence is high [S1].
*   **When not to use:** When the volume data lacks spatial smoothness or when the ray lists are not processed in a coherent order [S1].

## Background
The method was introduced in "Efficient Raycasting of Volumetric Depth Images for Remote Visualization of Large Volumes at High Frame Rates" by Gupta and Sbalzarini (2023) [S1]. It is part of a pipeline that uses the fast voxel traversal algorithm by Amanatides and Woo to determine intersected lists $\mathbb{L}$ and their entry/exit points [S1]. The algorithm is designed to optimize the search for supersegments within these lists, which are used to compute opacity accumulation [S1].

## Sources
[S1] Efficient Raycasting of Volumetric Depth Images for Remote Visualization of Large Volumes at High Frame Rates, https://doi.org/10.1109/pacificvis56936.2023.00014, fetched 2026-10-24
