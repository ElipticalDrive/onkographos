# Inverse Distance Weighted Interpolation

Also called: 3D Inverse Distance Weighting (IDW) Interpolation, Inverse Distance Weighting

## Overview
Inverse Distance Weighted (IDW) Interpolation is a spatial interpolation method that estimates the value of a continuous scalar field at a query point by computing a weighted average of nearby sample values, where the influence of each sample decreases as its distance from the query point increases [S1][S2, unofficial]. It is chosen when a smooth, non-linear reconstruction is required from sparse or irregular data points (such as drill-hole assays) and when the computational cost of higher-order schemes like tri-linear or tri-cubic interpolation is prohibitive or when the underlying data structure is not a regular grid [S1][S2, unofficial]. Unlike [Trilinear Interpolation](1531-trilinear-interpolation.md), which assumes a regular grid and guarantees $C^0$ continuity along interior faces, IDW is grid-agnostic and does not guarantee continuity across cell boundaries unless extended to neighboring cells [S1].

## How it works
**Inputs:**
- A set of $N$ sample points $\{P_i\}$ with associated scalar values $\{V_i\}$, where $P_i = (x_i, y_i, z_i)$ and $V_i$ is the value at $P_i$.
- A query point $Q = (x_q, y_q, z_q)$.
- Parameters:
  - `power`: The IDW exponent controlling the rate of decay. Typical value: 2 [S2, unofficial].
  - `searchRadius`: The maximum Euclidean distance to consider samples. Typical value: 60 world units [S2, unofficial].
  - `maxNeighbors`: The maximum number of nearest neighbors to include. Typical value: 16 [S2, unofficial].

**Outputs:**
- The interpolated scalar value $V(Q)$.

**Data Structures:**
- A spatial index (e.g., 3D Hash Grid, KD-Tree, or Octree) to efficiently retrieve neighbors within `searchRadius` or the `maxNeighbors` closest points [S2, unofficial].
- If using the cell-based approach described in [S1], the data is structured as a grid of hexahedral cells, and the query point is first located within a specific cell [S1].

**Steps:**

1. **Neighbor Retrieval**:
   - Find all sample points $P_i$ such that the Euclidean distance $d_i = \|Q - P_i\|$ is less than or equal to `searchRadius`.
   - If `maxNeighbors` is specified, select only the $k \le \text{maxNeighbors}$ points with the smallest $d_i$ from the retrieved set [S2, unofficial].
   - If no samples are found within the radius, return a nodata sentinel value (e.g., $< -10^{29}$) [S2, unofficial].

2. **Exact Hit Check**:
   - If any sample point $P_j$ coincides exactly with $Q$ (i.e., $d_j = 0$), return $V_j$ immediately [S1][S2, unofficial].

3. **Weight Calculation**:
   - For each neighbor $i$, compute the weight $w_i$ using the inverse distance formula:
     $$
     w_i = \frac{1}{d_i^{\text{power}}}
     $$
   - Note: If using the specific cell-based formulation from [S1] (where neighbors are the 8 vertices of the containing cell), the weights are computed as:
     $$
     w_i = \frac{\prod_{k=0, k \neq i}^{7} [d_k(Q)]^2}{\sum_{j=0, j \neq i}^{7} \prod_{l=0, l \neq j}^{7} [d_l(Q)]^2}
     $$
     where $d_k(Q)$ is the Euclidean distance between the $k$-th vertex of the cell and $Q$ [S1]. This formulation ensures that if $Q$ is at vertex $j$, $w_j=1$ and all other $w_i=0$ [S1].

