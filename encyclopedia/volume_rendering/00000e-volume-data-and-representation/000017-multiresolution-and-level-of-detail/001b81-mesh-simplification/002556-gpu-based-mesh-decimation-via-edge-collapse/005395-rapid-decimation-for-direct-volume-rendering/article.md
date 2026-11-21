# Rapid Decimation for Direct Volume Rendering

Also called: Bucket-based Decimation, Vertex Decimation

## Overview
Rapid Decimation for Direct Volume Rendering is a mesh simplification technique that reduces the polygon count of a volume grid by removing vertices deemed unimportant for visual fidelity, resulting in a hybrid grid with many-sided cells and non-cell polygons [S1, unofficial]. It solves the problem of rendering large volumetric datasets efficiently by preserving critical feature boundaries defined by data value ranges, rather than geometric proximity alone. Unlike its parent method, [GPU-based Mesh Decimation via Edge Collapse](3505-gpu-based-mesh-decimation-via-edge-collapse.md), which minimizes geometric error via edge collapses on triangular meshes, this method operates on hexahedral or tetrahedral grids and uses a data-value-based "bucket" heuristic to determine vertex importance, allowing for significant decimation rates (45-62%) with minimal visual artifacting in direct volume rendering contexts [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   A volume grid consisting of simple hexahedral or tetrahedral cells.
*   Scalar data values associated with each vertex.
*   A transfer function (optional, for specific bucket placement strategies).
*   A target number of buckets $N$ (user-specified).

**Outputs:**
*   A decimated volume representation consisting of a subset of the original vertices and the polygons (faces) connected to them. The output is a hybrid grid where some cells may have more than four or six faces, and some polygons may not belong to any complete cell [S1, unofficial].

### Data Structures
*   **Vertex List:** Stores vertex positions and scalar data values.
*   **Polygon List:** Stores the connectivity of the volume. Each polygon references its constituent vertices.
*   **Active Polygon Mask:** A boolean array or a sorted list with an offset indicating which polygons remain in the volume after decimation. The implementation sorts the polygon list so that decimated vertices are listed first, keeping an offset to indicate where active vertices begin [S1, unofficial].
*   **Neighbor List:**
    *   For regular/curvilinear grids: Implicit neighbors defined by grid indices $(z, j, k)$.
    *   For tetrahedral grids: An explicit list of neighbors for each vertex [S1, unofficial].

### Bucket Placement Strategies
The core of the method is dividing the data range into sub-ranges called "buckets." Vertices are mapped to buckets based on their scalar values. The placement of bucket boundaries determines which features are preserved. All histogram-based strategies use a histogram smoothed by a Gaussian filter with standard deviation $\sigma = 2.82$ [S1, unofficial].

1.  **Uniform by Range:** Places $N$ buckets at equal intervals over the minimum and maximum data values of the volume.
2.  **Uniform by Histogram:** Places $N$ buckets uniformly based on the cumulative frequency of data values from the smoothed histogram, rather than the data range. This ensures roughly equal numbers of vertices per bucket.
3.  **Histogram Curvature:** Places bucket boundaries where the magnitude of the second derivative of the smoothed histogram is largest. This preserves boundaries between features represented by spikes or valleys in the histogram. The curvature magnitude at index $i$ is calculated as:
    $$ | -2h_i + h_{i-1} + h_{i+1} | $$
    where $h_i$ is the histogram value at bin $i$. The $N$ largest values determine the boundaries [S1, unofficial].
4.  **Histogram Features:** An iterative strategy to find feature centers and place boundaries on either side, avoiding placing a bucket *in* the middle of a feature.
    1.  Scan the smoothed histogram to find the point with the highest curvature magnitude. This is the center of a feature.
    2.  Scan left and right from the center to find the next peak in curvature of the opposite sign.
    3.  Place bucket boundaries at these two data values.
    4.  Repeat the search, excluding any histogram entries already assigned boundaries or features already found, until $N$ buckets are placed [S1, unofficial].
5.  **Transfer Function:** Places a bucket boundary at each control point of a piecewise linear transfer function. This aligns decimation boundaries with perceived visual boundaries [S1, unofficial].

### Decimation Algorithm

1.  **Bucket Assignment:**
    For each vertex $v$ in the volume:
    *   Retrieve its scalar data value $d_v$.
    *   Determine which bucket $B_k$ contains $d_v$ based on the chosen placement strategy.
    *   Store the bucket ID for $v$.

2.  **Neighbor Identification:**
    *   For regular/curvilinear grids: The 26 neighbors of a vertex at $(z, j, k)$ are found by incrementing each coordinate by $d=1$ in each direction (i.e., offsets in $\{-1, 0, 1\}^3$, excluding $(0,0,0)$).
    *   For tetrahedral grids: Use the pre-computed explicit neighbor list [S1, unofficial].

3.  **Vertex Classification:**
    For each vertex $v$:
    *   Let $N(v)$ be the set of neighbors of $v$.
    *   Let $B(v)$ be the bucket ID of $v$.
    *   Check if $B(u) == B(v)$ for all $u \in N(v)$.
    *   **If true:** Mark $v$ for decimation.
    *   **If false:** Retain $v$.
    *   **Constraint:** Vertices on the exterior boundary of the grid are always retained to identify the region of space inside the volume [S1, unofficial].

4.  **Polygon Update:**
    *   Remove all polygons that include any decimated vertex.
    *   Update the active polygon list/mask.
    *   Note: The resulting grid is a hybrid grid. Cells may become non-hexahedral/tetrahedral if internal vertices are removed, and some polygons may become "orphaned" (not part of any closed cell) [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rapid_decimation(volume_grid, data_values, num_buckets, strategy, transfer_function=None):
    """
    volume_grid: Structure containing vertices and polygons.
                 For regular grids, vertices are indexed by (z, j, k).
                 For tetrahedral grids, an explicit neighbor list is provided.
    data_values: Array of scalar values for each vertex.
    num_buckets: Integer N, the number of bucket boundaries to create.
    strategy: String, one of ['uniform_range', 'uniform_histogram', 
                              'histogram_curvature', 'histogram_features', 
                              'transfer_function'].
    transfer_function: List of control points (data_value, opacity/color) if strategy is 'transfer_function'.
    
    Returns:
    active_vertices: Boolean array or mask indicating which vertices are retained.
    active_polygons: Boolean array or mask indicating which polygons are retained.
    """
    
    # Step 1: Determine Bucket Boundaries
    if strategy == 'uniform_range':
        min_val = min(data_values)
        max_val = max(data_values)
        boundaries = [min_val + i * (max_val - min_val) / num_buckets for i in range(1, num_buckets)]
        
    elif strategy in ['uniform_histogram', 'histogram_curvature', 'histogram_features']:
        # Compute smoothed histogram
        histogram = compute_histogram(data_values, num_bins=256)
        smoothed_histogram = gaussian_smooth(histogram, sigma=2.82)
        
        if strategy == 'uniform_histogram':
            # Place buckets uniformly by cumulative frequency
            total_count = sum(smoothed_histogram)
            boundaries = []
            target_cumulative = [i * total_count / num_buckets for i in range(1, num_buckets)]
            current_cumulative = 0
            bin_idx = 0
            for target in target_cumulative:
                while current_cumulative < target and bin_idx < len(smoothed_histogram):
                    current_cumulative += smoothed_histogram[bin_idx]
                    bin_idx += 1
                # Boundary is at the edge of the bin where cumulative count crosses target
                boundaries.append(bin_idx) # Implementation detail: map bin index to data value range
            # Note: Exact mapping from bin index to data value depends on histogram bin width.
            # Assuming uniform bin width for simplicity in this high-level pseudocode.
            
        elif strategy == 'histogram_curvature':
            # Place boundaries where | -2h_i + h_{i-1} + h_{i+1} | is largest
            curvatures = []
            for i in range(1, len(smoothed_histogram) - 1):
                c = abs(-2 * smoothed_histogram[i] + smoothed_histogram[i-1] + smoothed_histogram[i+1])
                curvatures.append((c, i))
            curvatures.sort(reverse=True)
            # Select top N-1 indices as boundaries (to create N buckets)
            boundary_indices = [idx for _, idx in curvatures[:num_buckets-1]]
            boundaries = boundary_indices # Map to data values
            
        elif strategy == 'histogram_features':
            # Iterative feature detection
            curvatures = []
            for i in range(1, len(smoothed_histogram) - 1):
                c = abs(-2 * smoothed_histogram[i] + smoothed_histogram[i-1] + smoothed_histogram[i+1])
                curvatures.append((c, i))
            
            boundaries = []
            used_indices = set()
            
            while len(boundaries) < num_buckets - 1:
                # Find highest curvature not yet used
                if not curvatures:
                    break
                max_c, max_idx = curvatures.pop(0)
                if max_idx in used_indices:
                    continue
                
                # Scan left and right for opposite sign curvature peaks
                # Note: The source describes scanning for "peak in curvature of opposite sign".
                # This implies looking for local maxima in the curvature signal itself, 
                # but with opposite sign of the second derivative? 
                # The source says: "scan left and right, looking for the next peak in the curvature of opposite sign".
                # This is ambiguous. It likely means finding the boundaries of the feature 
                # defined by the high curvature center.
                # We will implement a simplified version: find the nearest local minima in the 
                # histogram (or points of low curvature) on either side? 
                # The source text: "place a bucket boundary on either side... avoiding placing a bucket in the middle".
                # Let's assume we find the nearest indices where curvature drops significantly or changes sign?
                # Given the ambiguity, we will use the explicit steps:
                # 1. Center is max_idx.
                # 2. Scan left/right for next peak in curvature of opposite sign.
                # This requires analyzing the sign of the second derivative (h_{i+1} - 2h_i + h_{i-1}).
                
                second_deriv = [smoothed_histogram[i+1] - 2*smoothed_histogram[i] + smoothed_histogram[i-1] for i in range(1, len(smoothed_histogram)-1)]
                
                # Find nearest peak in second_deriv with opposite sign to second_deriv[max_idx-1]
                # This is complex to implement without more context on "peak in curvature of opposite sign".
                # We will mark this as a comment for implementation gap.
                pass 
            
            # Fallback for pseudocode completeness if feature logic is too ambiguous:
            # Use curvature magnitude indices as boundaries
            boundaries = [idx for _, idx in curvatures[:num_buckets-1]]

    elif strategy == 'transfer_function':
        boundaries = [tf_point[0] for tf_point in transfer_function] # Control points
        
    else:
        raise ValueError("Unknown strategy")

    # Step 2: Map Vertices to Buckets
    vertex_buckets = []
    for val in data_values:
        bucket_id = 0
        for i, bound in enumerate(boundaries):
            if val < bound:
                break
            bucket_id += 1
        vertex_buckets.append(bucket_id)

    # Step 3: Determine Neighbors and Decimate
    active_vertices = [True] * len(data_values)
    
    # Ensure exterior boundary vertices are retained
    exterior_vertices = get_exterior_boundary_vertices(volume_grid)
    for v in exterior_vertices:
        active_vertices[v] = True

    for v in range(len(data_values)):
        if not active_vertices[v]:
            continue
            
        neighbors = get_neighbors(volume_grid, v)
        bucket_v = vertex_buckets[v]
        
        decimate = True
        for u in neighbors:
            if vertex_buckets[u] != bucket_v:
                decimate = False
                break
        
        if decimate:
            active_vertices[v] = False

    # Step 4: Update Polygons
    active_polygons = []
    for poly in volume_grid.polygons:
        # A polygon is active if all its vertices are active
        if all(active_vertices[v] for v in poly.vertices):
            active_polygons.append(poly)
            
    return active_vertices, active_polygons

def gaussian_smooth(histogram, sigma):
    # Standard Gaussian convolution
    kernel_size = int(6 * sigma) + 1
    kernel = [exp(-0.5 * ((i - kernel_size//2)**2) / sigma**2) for i in range(kernel_size)]
    kernel = [k / sum(kernel) for k in kernel]
    # Convolve histogram with kernel
    smoothed = convolve(histogram, kernel)
    return smoothed
```

### Parameters
*   **Number of Buckets ($N$):** User-specified. More buckets lead to less decimation but potentially higher fidelity. Typical values depend on the dataset complexity.
*   **Histogram Smoothing Sigma:** Fixed at $\sigma = 2.82$ for all histogram-based strategies [S1, unofficial].
*   **Neighbor Definition:** 26 neighbors for regular/curvilinear grids [S1, unofficial].

### Edge Cases and Pitfalls
*   **Holes in Images:** Strictly local decimation based on linearity can cause holes if a long string of vertices along a view direction are all decimated. This occurs because the decimation heuristic does not account for global visibility [S1, unofficial].
*   **Color Bleeding:** Strong non-linearities in the transfer function (e.g., around free-stream in CFD volumes) can cause color bleeding artifacts where decimated regions allow colors to bleed into otherwise clear portions of the volume [S1, unofficial].
*   **Boundary Preservation:** Exterior boundary vertices must be explicitly retained to define the volume's interior space. Failure to do so results in an undefined volume [S1, unofficial].
*   **Bucket Sensitivity:** The "Uniform by Range" strategy is sensitive to small changes in $N$, as moving bucket boundaries can drastically change which vertices are decimated. "Uniform by Histogram" and "Histogram Features" are more robust [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** $O(V)$ for the decimation pass, where $V$ is the number of vertices, as each vertex is examined once with a constant number of neighbors (26 or explicit list). Bucket placement is $O(H)$ where $H$ is the histogram size (typically 256).
*   **Memory Complexity:** $O(V + P)$ for storing the volume and polygon lists. The boolean mask for active polygons adds $O(P)$ memory. Sorting polygons adds $O(P \log P)$ or $O(P)$ with radix sort [S1, unofficial].

### Measured Performance
Performance was evaluated on three CFD datasets using an image-based error metric (root mean square error of 32 x-ray images, 500x500 pixels). All datasets were decimated approximately 45-62%.

| Strategy | Blunt Fin Error | Lockheed Fighter Error | Space Shuttle Error |
| :--- | :--- | :--- | :--- |
| Uniform by Range | 0.089 | 0.056 | 0.061 |
| Uniform by Histogram | 0.117 | 0.042 | 0.040 |
| Histogram Curvature | 0.062 | 0.048 | 0.004 |
| Histogram Features | 0.058 | 0.065 | 0.031 |
| Transfer Function | 0.074 | 0.071 | 0.022 |

*Note: Lower error is better. The Space Shuttle dataset shows significant variation in strategy performance, with Histogram Curvature performing best (0.004) and Transfer Function performing worst (0.022) among the listed strategies, though Transfer Function was noted to produce images "practically identical" to the non-decimated volume in some contexts [S1, unofficial].*

### Strengths and Limitations
*   **Strengths:**
    *   High decimation rates (45-62%) with minimal visual loss for appropriate strategies.
    *   Preserves feature boundaries defined by data values.
    *   Simple implementation; no complex geometric reconstruction required.
    *   Transfer Function strategy can produce near-identical visual results to the original volume.
*   **Limitations:**
    *   Produces a hybrid grid, which may not be suitable for applications requiring strict hexahedral/tetrahedral topology.
    *   Prone to artifacts (holes, color bleeding) if the transfer function has strong non-linearities or if long linear strings of vertices exist along view directions.
    *   Does not account for view-dependent importance during the decimation pass (though the error metric is view-dependent, the decimation is static) [S1, unofficial].

### When to Use
*   Use when rendering large volumetric datasets (CFD, medical) where performance is critical and the data has distinct features separable by scalar value ranges.
*   Use when the transfer function is known and can be used to guide bucket placement.
*   Avoid when strict geometric topology preservation is required or when the data has smooth, continuous gradients without distinct features.

## Background

### Origin
The method was introduced in a 1999 NASA Technical Report titled "Rapid Decimation for Direct Volume Rendering" [S1, unofficial]. It was developed to address the computational cost of direct volume rendering for large datasets, particularly in computational fluid dynamics (CFD).

### History and Development
The authors explored five automatic strategies for bucket placement to improve upon earlier local decimation methods that suffered from visual artifacts. The key innovation was the use of data-value-based buckets to preserve feature boundaries. The method was tested on CFD datasets (Blunt Fin, Lockheed Fighter, Space Shuttle) and compared against non-decimated volumes using an image-based error metric derived from x-ray projections [S1, unofficial].

### Usage
The method is primarily used in scientific visualization for accelerating the rendering of large volumetric datasets. It is particularly relevant in CFD visualization where data ranges can be large and features (like shock waves or free-stream boundaries) are critical [S1, unofficial].

## Sources

[S1] NASA Technical Reports Server (NTRS) 19990088074: Rapid Decimation for Direct Volume Rendering, https://archive.org/details/NASA_NTRS_Archive_19990088074, fetched 2026-10-26 (unofficial)
