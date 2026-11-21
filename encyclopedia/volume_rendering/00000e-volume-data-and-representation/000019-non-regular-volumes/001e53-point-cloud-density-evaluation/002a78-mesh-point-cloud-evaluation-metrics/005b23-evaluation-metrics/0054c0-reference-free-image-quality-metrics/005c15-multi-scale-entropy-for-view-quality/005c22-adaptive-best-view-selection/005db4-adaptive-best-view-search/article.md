# Adaptive Best View Search

Also called: Adaptive Spherical Subdivision, Lipschitz-Bounded Best View Search

## Overview
Adaptive Best View Search is an algorithm to efficiently select the viewpoint with the highest Multi-Scale Entropy value on a bounding sphere, avoiding brute-force search over all possible viewpoints. It differs from exhaustive sampling by adaptively subdividing a spherical mesh based on a Lipschitz-continuous quality estimator, focusing computational effort only on regions where the entropy could potentially exceed the current maximum.

## How it works
**Inputs:**
- A volume model or scene representation.
- A bounding sphere enclosing the model.
- A quality function $H(v)$ that computes the Multi-Scale Entropy for a viewpoint $v$ on the sphere.
- An initial resolution for quality evaluation (e.g., $128 \times 128$ pixels is sufficient for interactive rates [S1]).

**Outputs:**
- The viewpoint $v_{best}$ with the maximum estimated entropy $H_{max}$.

**Data Structures:**
- **Spherical Triangle Mesh:** A set of triangles covering the bounding sphere. Vertices are 3D unit vectors representing viewpoints.
- **Quality Cache:** A map from vertex index to entropy value $H_i$.
- **Global Maximum:** $H_{max}$, initialized to the minimum possible entropy or the first evaluated view.

**Parameters:**
- **Lipschitz Constant ($L$):** An upper bound on the rate of change of the quality function. Recalculated after each new evaluation [S1].
- **Safety Constant ($K_{safe}$):** A conservative margin to prevent missing the global maximum. If $L < 1$, $K_{safe} = 0.02 \times H_{max}$; otherwise, $K_{safe} = 0$ [S1].
- **Angular Threshold:** Views are considered too close if they are within 5 degrees of existing analyzed positions [S1].

**Algorithm Steps:**

1.  **Initialization:**
    - Create a spherical triangle mesh with vertices placed at the vertices of an icosahedron.
    - For each vertex $i$ in the mesh, evaluate the quality $H_i$ by rendering the viewpoint.
    - Set $H_{max} = \max_i(H_i)$.
    - Calculate the initial Lipschitz constant $L$ for the mesh. This is done by finding the maximum ratio of quality difference to angular distance between all pairs of analyzed points in the current mesh:
      $$ L = \max_{i \neq j} \frac{|H_i - H_j|}{\theta_{ij}} $$
      where $\theta_{ij}$ is the angular distance between vertices $i$ and $j$ [S1].

