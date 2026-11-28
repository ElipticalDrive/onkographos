# Foreground Rendering with Point-NeRF

Also called: Per-point Aggregate Module, Point-based Radiance Field

## Overview
This method renders foreground features by sampling points along rays from a dense point cloud and aggregating local geometric structure from neighboring neural points [S1]. It differs from standard Point-NeRF by replacing the original feature encoding with a redesigned per-point aggregate module that perceives local geometry, necessary because the upstream diffusion module generates coordinates without color information [S1].

## How it works
**Inputs:**
- Dense point cloud coordinates $x_0$ [S1].
- Ray direction $d$ [S1].
- Sampling points $p_i$ along the rays [S1].

**Outputs:**
- Volume density $\sigma$ [S1].
- View-dependent radiance $r$ [S1].
- Local geometric features $f_i$ for each sampling point [S1].

**Data Structures:**
- Neural points: Stored as coordinates $c$.
- Neighborhood points $kp_i$: A set of $K$ points queried within radius $R$ [S1].
- Geometric differences $kg_i$: Encoded vector representation of local geometry [S1].

**Parameters:**
- Number of neighbors $K=8$ [S1].
- Neighborhood geometric feature dimension expanded to 8 [S1].
- Euclidean distance radius $R$ [S1].

**Steps:**

1.  **Query Neighbors:** For each sampling point $p_i$, query $K$ neighboring neural points $kp_i = \{kp_{1i}, kp_{2i}, ..., kp_{Ki}\}$ within a Euclidean distance radius $R$ [S1].

2.  **Encode Geometric Differences:** For each neighborhood point $kp_j$ (with coordinate $c_j$) relative to the sampling point $p_i$ (with coordinate $c_i$), encode the geometric differences $kg_i$ using an MLP [S1]:
    $$ kg_i = \text{MLP}(c_i \oplus c_j \oplus (c_i - c_j) \oplus d(c_i, c_j)) $$
    where $\oplus$ is concatenation, $d(\cdot, \cdot)$ is Euclidean distance, and $c_i, c_j$ are coordinates [S1].

3.  **Compute Local Geometric Features:** Obtain the local geometric feature $f_i$ by weighted summation of the neighborhood points' encoded differences [S1]:
    $$ f_i = \text{SUM}(\text{softmax}(\text{MLP}(kg_i)) \odot kg_i) $$
    where $\text{softmax}$ is performed on each dimension, $\odot$ is the Hadamard product, and $\text{SUM}$ aggregates the weighted features [S1].

4.  **Regress Density and Radiance:** Regress volume density $\sigma$ and view-dependent radiance $r$ from the coordinates $c$, local geometric features $f$, and ray direction $d$ using the Point-NeRF module [S1]:
    $$ (\sigma, r) = \text{Point-NeRF}(c, d, f) $$
    [S1].

