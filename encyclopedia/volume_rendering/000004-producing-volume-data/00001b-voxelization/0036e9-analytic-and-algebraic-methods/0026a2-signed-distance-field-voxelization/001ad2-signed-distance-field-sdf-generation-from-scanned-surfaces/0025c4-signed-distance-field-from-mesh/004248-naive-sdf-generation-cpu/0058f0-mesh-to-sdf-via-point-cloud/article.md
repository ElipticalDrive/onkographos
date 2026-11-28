# Mesh to SDF via Point Cloud

Also called: Mesh to Signed Distance Field (SDF) via Point Cloud, SDF Voxelization, SurfacePointCloud SDF computation, Voxelization of SDF

## Overview
This method generates a Signed Distance Field (SDF) for a triangular mesh by first constructing a dense surface point cloud and then evaluating the distance from query points to this cloud. It solves the problem of converting arbitrary manifold geometry into a voxelized or point-based distance representation suitable for volume rendering or neural implicit surface training. It differs from direct mesh rasterization [GPU-based SDF Generation](10690-gpu-based-sdf-generation.md) by decoupling surface sampling from distance evaluation, allowing for flexible sign determination via normals or depth buffers, and handling large query sets via batching.

## How it works
The method operates in two main phases: surface point cloud generation and SDF evaluation. The input mesh is first normalized to a unit cube.

### Inputs and Outputs
- **Input**: A triangular mesh (vertices and faces), query points (optional, for direct evaluation), and configuration parameters.
- **Output**: Either a voxel grid (3D array of SDF values) or an array of SDF values for specific query points.

### Data Structures
- **SurfacePointCloud**:
    - `mesh`: The input trimesh object.
    - `points`: Numpy array of shape $(N, 3)$, containing the sampled surface points.
    - `normals`: Numpy array of shape $(N, 3)$, containing unit normals at each point (optional, required for 'normal' sign method).
    - `scans`: List of `Scan` objects, each containing depth and normal buffers from a specific camera viewpoint.
    - `kd_tree`: A KD-Tree data structure built on `points` for efficient nearest-neighbor queries.

### Steps

1.  **Input Validation**:
    - Ensure `query_points` is a numpy array of shape $(N, 3)$.
    - If `surface_point_method` is `'sample'` and `sign_method` is `'depth'`, override `sign_method` to `'normal'` due to incompatibility [S1, unofficial].

2.  **Mesh Normalization**:
    - Scale the mesh to fit within a unit cube centered at the origin. This is typically done by computing the bounding box and applying a uniform scale and translation [S1, unofficial].

3.  **Surface Point Cloud Generation**:
    - Generate points on the mesh surface using one of two methods:
        - **Scan**: Render the mesh from multiple camera viewpoints to generate depth and normal buffers. Points are extracted from these buffers.
            - Parameters: `scan_count` (default 100), `scan_resolution` (default 400).
            - Camera positions are distributed using rotation matrices around the X and Y axes [S1, unofficial].
        - **Sample**: Uniformly sample points directly from the mesh triangles.
            - Parameter: `sample_point_count` (default 10,000,000).
    - If `sign_method` is `'normal'`, compute vertex/face normals and associate them with the sampled points [S1, unofficial].
    - Build a KD-Tree on the generated points for $O(\log N)$ nearest-neighbor queries.

4.  **SDF Evaluation**:
    - For each query point $q$, find the nearest neighbor(s) in the point cloud.
    - **Sign Determination**:
        - **Normal Method**:
            - Query the KD-Tree for $k$ nearest neighbors (default $k=11$, parameter `normal_sample_count`).
            - Compute direction vectors: $d_i = q - p_i$, where $p_i$ are the nearest points.
            - Determine if $q$ is inside the surface by checking the dot product with normals $n_i$:
              $$ \text{inside} = \sum_{i=1}^{k} \text{sign}(d_i \cdot n_i) > \frac{k}{2} $$
              Specifically, if the sum of dot products is less than $k \times 0.5$ (assuming normalized vectors and specific orientation conventions), the point is considered inside [S1, unofficial].
            - The distance is the distance to the nearest neighbor. If inside, negate the distance.
        - **Depth Method**:
            - Use the depth buffers from the scans.
            - Determine if $q$ is outside by ray casting or checking visibility against the depth buffers.
            - If not outside (i.e., inside), negate the distance [S1, unofficial].
    - **Gradient Calculation (Optional)**:
        - Compute raw gradient as the normalized direction vector to the nearest neighbor.
        - If inside, negate the gradient.
        - **Near-Surface Correction**: For points where $|dist| < \sqrt{0.0025^2 \times 3} \times 3$, replace the gradient with the pre-computed surface normal of the nearest neighbor to reduce noise [S1, unofficial].
        - Normalize the final gradient.

