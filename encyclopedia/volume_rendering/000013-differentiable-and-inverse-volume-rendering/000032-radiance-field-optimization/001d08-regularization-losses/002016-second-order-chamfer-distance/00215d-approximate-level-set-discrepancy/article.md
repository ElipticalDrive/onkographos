# Approximate Level Set Discrepancy

Also called: LSD-A

## Overview
Approximate Level Set Discrepancy (LSD-A) is a metric used to quantify the geometric discrepancy between a generated mesh and a ground truth mesh, specifically designed for scenarios where the ground truth is non-watertight or an analytic Signed Distance Field (SDF) is ill-defined [S1]. Unlike the Precise Level Set Discrepancy (LSD-P), which evaluates distance errors on a fixed grid near the surface, LSD-A measures the point-to-face distance from points sampled directly on the ground truth mesh to the generated mesh [S1]. It serves as a robust alternative to standard Chamfer Distance when strict watertightness cannot be guaranteed [S1].

## How it works
**Inputs:**
- $\mathcal{M}^*$: The ground truth mesh.
- $\mathcal{M}$: The generated (reconstructed) mesh.
- $N$: The number of points sampled on $\mathcal{M}^*$.

**Outputs:**
- A scalar value representing the average point-to-face distance.

**Data Structures:**
- Points $q$: A set of $N$ points sampled uniformly or via stratified sampling on the surface of $\mathcal{M}^*$.
- Mesh $\mathcal{M}$: Represented as a collection of vertices and faces, supporting point-to-face distance queries.

**Steps:**
1. Sample $N$ points $q$ distributed over the surface of the ground truth mesh $\mathcal{M}^*$ [S1].
2. For each point $q \in \mathcal{M}^*$, compute the Euclidean distance to the closest point on the generated mesh $\mathcal{M}$ [S1]. This distance is denoted as $d^p(q, \mathcal{M})$ [S1].
3. Sum these distances and divide by $N$ to obtain the mean [S1].

**Formula:**
$$ \text{LSD-A} = \frac{1}{N} \sum_{q \in \mathcal{M}^*} d^p(q, \mathcal{M}) $$