2.  **Estimation Phase:**
    - For each triangle $T$ in the current mesh with vertices $v_1, v_2, v_3$ and known qualities $H_1, H_2, H_3$:
        - Estimate the maximum reachable entropy at each edge. For an edge between vertices $a$ and $b$ with qualities $H_a$ and $H_b$, the maximum possible entropy at any point $p$ on the edge is bounded by:
          $$ H_{edge}(p) \leq \min(H_a, H_b) + L \cdot \max(d(p, v_a), d(p, v_b)) $$
          However, the source describes estimating a specific point $H_{e1}, H_{e2}, H_{e3}$ for each edge. Based on the Lipschitz property, the maximum value on an edge is typically at one of the endpoints or determined by the slope. The source states: "estimate the maximum reachable entropy point at each edge ($H_{e1} \dots H_{e3}$) using $L$". A standard interpretation for a Lipschitz function on a segment is that the maximum is at one of the endpoints unless the function is concave, but the algorithm uses $L$ to bound the variation. The text implies calculating a potential peak. Let $H_{e, max}$ be the maximum possible value on the edge. A safe upper bound for any point on the edge is $\max(H_a, H_b)$. To find if a higher value *could* exist, we look at the interpolation. The source says: "estimate the maximum value inside the triangle $H_{emax}$ by interpolating the computed edge values ($H_{e1}$ to $H_{e3}$)".
          - Let's refine the estimation logic based on the text: "For each triangle... estimate the maximum reachable entropy point at each edge... Then, estimate the maximum value inside the triangle $H_{emax}$ by interpolating the computed values."
          - This implies a hierarchical bound. First, bound the edges. Then bound the triangle.
          - Let $H_{edge\_max}(e)$ be the upper bound of entropy on edge $e$. If the function is Lipschitz, the maximum on the edge is simply $\max(H_u, H_v)$ if we only know endpoints. However, the algorithm likely assumes the quality function might have a peak *within* the edge if the endpoints are not the maxima. But with only Lipschitz continuity, we cannot guarantee a peak inside without more info. The text says "estimate... using L".
          - A common approach in such adaptive sampling (like Gumhold) is to use the Lipschitz constant to define a cone of possible values. For an edge with endpoints $u, v$ and values $H_u, H_v$, the maximum possible value at any point $p$ on the edge is $\min(H_u + L \cdot d(p,u), H_v + L \cdot d(p,v))$. The maximum of this function over the edge occurs at the point where the two lines intersect, or at an endpoint.
          - Let's assume the "estimated edge value" $H_{ei}$ is this theoretical upper bound peak for that edge.
          - Then $H_{emax}$ for the triangle is estimated by interpolating these three edge peaks. This likely means taking the maximum of the three edge peaks, or a weighted average if the peaks are interior. The text says "interpolating", which suggests a smooth estimate. However, for a *bound* to decide subdivision, we need an upper bound. The maximum possible value in the triangle is bounded by the maximum of the upper bounds of its edges. So, $H_{emax} = \max(H_{e1}, H_{e2}, H_{e3})$.
        - If $H_{emax} + K_{safe} > H_{max}$:
            - Identify the specific position on the triangle (or edge) that yields this maximum estimate.
            - Render the viewpoint at this position to get the actual entropy $H_{new}$.
            - Update $H_{max} = \max(H_{max}, H_{new})$.
            - Subdivide the triangle (e.g., mid-point subdivision) and add the new vertices/triangles to the mesh.
            - Recalculate the Lipschitz constant $L$ for the entire mesh using all newly added and existing points [S1].