5.  **Synthesize Feature:** Synthesize the foreground feature by combining the outputs from each neural sampling point along the sampling ray [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def query_neighbors(point_cloud, query_point, radius, k):
    """
    Query K nearest neighbors within radius R.
    point_cloud: Array of shape (N, 3) containing coordinates c.
    query_point: Array of shape (3,) containing c_i.
    radius: Float R.
    k: Integer K.
    Returns: Array of shape (K, 3) containing coordinates of neighbors.
    """
    # Calculate Euclidean distances
    diffs = point_cloud - query_point
    dists = np.linalg.norm(diffs, axis=1)
    
    # Filter by radius
    mask = dists <= radius
    valid_indices = np.where(mask)[0]
    valid_dists = dists[valid_indices]
    
    # Sort by distance and select top K
    sorted_indices = np.argsort(valid_dists)
    selected_indices = valid_indices[sorted_indices[:k]]
    
    return point_cloud[selected_indices]

def encode_geometric_diff(c_i, c_j):
    """
    Encode geometric differences kg_i.
    c_i: Array (3,) coordinate of sampling point.
    c_j: Array (3,) coordinate of neighbor.
    Returns: Array representing kg_i.
    """
    # Concatenation and difference
    # Note: The source defines kg_i per neighbor pair, but the aggregation
    # implies an MLP processes these. We assume an MLP function exists.
    # Input to MLP: [c_i, c_j, c_i - c_j, distance]
    diff = c_i - c_j
    dist = np.linalg.norm(diff)
    input_vec = np.concatenate([c_i, c_j, diff, [dist]])
    
    # Placeholder for MLP implementation
    # In a real implementation, this would be a neural network layer.
    # Here we return a dummy vector to satisfy the structure requirement.
    # [G] Assuming a linear projection for the sake of valid Python syntax.
    return input_vec @ np.random.randn(input_vec.shape[0], 8)

def compute_local_feature(neighbor_coords, c_i):
    """
    Compute local geometric feature f_i.
    neighbor_coords: Array (K, 3) of neighbor coordinates.
    c_i: Array (3,) of sampling point coordinate.
    Returns: Array (D,) of local feature f_i.
    """
    K = neighbor_coords.shape[0]
    kg_vectors = []
    
    for j in range(K):
        c_j = neighbor_coords[j]
        kg = encode_geometric_diff(c_i, c_j)
        kg_vectors.append(kg)
        
    kg_matrix = np.array(kg_vectors) # Shape (K, D_kg)
    
    # Apply MLP to kg_i (element-wise or per-vector)
    # Source says softmax(MLP(kg_i)). Assuming MLP outputs same dim as kg_i.
    # [G] Using a dummy MLP output for valid Python.
    mlp_output = kg_matrix @ np.random.randn(kg_matrix.shape[1], kg_matrix.shape[1])
    
    # Softmax on each dimension (axis=0 for per-dimension softmax across neighbors? 
    # Or axis=1 for per-neighbor softmax? Source says "on each dimension".
    # Usually softmax is over the aggregation dimension (neighbors).
    # Let's assume softmax over K (axis=0) to get weights for summation.
    exp_vals = np.exp(mlp_output - np.max(mlp_output, axis=0, keepdims=True))
    weights = exp_vals / np.sum(exp_vals, axis=0, keepdims=True)
    
    # Hadamard product and Sum
    # weights * kg_matrix
    weighted_kg = weights * kg_matrix
    f_i = np.sum(weighted_kg, axis=0)
    
    return f_i

def point_nerf_render(c, d, f):
    """
    Regress density and radiance.
    c: Coordinate.
    d: Ray direction.
    f: Local feature.
    Returns: (sigma, r)
    """
    # [G] Dummy implementation of Point-NeRF module.
    input_vec = np.concatenate([c, d, f])
    # Dummy MLP
    out = input_vec @ np.random.randn(input_vec.shape[0], 2)
    sigma = np.exp(out[0]) # Ensure positive density
    r = out[1:]
    return sigma, r

def render_foreground(point_cloud, rays, K=8, R=1.0):
    """
    Main rendering function.
    point_cloud: (N, 3)
    rays: List of (origin, direction) tuples.
    """
    rendered_features = []
    
    for origin, direction in rays:
        # Sample points along ray from point cloud prior
        # [G] Assuming sampling generates points p_i. 
        # In practice, this might involve intersecting rays with point cloud bounds.
        # For this method, we sample p_i along the ray.
        # Let's assume we have a set of sampling points p_i for this ray.
        # Since the source doesn't specify sampling strategy, we generate dummy points.
        # [G] Dummy sampling points along the ray.
        t_vals = np.linspace(0, 10, 10)
        p_i_list = origin + t_vals[:, None] * direction
        
        ray_features = []
        for p_i in p_i_list:
            # Step 1: Query neighbors
            neighbors = query_neighbors(point_cloud, p_i, R, K)
            
            # Step 2 & 3: Compute local feature
            f_i = compute_local_feature(neighbors, p_i)
            
            # Step 4: Regress density and radiance
            sigma, r = point_nerf_render(p_i, direction, f_i)
            
            # Step 5: Synthesize feature (accumulation logic omitted as not detailed)
            # [G] Assuming simple accumulation or final point output.
            ray_features.append((sigma, r))
            
        rendered_features.append(ray_features)
        
    return rendered_features
```

## Operating characteristics
- **Time Complexity:** Depends on the number of sampling points, the number of neighbors $K$, and the cost of the MLPs. Querying neighbors requires distance calculations.
- **Memory Complexity:** Stores the point cloud coordinates and the MLP parameters.
- **Performance:** Trained on 4 A100 GPUs for around one day in the first stage, and 20 hours on a single A100 GPU in the second stage [S1].
- **Strengths:** Preserves detail from dense surface points; handles foreground effectively [S1].
- **Limitations:** Point clouds cannot handle unbounded backgrounds, requiring a separate background module (e.g., Mip-NeRF 360) [S1].

## Background
This method is part of a point diffusion implicit function for large-scale scene neural representation [S1]. It builds on Point-NeRF [34] and Mip-NeRF 360 [2] [S1]. The specific adaptation addresses the lack of color information in point clouds generated by a point super-resolution diffusion module [S1].

## Sources
[S1] Point Diffusion Implicit Function for Large-scale Scene Neural Representation, https://doi.org/10.48550/arXiv.2311.01773, fetched 2026-10-29