5.  **Voxelization (if generating a grid)**:
    - Generate a grid of query points spanning the unit cube $[-1, 1]^3$ with resolution $V \times V \times V$ (parameter `voxel_resolution`, default 64).
    - Evaluate SDF for all grid points (using batching if $N > 1,000,000$).
    - Reshape results to $(V, V, V)$.
    - **Padding**: If `pad=True`, pad the voxel grid by 1 voxel with a constant value of 1.0 (for SDF) or edge mode (for gradients) [S1, unofficial].
    - **Validation**: If `check_result=True`, verify smoothness. The max difference between adjacent voxels must be $< \frac{2.0}{V} \sqrt{3} \times 1.1$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from sklearn.neighbors import KDTree

def mesh_to_sdf_via_point_cloud(mesh, query_points, surface_point_method='scan', 
                                sign_method='normal', scan_count=100, 
                                scan_resolution=400, sample_point_count=10000000, 
                                normal_sample_count=11, batch_size=1000000):
    """
    Computes SDF for query_points based on a mesh.
    """
    # 1. Input Validation
    if not isinstance(query_points, np.ndarray) or query_points.shape[1] != 3:
        raise ValueError("query_points must be a numpy array of shape (N, 3)")
    
    if surface_point_method == 'sample' and sign_method == 'depth':
        sign_method = 'normal'  # Incompatible combination fallback

    # 2. Normalize Mesh
    mesh = scale_to_unit_cube(mesh)

    # 3. Generate Point Cloud
    points, normals = get_surface_point_cloud(
        mesh, 
        surface_point_method, 
        bounding_radius=np.sqrt(3), 
        scan_count=scan_count, 
        scan_resolution=scan_resolution, 
        sample_point_count=sample_point_count,
        calculate_normals=(sign_method == 'normal')
    )
    
    kd_tree = KDTree(points)
    n_points = points.shape[0]

    # 4. Evaluate SDF in Batches
    all_sdfs = []
    all_gradients = []
    
    num_batches = (query_points.shape[0] + batch_size - 1) // batch_size
    
    for i in range(num_batches):
        start_idx = i * batch_size
        end_idx = min((i + 1) * batch_size, query_points.shape[0])
        batch_queries = query_points[start_idx:end_idx]
        
        sdf_batch, grad_batch = compute_sdf_batch(
            batch_queries, points, normals, kd_tree, 
            sign_method, normal_sample_count
        )
        all_sdfs.append(sdf_batch)
        all_gradients.append(grad_batch)

    return np.concatenate(all_sdfs), np.concatenate(all_gradients)

def compute_sdf_batch(queries, points, normals, kd_tree, sign_method, k):
    """
    Computes SDF for a batch of query points.
    """
    n_queries = queries.shape[0]
    sdfs = np.zeros(n_queries)
    gradients = np.zeros((n_queries, 3))
    
    # Query KD-Tree
    # distances: (n_queries, k), indices: (n_queries, k)
    distances, indices = kd_tree.query(queries, k=k)
    
    if sign_method == 'normal':
        # Get normals for nearest neighbors
        neighbor_normals = normals[indices] # (n_queries, k, 3)
        neighbor_points = points[indices]   # (n_queries, k, 3)
        
        # Direction vectors
        directions = queries[:, np.newaxis, :] - neighbor_points # (n_queries, k, 3)
        
        # Dot products
        dots = np.sum(directions * neighbor_normals, axis=2) # (n_queries, k)
        
        # Determine inside status
        # If sum of dots < k * 0.5, it is inside (based on source logic)
        inside = np.sum(dots, axis=1) < (k * 0.5)
        
        # Distance is to the nearest neighbor (index 0)
        dists = distances[:, 0]
        sdfs = np.where(inside, -dists, dists)
        
        # Gradient calculation
        raw_grads = directions[:, 0, :] # Direction to nearest neighbor
        raw_grads = np.where(inside[:, np.newaxis], -raw_grads, raw_grads)
        
        # Normalize
        norms = np.linalg.norm(raw_grads, axis=1, keepdims=True)
        norms = np.maximum(norms, 1e-8)
        gradients = raw_grads / norms
        
        # Near-surface gradient correction
        near_surface = np.abs(sdfs) < (np.sqrt(0.0025**2 * 3) * 3)
        if np.any(near_surface):
            # Replace gradient with normal of nearest neighbor
            gradients[near_surface] = neighbor_normals[near_surface, 0, :]
            
    elif sign_method == 'depth':
        # Depth method requires scan data and ray casting logic
        # Simplified placeholder for depth-based sign
        # In practice, this involves checking visibility against stored depth buffers
        raise NotImplementedError("Depth sign method requires Scan objects and visibility checks")
        
    return sdfs, gradients