3.  **Termination:**
    - The algorithm stops when:
        - None of the estimated entropy values ($H_{emax}$) for any triangle is higher than $H_{max} + K_{safe}$.
        - OR, the potential new viewpoints are too close (within 5 degrees) to existing analyzed positions [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_best_view_search(volume_model, bounding_sphere_radius=1.0):
    # 1. Initialize Mesh
    mesh = create_icosahedron_mesh()
    
    # 2. Initial Evaluation
    H_cache = {}
    for vertex in mesh.vertices:
        H_cache[vertex.id] = compute_entropy(volume_model, vertex.position)
    
    H_max = max(H_cache.values())
    K_safe = 0.0
    L = calculate_lipschitz_constant(mesh, H_cache)
    
    # Set safety constant if L < 1
    if L < 1.0:
        K_safe = 0.02 * H_max
    else:
        K_safe = 0.0
        
    # 3. Adaptive Loop
    while True:
        candidates = []
        
        for triangle in mesh.triangles:
            v1, v2, v3 = triangle.vertices
            h1, h2, h3 = H_cache[v1.id], H_cache[v2.id], H_cache[v3.id]
            
            # Estimate max on edges
            # Edge 1-2
            he1 = estimate_edge_max(v1, v2, h1, h2, L)
            # Edge 2-3
            he2 = estimate_edge_max(v2, v3, h2, h3, L)
            # Edge 3-1
            he3 = estimate_edge_max(v3, v1, h3, h1, L)
            
            # Estimate max in triangle
            hemax = max(he1, he2, he3)
            
            if hemax + K_safe > H_max:
                # Determine the position to test
                # This position is derived from the edge/triangle geometry that yielded hemax
                test_pos = get_peak_position(triangle, he1, he2, he3)
                
                # Check angular distance to existing views
                if is_too_close(test_pos, mesh.vertices, threshold_degrees=5.0):
                    continue
                    
                candidates.append((hemax, test_pos, triangle))
        
        if not candidates:
            break
            
        # Process best candidate (or all candidates, source implies iterative refinement)
        # Source says "If Hemax + Ksafe is higher... measure... and add subdivided triangle"
        # It implies processing the triangle that has the highest potential.
        
        # Sort candidates by hemax descending
        candidates.sort(key=lambda x: x[0], reverse=True)
        
        best_hemax, best_pos, best_triangle = candidates[0]
        
        # Render and evaluate
        h_new = compute_entropy(volume_model, best_pos)
        
        # Add new vertex to mesh if not already present (or create new one)
        new_vertex = mesh.add_vertex(best_pos)
        H_cache[new_vertex.id] = h_new
        
        # Update global max
        if h_new > H_max:
            H_max = h_new
            
        # Recalculate Lipschitz constant
        L = calculate_lipschitz_constant(mesh, H_cache)
        
        # Update safety constant
        if L < 1.0:
            K_safe = 0.02 * H_max
        else:
            K_safe = 0.0
            
        # Subdivide the triangle
        mesh.subdivide(best_triangle)
        
        # Re-evaluate H_cache for new vertices if not already done (they are in H_cache)
        # The loop continues to check the new triangles
        
    return mesh.get_vertex_with_max_entropy(H_cache)

def estimate_edge_max(v1, v2, h1, h2, L):
    # Upper bound of a Lipschitz function on a segment
    # The maximum possible value is at one of the endpoints
    # unless we assume a specific shape. 
    # However, the text says "estimate... using L".
    # A tighter bound for the *maximum* on the edge is simply max(h1, h2).
    # But if we want to find a point *inside* that might be higher, 
    # we can't know without more info. 
    # The algorithm likely uses the intersection of the Lipschitz cones.
    # Max possible value at point p on edge: min(h1 + L*d(p,v1), h2 + L*d(p,v2))
    # The peak of this function is at the midpoint if h1=h2, or closer to the higher value.
    # The value at the peak is (h1 + h2 + L * dist(v1,v2)) / 2.
    dist = angular_distance(v1, v2)
    peak_val = (h1 + h2 + L * dist) / 2.0
    return peak_val

def calculate_lipschitz_constant(mesh, H_cache):
    max_ratio = 0.0
    vertices = list(mesh.vertices)
    for i in range(len(vertices)):
        for j in range(i + 1, len(vertices)):
            v1 = vertices[i]
            v2 = vertices[j]
            h1 = H_cache[v1.id]
            h2 = H_cache[v2.id]
            dist = angular_distance(v1, v2)
            if dist > 1e-6:
                ratio = abs(h1 - h2) / dist
                if ratio > max_ratio:
                    max_ratio = ratio
    return max_ratio

def is_too_close(pos, vertices, threshold_degrees=5.0):
    threshold_rad = math.radians(threshold_degrees)
    for v in vertices:
        if angular_distance(pos, v) < threshold_rad:
            return True
    return False
```

## Subtopics

### Importance-Sampled Similarity Computation
A computational optimization for finding the maximum feature similarity by restricting the search space to the 10 closest views and using importance sampling. This approach introduces slight bias because it ignores views further away, but it significantly reduces computation time while still finding the maximum similarity [S2].

## Operating characteristics
- **Time Complexity:** Significantly faster than brute-force search. The source notes that it obtains very similar maximum values to brute-force but at a "fraction of the time" [S1]. The exact complexity depends on the distribution of the entropy function and the Lipschitz constant.
- **Memory Complexity:** Proportional to the number of vertices in the adaptive mesh, which is typically much smaller than the number of views in a dense grid search.
- **Strengths:** Interactive rates are achievable. Small image resolutions (e.g., $128 \times 128$) are sufficient for the quality evaluation, further speeding up the process [S1].
- **Limitations:** The method relies on the Lipschitz assumption. If the quality function has sharp, narrow peaks that violate the Lipschitz bound derived from the current mesh, the algorithm might miss the global maximum (mitigated by $K_{safe}$). It assumes the quality function is relatively smooth.
- **When to use:** When interactive view selection is required and the quality metric (entropy) is computationally expensive to evaluate.
- **When not to use:** When the quality function is highly discontinuous or non-Lipschitz.

## Background
The method was introduced in the paper "Representative Views and Paths for Volume Models" [S1]. It builds on the idea of treating the quality function as Lipschitz-continuous, similar to approaches by Gumhold [S1]. The goal was to achieve interactive rates for selecting representative views of volume models.

## Sources
[S1] Representative Views and Paths for Volume Models, https://doi.org/10.1007/978-3-540-85412-8_10, fetched 2026-10-26
[S2] NeRF Analogies: Example-Based Visual Attribute Transfer for NeRFs, https://doi.org/10.1109/cvpr52733.2024.00444, fetched 2026-10-26
