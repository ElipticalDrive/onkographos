# Interval Collection Embedding

Also called: Integer Embedding of a Collection of Intervals

## Overview
Interval Collection Embedding maps a collection of $N$ intervals on the real line to a discrete integer grid while preserving the relative ordering of all interval endpoints [S1, unofficial]. It solves the problem of compressing sparse interval data into a bounded integer domain $[1, 2N]$ suitable for further algorithmic processing, such as covering problems [S1, unofficial]. It sets itself apart from general compression methods by providing a strict monotonic mapping that guarantees no endpoint collisions and exact order preservation, at the cost of requiring a sorted endpoint structure [S1, unofficial].

## How it works
**Inputs:** A collection $\mathcal{R} = \{[a_i, b_i] : i \in [N]\}$ of $N$ intervals on $\mathbb{R}$.
**Outputs:** A map $\phi: \mathbb{R} \to [1, 2N]$ that is an interval collection embedding.

**Data Structures:**
1.  **Set $S$**: A sorted list of unique endpoints.
    *   Layout: Array of size $|S| \le 2N$.
    *   Content: $c_1 < c_2 < \dots < c_{|S|}$, where $S = \bigcup_{[a_i, b_i] \in \mathcal{R}} \{a_i, b_i\}$ [S1, unofficial].
2.  **Map $\phi$**: Defined by linear interpolation between consecutive elements of $S$.

**Steps:**
1.  **Collect Endpoints**: Extract all $a_i$ and $b_i$ from the input intervals.
2.  **Sort and Deduplicate**: Sort the collected endpoints in ascending order and remove duplicates to form the sequence $c_1, c_2, \dots, c_{|S|}$ [S1, unofficial].
3.  **Define Mapping**: Construct $\phi$ such that for any $j \in [|S|-1]$ and $t \in [0, 1]$:
    $$ \phi((1-t)c_j + t c_{j+1}) = t(j+1) + (1-t)j $$
    Specifically, the endpoints map to integers: $\phi(c_j) = j$ [S1, unofficial].

**Formula:**
The embedding preserves order:
$$ s_1 < s_2 \iff \phi(s_1) < \phi(s_2), \forall s_1, s_2 \in S $$
The range of $\phi$ is $[1, |S|] \subseteq [1, 2N]$ [S1, unofficial].

**Parameters:**
*   $N$: Number of input intervals.
*   Typical values: None specified; depends on input density [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Duplicate Endpoints**: If multiple intervals share endpoints, deduplication in Step 2 is critical to maintain the bijection between sorted unique endpoints and integer indices [S1, unofficial].
*   **Precision**: The source defines the map on $\mathbb{R}$, but the output is integer-indexed. Implementation must handle floating-point comparisons for sorting carefully [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def interval_collection_embedding(intervals):
    """
    Computes the interval collection embedding for a list of intervals.
    Returns a function phi that maps any real number to the embedded coordinate.
    """
    # Step 1 & 2: Collect and sort unique endpoints
    endpoints = set()
    for a, b in intervals:
        endpoints.add(a)
        endpoints.add(b)
    
    sorted_endpoints = sorted(list(endpoints))
    n_unique = len(sorted_endpoints)
    
    # Precompute mapping for endpoints
    endpoint_to_index = {val: i for i, val in enumerate(sorted_endpoints)}
    
    # Step 3: Define linear interpolation map
    # phi(c_j) = j (using 0-based indexing for implementation, 
    # though math definition uses 1-based [1, 2N])
    # For any x between c_j and c_{j+1}:
    # phi(x) = j + (x - c_j) / (c_{j+1} - c_j)
    
    def phi(x):
        # Find interval index j such that c_j <= x < c_{j+1}
        # Using binary search for efficiency
        import bisect
        idx = bisect.bisect_right(sorted_endpoints, x) - 1
        
        # Handle boundary cases
        if idx < 0:
            # x is before the first endpoint
            # Linear extrapolation or clamp? 
            # Source implies domain is R, but mapping is defined by interpolation.
            # Assuming standard linear interpolation behavior or clamping to 0.
            # Given the source focuses on the range [1, 2N], we assume x is within bounds or extrapolate.
            # For strict adherence to "linear interpolation between sorted endpoints",
            # we define it piecewise.
            return 0 # [G] Extrapolation behavior not explicitly defined for x < c_1
        elif idx >= n_unique - 1:
            return n_unique - 1 # [G] Extrapolation behavior not explicitly defined for x >= c_{last}
        
        c_j = sorted_endpoints[idx]
        c_j1 = sorted_endpoints[idx + 1]
        
        if c_j == c_j1:
            return idx # [G] Avoid division by zero if duplicates existed despite set()
        
        t = (x - c_j) / (c_j1 - c_j)
        return idx + t

    return phi
```

## Subtopics

### Box Collection Embedding
A generalization of Interval Collection Embedding to $k$ dimensions. It projects $N$ boxes in $\mathbb{R}^k$ onto each dimension independently, computes a 1D interval collection embedding for each dimension's projections, and combines them to map boxes to $\mathbb{Z}^k$ within $[2N]^k$ [S1, unofficial]. Choose this when embedding multi-dimensional sparse box data while preserving axis-aligned order. It runs in $O(k N \log N)$ time [S1, unofficial].

## Operating characteristics
**Time Complexity:** $O(N \log N)$, dominated by the sorting of $2N$ endpoints [S1, unofficial]. The interpolation mapping construction is $O(N)$ [S1, unofficial].
**Memory Complexity:** $O(N)$ to store the sorted unique endpoints [S1, unofficial].
**Strengths:** Guarantees order preservation; simple linear interpolation; bounded output range $[1, 2N]$ [S1, unofficial].
**Limitations:** Requires sorting; output resolution depends on $N$; not directly applicable to non-interval data without projection [S1, unofficial].
**When to use:** When compressing sparse interval data for algorithms requiring integer coordinates (e.g., covering problems) [S1, unofficial].
**When not to use:** When preserving geometric distances is required (this preserves order, not metric distance) [S1, unofficial].

## Background
Originated in the context of hardness proofs for covering problems, specifically relating to the "Special Rectangle 3SC Problem" and "2-DimIntVGridBoxCover" [S1, unofficial]. It was introduced to embed range space instances into a bounded integer grid to facilitate complexity analysis [S1, unofficial].

## Sources
[S1] Compression of Voxelized Vector Field Data by Boxes is Hard, https://doi.org/10.48550/arxiv.2510.20801, fetched 2026-10-26 (unofficial)