def get_surface_point_cloud(mesh, method, bounding_radius, scan_count, scan_resolution, sample_point_count, calculate_normals):
    """
    Generates surface points from mesh.
    """
    if method == 'scan':
        # Render mesh from multiple viewpoints
        # Returns points and normals from depth/normal buffers
        points, normals = render_and_extract_points(mesh, scan_count, scan_resolution)
    elif method == 'sample':
        # Uniformly sample triangles
        points = sample_mesh_triangles(mesh, sample_point_count)
        if calculate_normals:
            normals = compute_vertex_normals(mesh, points)
        else:
            normals = None
    else:
        raise ValueError("Unknown method")
        
    return points, normals
```

### Parameters
- `scan_count`: Number of camera scans for surface point generation. Default: 100 [S1, unofficial].
- `scan_resolution`: Resolution of each scan buffer. Default: 400 [S1, unofficial].
- `sample_point_count`: Number of points to sample if using 'sample' method. Default: 10,000,000 [S1, unofficial].
- `normal_sample_count`: Number of nearest neighbors ($k$) for KD-tree query in normal sign method. Default: 11 [S1, unofficial].
- `batch_size`: Max query points per batch. Default: 1,000,000 [S1, unofficial].
- `voxel_resolution`: Output voxel grid size $N \times N \times N$. Default: 64 [S1, unofficial].

### Pitfalls
- **Incompatible Methods**: Using `'sample'` for surface generation with `'depth'` for sign determination is invalid. The code automatically falls back to `'normal'` [S1, unofficial].
- **Near-Surface Gradient Noise**: Distances very close to zero may have noisy gradients from the KD-tree direction. These are replaced by pre-computed surface normals [S1, unofficial].
- **Precision**: The validation check for voxel smoothness uses a threshold of $\frac{2.0}{V} \sqrt{3} \times 1.1$. Deviations may indicate sampling artifacts or insufficient point density [S1, unofficial].

## Subtopics

### [Sampling SDF Near Surface](16321-sampling-sdf-near-surface.md)
Generates query points concentrated near the mesh surface for training SDFs, addressing insufficient surface detail in uniform sampling.

## Operating characteristics
- **Time Complexity**: $O(N \log M)$ for $N$ query points and $M$ surface points, due to KD-tree queries. Batching allows processing large $N$ within memory limits.
- **Memory Complexity**: $O(M + N)$ for storing the point cloud and query results. The KD-tree adds $O(M)$ overhead.
- **Performance**: Depends heavily on `scan_count` and `sample_point_count`. High resolution scans (400x400) and millions of sample points ensure accuracy but increase memory and computation time.
- **Strengths**: Flexible sign determination; handles complex geometry via point cloud abstraction; supports both voxel and point-based outputs.
- **Limitations**: Requires significant memory for large point clouds; depth-based sign determination is more complex to implement than normal-based; near-surface noise requires correction.

## Background
The method is derived from the `mesh_to_sdf` library, which implements strategies for generating Signed Distance Fields from triangular meshes. It builds on techniques from neural implicit surface representations, such as those proposed in the DeepSDF paper, which emphasize sampling near the surface for effective training [S1, unofficial].

## Sources
[S1] mesh_to_sdf, https://github.com/marian42/mesh_to_sdf, fetched 2026-10-28 (unofficial)