Where:
- $\mathcal{M}^*$ is the ground truth mesh [S1].
- $\mathcal{M}$ is the generated mesh [S1].
- $q$ are points sampled on $\mathcal{M}^*$ [S1].
- $d^p(q, \mathcal{M})$ is the point-to-face distance from point $q$ to the generated mesh $\mathcal{M}$ [S1].
- $N$ is the number of sampled points [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def approximate_level_set_discrepancy(ground_truth_mesh, generated_mesh, num_samples):
    """
    Computes the Approximate Level Set Discrepancy (LSD-A).
    
    Args:
        ground_truth_mesh: Mesh object representing M*.
        generated_mesh: Mesh object representing M.
        num_samples: Integer N, number of points to sample on M*.
        
    Returns:
        float: The LSD-A score.
    """
    # Step 1: Sample points on the ground truth mesh
    # Implementation note: Use a standard mesh sampling method (e.g., uniform area-weighted)
    # to generate 'num_samples' points on ground_truth_mesh.
    sampled_points = sample_points_on_mesh(ground_truth_mesh, num_samples)
    
    total_distance = 0.0
    
    # Step 2 & 3: Compute point-to-face distance and average
    for q in sampled_points:
        # Compute distance from point q to the closest point on generated_mesh
        # This involves finding the closest face and projecting q onto it
        dist = point_to_face_distance(q, generated_mesh)
        total_distance += dist
        
    return total_distance / num_samples

def point_to_face_distance(point, mesh):
    """
    Computes the Euclidean distance from a point to the closest point on a mesh.
    For a triangle mesh, this is the minimum distance to any triangle face.
    
    Args:
        point: 3D vector (x, y, z).
        mesh: Mesh object with vertices and faces.
        
    Returns:
        float: Distance to the closest point on the mesh surface.
    """
    min_dist = infinity
    
    for face in mesh.faces:
        # Calculate distance from point to triangle defined by face vertices
        dist = distance_point_to_triangle(point, face.vertices)
        if dist < min_dist:
            min_dist = dist
            
    return min_dist

def distance_point_to_triangle(point, triangle_vertices):
    """
    Computes the distance from a point to a triangle.
    
    Args:
        point: 3D vector.
        triangle_vertices: List of 3 3D vectors.
        
    Returns:
        float: Distance.
    """
    # Standard geometric algorithm for point-to-triangle distance
    # Returns the Euclidean distance to the closest point on the triangle surface
    # (including interior, edges, and vertices)
    return compute_closest_point_on_triangle(point, triangle_vertices).distance_to(point)
```

**Parameters:**
- $N$: The number of points sampled on the ground truth mesh $\mathcal{M}^*$ [S1]. Typical values are not specified in the sources, but sufficient sampling density is required to represent the surface geometry accurately [S1].

**Edge Cases and Pitfalls:**
- **Non-watertight Ground Truth:** LSD-A is specifically intended for cases where the ground truth mesh is non-watertight, making analytic SDF computation ill-defined [S1].
- **Sampling Bias:** If $N$ is too small, the metric may fail to capture fine geometric details or holes in the ground truth mesh [S1].
- **Distance Definition:** The metric uses point-to-face distance, not point-to-vertex distance, ensuring that the entire surface area contributes to the error measurement [S1].

## Subtopics

### Precise Level Set Discrepancy
Precise Level Set Discrepancy (LSD-P) gauges the discrepancy in signed distance values between a reconstructed mesh and the ground truth mesh at fixed points sampled on a regular grid close to the ground truth surface [S1]. It is defined as $\text{LSD-P} = \sum_{g \in \tilde{\mathcal{G}}} \|\text{abs}(\mathbf{d}^p(\mathcal{M}^*, g)) - \text{abs}(\mathbf{d}^p(\mathcal{M}, g))\|_2$, where $\tilde{\mathcal{G}}$ is the grid of points and $\mathbf{d}^p$ is the distance from a query point to its closest point on the surface [S1]. LSD-P is chosen when a fixed spatial grid is preferred over surface sampling, potentially offering more consistent evaluation across different mesh resolutions, whereas LSD-A is preferred for non-watertight meshes where surface sampling is more robust [S1].

## Operating characteristics
**Time Complexity:**
- Sampling points on $\mathcal{M}^*$: Depends on the sampling algorithm, typically $O(N)$ or $O(N \log N)$ depending on spatial indexing [S1].
- Point-to-face distance: For each of the $N$ points, finding the closest point on $\mathcal{M}$ requires searching the mesh. With a naive search, this is $O(N \cdot F)$ where $F$ is the number of faces in $\mathcal{M}$. With spatial acceleration structures (e.g., BVH, grid), this can be reduced to $O(N \log F)$ or better [S1].
- Total: $O(N \cdot \text{search\_cost})$ [S1].

**Memory Complexity:**
- $O(N)$ to store the sampled points [S1].
- $O(V + F)$ to store the generated mesh $\mathcal{M}$, where $V$ is vertices and $F$ is faces [S1].

**Strengths and Limitations:**
- **Strengths:** Robust to non-watertight ground truth meshes; does not require an analytic SDF; directly measures surface proximity [S1].
- **Limitations:** Requires sampling on the ground truth mesh; sensitive to the number of samples $N$; does not account for signed distance errors in the same way LSD-P does [S1].

**When to Use:**
- Use LSD-A when evaluating generated meshes against non-watertight ground truth meshes (e.g., ShapeNet dataset) where analytic SDF is ill-defined [S1].

**When Not to Use:**
- Avoid when the ground truth is watertight and an analytic SDF is available, as LSD-P might provide more consistent grid-based evaluation [S1].

## Background
Approximate Level Set Discrepancy (LSD-A) was introduced in the context of self-supervised dual contouring for 3D mesh reconstruction [S1]. It was specifically designed to handle the challenges posed by datasets like ShapeNet, which contain non-watertight meshes, making traditional SDF-based metrics unreliable [S1]. The metric was used in experiments to evaluate the quality of reconstructed meshes against ground truth in these challenging scenarios [S1].

## Sources
[S1] Self-Supervised Dual Contouring, https://doi.org/10.1109/cvpr52733.2024.00448, fetched 2026-10-26
