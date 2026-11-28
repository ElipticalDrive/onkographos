# Fixed-Size Neighborhood Reconstruction

Also called: Fixed-Size Neighbor Gradient

## Overview
Fixed-Size Neighborhood Reconstruction is a gradient estimation method that approximates the continuous gradient $\nabla f(\mathbf{v})$ using a linear combination of a fixed number $K$ of neighboring samples, rather than all available connectivity neighbors [S1]. It solves the problem of encoding irregular vertex-to-vertex connectivity in GPU texture memory, enabling efficient on-the-fly computation for dynamic or time-varying meshes [S1]. Unlike full-connectivity methods, it trades off some accuracy for regular memory access patterns and reduced storage overhead [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of scalar values $f(\mathbf{v}_i)$ at discrete sample points (vertices) $\mathbf{v}_i$, and a pre-selected subset of $K$ neighbors for each query vertex.
*   **Output:** The estimated gradient vector $\nabla f(\mathbf{v})$ at the query vertex.

**Data Structures:**
The method relies on a fixed-size neighborhood structure encoded in a 2D texture for GPU efficiency [S1].
*   **Neighbor Texture:** A 2D texture where each texel (or block of texels) corresponds to a vertex. It stores up to $K$ neighbor indices (or positions) per vertex. The layout is regular, allowing standard texture sampling [S1].
*   **Scalar Field Texture:** Stores the scalar values $f(\mathbf{v}_i)$ at the vertex locations [S1].

**Numbered Steps:**
1.  **Neighbor Selection:** For each vertex, select $K$ neighbors from the available connectivity. The selection criteria significantly impact accuracy [S1]. Three ranking methods are defined:
    *   **Unsorted:** Neighbors are chosen randomly or in arbitrary order [S1].
    *   **Distance:** Neighbors are ranked in decreasing order of distance from the query vertex (closest first) [S1].
    *   **Inverse Distance:** Neighbors are ranked in increasing order of distance (closest first), effectively weighting closer neighbors more heavily in the selection process [S1].
2.  **Preprocessing (Optional):** For static data, a prepass can compute gradients. This involves traversing all vertices in parallel by drawing a quadrilateral covering the vertex texture. A shader implements the regression on the fixed-size neighborhood and writes results to a gradient texture [S1].
3.  **On-the-Fly Computation (Alternative):** For dynamic data, the regression is included directly in the volume rendering shader. This avoids storing a gradient texture but increases per-pixel computation cost [S1].
4.  **Gradient Estimation:** Compute the gradient using linear regression on the $K$ selected neighbors. The weights $w_i$ are derived from the basis function derivative, but in the fixed-size context, they are often determined by the regression fit to the subset [S1].

**Formulas:**
The gradient is estimated as a linear combination of the scalar values of the $K$ neighbors:
$$ \nabla f(\mathbf{v}) \approx \sum_{i=1}^{K} w_i \cdot f(\mathbf{v}_i) $$
where $w_i$ are the regression weights [S1]. The weights are chosen to minimize the error between the estimated gradient and the finite differences implied by the neighbor positions [S1].

**Parameters:**
*   **$K$ (Fixed Size):** The number of neighbors to encode.
    *   $K=4$: Lowest quality, fastest computation [S1].
    *   $K=8$: Medium quality [S1].
    *   Typical values are small integers (4 or 8) to fit within texture limits and maintain performance [S1].

**Edge Cases and Pitfalls:**
*   **Irregular Connectivity:** Vertex-to-vertex connectivity is irregular and costly to encode in textures. Fixed-size neighborhoods solve this by enforcing a regular 2D texture layout [S1].
*   **Selection Bias:** The order of neighbor selection matters. Inverse distance ranking generally produces more accurate estimates than naive distance or random selection for $K > 3$ [S1].
*   **Outliers:** Using a subset of neighbors (e.g., 4 or 8) with inverse distance ranking can yield higher accuracy than using the full neighborhood, suggesting that distant neighbors may act as outliers [S1].
*   **Small $K$:** For $K=3$, unsorted neighbors produced a better estimate than distance-ranked neighbors in tested grid datasets [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient_fixed_size(vertex_pos, neighbor_indices, scalar_field, K):
    """
    Computes gradient at vertex_pos using K nearest neighbors.
    
    Args:
        vertex_pos: (x, y, z) position of the query vertex.
        neighbor_indices: List of K indices into the scalar field.
        scalar_field: Array of scalar values f(v_i).
        K: Number of neighbors (fixed).
        
    Returns:
        gradient: (gx, gy, gz) estimated gradient vector.
    """
    # Step 1: Retrieve neighbor positions and scalar values
    # Note: In GPU implementation, this is done via texture lookups
    neighbors = []
    for i in range(K):
        idx = neighbor_indices[i]
        pos = get_vertex_position(idx) # Helper to get 3D position from index
        val = scalar_field[idx]
        neighbors.append((pos, val))
        
    # Step 2: Perform linear regression to find gradient weights
    # We solve for gradient g in: f(v_i) ≈ f(v_0) + g . (v_i - v_0)
    # This is a least-squares problem: min || A*g - b ||^2
    # where A is a matrix of displacement vectors (v_i - v_0),
    # and b is the vector of scalar differences (f(v_i) - f(v_0))
    
    A = []
    b = []
    for pos, val in neighbors:
        displacement = pos - vertex_pos
        A.append(displacement)
        b.append(val - scalar_field[neighbor_indices[0]]) # Reference to first neighbor
        
    # Solve least squares: g = (A^T A)^-1 A^T b
    # Note: For K < 3 in 3D, the system is underdetermined. 
    # The source notes imply K >= 3 is used.
    # Implementation detail: Use a standard linear algebra solver.
    
    gradient = solve_least_squares(A, b)
    
    return gradient

def select_neighbors(vertex_pos, all_neighbors, K, method='inverse_distance'):
    """
    Selects K neighbors based on ranking method.
    
    Args:
        vertex_pos: Query vertex position.
        all_neighbors: List of all connected neighbor indices.
        K: Number of neighbors to select.
        method: 'unsorted', 'distance', or 'inverse_distance'.
        
    Returns:
        selected_indices: List of K indices.
    """
    if method == 'unsorted':
        # Random or arbitrary selection
        return all_neighbors[:K]
        
    elif method == 'distance':
        # Rank by distance, closest first
        dists = [(get_distance(vertex_pos, get_vertex_position(idx)), idx) for idx in all_neighbors]
        dists.sort(key=lambda x: x[0])
        return [idx for _, idx in dists[:K]]
        
    elif method == 'inverse_distance':
        # Rank by inverse distance (closest first), same as distance ranking
        # Source notes suggest 'inverse distance' ranking produces better results
        # than 'distance' ranking, implying a specific weighting or selection nuance.
        # However, the text says "inverse distance, which performs the inverse ranking"
        # relative to "distance, which ranks them in decreasing order based on distance".
        # "Decreasing order based on distance" usually means farthest first? 
        # Let's re-read carefully: "distance, which ranks them in decreasing order based on distance"
        # and "inverse distance, which performs the inverse ranking".
        # If distance ranks farthest first, inverse distance ranks closest first.
        # Standard nearest neighbor is closest first.
        # The text says: "inverse distance ranking... produces more accurate estimates than naive (distance) or random".
        # And "nearest neighbors should contribute more".
        # So 'distance' likely means sorting by distance descending (farthest first) or ascending?
        # "decreasing order based on distance" -> Distance values decrease? No, that's ascending.
        # "decreasing order" usually means largest to smallest.
        # So 'distance' = Farthest to Closest.
        # 'inverse distance' = Closest to Farthest.
        
        dists = [(get_distance(vertex_pos, get_vertex_position(idx)), idx) for idx in all_neighbors]
        # Sort by distance ascending (closest first) for inverse distance ranking
        dists.sort(key=lambda x: x[0])
        return [idx for _, idx in dists[:K]]
        
    return all_neighbors[:K]
```

## Subtopics

### Precomputed Gradient Texture
A variant where gradients are computed in a prepass and stored in a texture. This saves rendering time but requires extra GPU memory [S1].

### On-the-Fly Gradient Reconstruction
Computing gradients directly in the volume rendering shader. This saves memory but increases rendering cost [S1].

## Operating characteristics
*   **Time Complexity:** $O(K)$ per vertex for gradient computation, where $K$ is small (4 or 8) [S1].
*   **Memory Complexity:** $O(K \cdot N)$ for storing neighbor indices, where $N$ is the number of vertices [S1]. This is significantly less than storing full connectivity for irregular meshes [S1].
*   **Performance:** Enables interactive exploration of time-varying data sets with low computation cost [S1].
*   **Strengths:** Efficient GPU encoding via regular 2D textures; suitable for dynamic meshes; reduces memory requirements compared to full connectivity [S1].
*   **Limitations:** Accuracy depends on neighbor selection; may be less accurate than full-connectivity methods due to outlier inclusion or insufficient sampling [S1].
*   **When to Use:** Dynamic or time-varying meshes where precomputing gradients is not feasible; scenarios with limited GPU memory; interactive exploration where approximate gradients are acceptable [S1].
*   **When Not to Use:** Static meshes where high-accuracy gradients are critical and memory is not a constraint; scenes with highly irregular scalar fields where neighbor selection is difficult [S1].

## Background
The method was introduced in the context of volume rendering on unstructured meshes, specifically to address the challenges of encoding irregular vertex-to-vertex connectivity for gradient estimation [S1]. It builds on regression methods for gradient reconstruction, which approximate the gradient as a linear combination of neighbor values [S1]. The approach was evaluated for its suitability in GPU-assisted volume rendering, particularly for dynamic data [S1].

## Sources
[S1] A Comparison of Gradient Estimation Methods for Volume Rendering on Unstructured Meshes, https://www.computer.org/csdl/journal/tg/2011/03/ttg2011030305/13rRUx0xPi5, fetched 2026-11-01