4. **Normalization and Interpolation**:
   - Compute the sum of weights $W = \sum_{i} w_i$.
   - Compute the interpolated value:
     $$
     V(Q) = \frac{\sum_{i} w_i V_i}{W}
     $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def idw_interpolate(query_point, samples, spatial_index, power=2, search_radius=60, max_neighbors=16):
    """
    query_point: tuple (x, y, z)
    samples: list of tuples (point, value)
    spatial_index: object with query_radius and query_k_nearest methods
    power: float, IDW exponent
    search_radius: float, max distance to consider
    max_neighbors: int, max number of neighbors to use
    """
    # Step 1: Retrieve neighbors
    # Try to get k-nearest neighbors first if max_neighbors is small and efficient
    # Otherwise, get all within radius and sort/filter
    candidates = spatial_index.query_radius(query_point, search_radius)
    
    if not candidates:
        return -1e30  # Nodata sentinel [S2, unofficial]
    
    # If we have more candidates than max_neighbors, select the closest ones
    if len(candidates) > max_neighbors:
        # Sort by distance to query_point
        candidates.sort(key=lambda s: distance(query_point, s.point))
        candidates = candidates[:max_neighbors]
    
    # Step 2: Exact Hit Check
    for sample in candidates:
        if distance(query_point, sample.point) == 0:
            return sample.value
    
    # Step 3 & 4: Weight Calculation and Interpolation
    total_weight = 0.0
    weighted_sum = 0.0
    
    for sample in candidates:
        d = distance(query_point, sample.point)
        # Avoid division by zero, though exact hit check above should catch d=0
        if d == 0:
            return sample.value
        
        w = 1.0 / (d ** power)
        total_weight += w
        weighted_sum += w * sample.value
        
    if total_weight == 0:
        return -1e30 # Should not happen if candidates exist and d > 0
        
    return weighted_sum / total_weight

def distance(p1, p2):
    return ((p1[0]-p2[0])**2 + (p1[1]-p2[1])**2 + (p1[2]-p2[2])**2)**0.5
```

**Edge Cases and Pitfalls:**
- **Continuity**: The basic scheme does not guarantee $C^0$ continuity along interior faces of the grid [S1]. This can be minimized by including all vertices of all cells that contain a point on an interior face, effectively extending the neighbor set beyond the single containing cell [S1].
- **Local Sparsity**: Considering only surrounding cells ignores contributions from cells in the original volume that do not include the interpolated point, which can lead to sparse sampling artifacts [S1].
- **Arbitrary Hexahedra**: If the underlying cells are arbitrary hexahedra, vertices may be far from the query point, potentially degrading accuracy compared to regular grids [S1].

## Operating characteristics
- **Time Complexity**: $O(N)$ for naive search, or $O(\log N + k)$ for spatial index search, where $N$ is the total number of samples and $k$ is the number of neighbors (typically small, e.g., 8-16) [S2, unofficial].
- **Memory Complexity**: $O(N)$ to store the sample points and the spatial index [S2, unofficial].
- **Performance**: Predictable performance for static datasets using hash grids or KD-trees [S2, unofficial]. The method is computationally lightweight per query compared to tri-cubic interpolation, making it suitable for real-time or interactive visualization of sparse data [S2, unofficial].
- **Strengths**: Works with irregular, non-grid data; simple to implement; computationally efficient; handles sparse data well [S1][S2, unofficial].
- **Limitations**: Does not guarantee continuity across cell boundaries; sensitive to the choice of `power` and `searchRadius`; may produce "bullseye" artifacts around sample points if `power` is too high [S1][S2, unofficial].
- **When to use**: When data is sparse, irregular, or not on a regular grid; when real-time performance is critical; when a smooth, non-linear interpolation is desired without the complexity of higher-order schemes [S1][S2, unofficial].
- **When not to use**: When high-order continuity is required; when the data is on a regular grid and higher-order interpolation (like [Trilinear Interpolation](1531-trilinear-interpolation.md)) is feasible and preferred for accuracy [S1].

## Background
Inverse Distance Weighted Interpolation was originally proposed for distance-weighted interpolation schemes in the context of volume rendering and spatial data analysis [S1]. The specific weighting scheme used in the cell-based implementation is presented in [BS84] [S1]. It has been applied to computational fluid dynamics simulations, such as the "blunt fin" from NASA-Ames Research Center [S1]. More recently, it has been implemented in web-based 3D visualization libraries for rendering sparse assay data (e.g., gold assays) as volumetric textures [S2, unofficial].

## Sources
[S1] Direct volume rendering of curvilinear volumes, https://doi.org/10.1145/99308.99318, fetched 2026-10-28
[S2] feat: 3D IDW interpolation volume and voxel visualisation, https://github.com/darkmine-oss/baselode/pull/50, fetched 2026-10-27 (unofficial)
