# Sampling SDF Near Surface

Also called: DeepSDF Sampling

## Overview
This method generates training data for Signed Distance Fields (SDFs) by sampling query points densely near a mesh surface and uniformly in the surrounding volume, following the strategy proposed in the DeepSDF paper [S1, unofficial]. It solves the problem of creating balanced datasets where SDF values change rapidly near the surface while maintaining interior and exterior coverage [S1, unofficial]. It is distinguished from purely uniform sampling by its focus on surface proximity, which improves reconstruction accuracy for complex geometries [S1, unofficial].

## How it works
**Inputs:**
- `mesh`: The input triangle mesh [S1, unofficial].
- `number_of_points`: Total number of query points to generate, default 500,000 [S1, unofficial].
- `surface_point_method`: Method to generate initial surface points, either `'scan'` or `'sample'` [S1, unofficial].
- `sign_method`: Method to determine the sign of the SDF, either `'normal'` or `'depth'` [S1, unofficial].
- `scan_count`: Number of virtual camera scans if `surface_point_method` is `'scan'`, default 100 [S1, unofficial].
- `scan_resolution`: Resolution of each scan buffer, default 400 [S1, unofficial].
- `sample_point_count`: Number of points to sample per scan or uniformly, default 10,000,000 [S1, unofficial].
- `normal_sample_count`: Number of samples to average for normal estimation if `sign_method` is `'normal'`, default 11 [S1, unofficial].
- `min_size`: Minimum volume ratio threshold, default 0 [S1, unofficial].
- `return_gradients`: Boolean flag to return gradients, default False [S1, unofficial].

**Outputs:**
- Arrays of query point positions and their corresponding SDF values [S1, unofficial].

**Data Structures:**
- `surface_point_cloud`: A collection of 3D points representing the mesh surface, optionally with normals, generated via scanning or uniform sampling [S1, unofficial].
- `query_points`: An array of shape `(number_of_points, 3)` containing the final query locations [S1, unofficial].
- `sdf_values`: An array of shape `(number_of_points,)` containing the signed distance values [S1, unofficial].

**Steps:**
1. **Scale Mesh:** Transform the input mesh to fit within a unit sphere centered at the origin using `scale_to_unit_sphere` [S1, unofficial].
2. **Generate Surface Point Cloud:** Call `get_surface_point_cloud` with the mesh, method, bounding radius 1, scan parameters, sample count, and a flag to calculate normals if `sign_method` is `'normal'` or `return_gradients` is True [S1, unofficial].
3. **Generate Query Points:** Sample surface points for 47/50 of `number_of_points`, split into two batches of half that count each; add Gaussian noise with scale 0.0025 to the first batch and 0.00025 to the second batch; sample the remaining 3/50 of points uniformly within the unit sphere [S1, unofficial].
4. **Compute SDF Values:** Call `get_sdf_in_batches` with the query points and the chosen `sign_method` to compute distances and signs [S1, unofficial].
5. **Size Check (Optional):** If `min_size` > 0, calculate the ratio of negative SDF values among the uniform sphere samples; raise `BadMeshException` if the ratio is less than `min_size` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_sdf_near_surface(mesh, number_of_points=500000, surface_point_method='scan', sign_method='normal', scan_count=100, scan_resolution=400, sample_point_count=10000000, normal_sample_count=11, min_size=0, return_gradients=False):
    # Step 1: Scale mesh to unit sphere
    mesh = scale_to_unit_sphere(mesh)
    
    # Check for incompatible methods
    if surface_point_method == 'sample' and sign_method == 'depth':
        sign_method = 'normal'
        
    # Step 2: Generate surface point cloud
    surface_point_cloud = get_surface_point_cloud(
        mesh, surface_point_method, 1, scan_count, scan_resolution,
        sample_point_count, (sign_method == 'normal' or return_gradients)
    )
    
    # Step 3: Generate query points
    query_points = []
    surface_proximate_count = int(number_of_points * 47 / 50)
    half_count = surface_proximate_count // 2
    
    # Batch 1: coarse noise
    coarse_pts = random_sample(surface_point_cloud.points, half_count)
    query_points.append(coarse_pts + np.random.normal(0, 0.0025, coarse_pts.shape))
    
    # Batch 2: fine noise
    fine_pts = random_sample(surface_point_cloud.points, surface_proximate_count - len(coarse_pts))
    query_points.append(fine_pts + np.random.normal(0, 0.00025, fine_pts.shape))
    
    # Batch 3: uniform sphere
    uniform_count = number_of_points - len(query_points)
    query_points.append(sample_uniform_in_sphere(uniform_count))
    
    query_points = np.vstack(query_points)
    
    # Step 4: Compute SDF values
    sdf_values = compute_sdf_batch(query_points, surface_point_cloud, sign_method, normal_sample_count)
    
    # Step 5: Size check
    if min_size > 0:
        uniform_sdf = sdf_values[-uniform_count:]
        ratio = np.sum(uniform_sdf < 0) / uniform_count
        if ratio < min_size:
            raise BadMeshException("Mesh volume ratio below threshold")
            
    return query_points, sdf_values

# Helper definitions
def scale_to_unit_sphere(mesh):
    # Computes bounding sphere of mesh and scales vertices to radius 1.0
    return mesh * (1.0 / np.max(np.linalg.norm(mesh.vertices, axis=1)))

def get_surface_point_cloud(mesh, method, radius, scan_count, resolution, sample_count, calc_normals):
    # Generates points via scan or uniform sampling
    if method == 'scan':
        return render_scan_cloud(mesh, scan_count, resolution, calc_normals)
    else:
        return uniform_mesh_sample(mesh, sample_count, calc_normals)

def random_sample(points, count):
    # Returns `count` randomly selected points from `points`
    indices = np.random.choice(len(points), count, replace=False)
    return points[indices]

def sample_uniform_in_sphere(count):
    # Rejection sampling for uniform points in unit sphere
    pts = np.random.uniform(-1, 1, (count, 3))
    while np.any(np.sum(pts**2, axis=1) > 1):
        pts = np.random.uniform(-1, 1, (count, 3))
    return pts

def compute_sdf_batch(points, cloud, sign_method, normal_count):
    # Computes distances and applies sign logic
    distances = np.linalg.norm(points[:, np.newaxis] - cloud.points, axis=2).min(axis=1)
    if sign_method == 'normal':
        normals = get_normals(cloud, points, normal_count)
        vectors = points - cloud.points[np.argmin(np.linalg.norm(points[:, np.newaxis] - cloud.points, axis=2), axis=1)]
        signs = np.sign(np.sum(vectors * normals, axis=1))
    else:
        signs = depth_sign_logic(points, cloud)
    return distances * signs

def get_normals(cloud, points, count):
    # Approximates normals via k-nearest neighbors
    return cloud.normals[np.argmin(np.linalg.norm(points[:, np.newaxis] - cloud.points, axis=2), axis=1)]

def depth_sign_logic(points, cloud):
    # Placeholder for depth-buffer based sign determination
    return np.ones(len(points)) # [G] Standard fallback when depth logic is not detailed in sources
```

**Parameters with Typical Values:**
- `number_of_points`: 500,000 [S1, unofficial].
- `scan_count`: 100 [S1, unofficial].
- `scan_resolution`: 400 [S1, unofficial].
- `sample_point_count`: 10,000,000 [S1, unofficial].
- `normal_sample_count`: 11 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Incompatible Methods:** If `surface_point_method` is `'sample'` and `sign_method` is `'depth'`, the method automatically switches `sign_method` to `'normal'` [S1, unofficial].
- **Precision:** The noise scales (0.0025 and 0.00025) are small relative to the unit sphere, requiring floating-point precision to avoid quantization artifacts near the surface [S1, unofficial].
- **Mesh Quality:** The size check helps detect degenerate meshes with negligible volume, which can lead to incorrect SDF signs or training instability [S1, unofficial].

## Subtopics

### [Normal-Based Sign Determination](16359-normal-based-sign-determination.md)
Normal-Based Sign Determination assigns the sign of a Signed Distance Field (SDF) value at a query point by evaluating the dot product between the vector from the closest surface point to the query point and the surface normal at that point. It is distinguished from Depth Buffer Sign Determination by its ability to handle meshes with holes, though it is prone to cone-shaped artifacts on meshes with inconsistent triangle orientations.

### Voxel Initialization with Negative Distances
This strategy initializes Signed Distance Field (SDF) values for voxel corners by assuming all are inside the object (negative SDF), providing a stable starting point for optimization compared to unsigned distances [S2, unofficial]. It defines a uniform dense grid at $L = 6$ ($G = 2^6$) within a predefined cube region, builds a KD-tree over subsampled aligned points $P_{world}^{sub}$ for efficient nearest-neighbor queries, and assigns the negative Euclidean distance to the nearest point for each corner node $p_{geo}$ as $\tilde{f}(p_{geo}) = -\sqrt{\sum_{p_{sub} \in P_{world}^{sub}} |p_{geo} - p_{sub}|^2}$ [S2, unofficial]. Visible corners are later flipped to positive (outside) in a subsequent sign-flipping step [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Dominated by the surface point cloud generation (scanning or sampling) and the nearest neighbor search for each query point [S1, unofficial]. Scanning is $O(scan\_count \times scan\_resolution^2)$, while sampling is $O(sample\_point\_count)$ [S1, unofficial]. SDF computation is $O(number\_of\_points \times log(surface\_point\_cloud\_size))$ with efficient data structures [S1, unofficial].
- **Memory Complexity:** Proportional to the size of the surface point cloud and the number of query points [S1, unofficial].
- **Strengths:** Provides high-density samples near the surface, which is crucial for learning fine geometric details [S1, unofficial]. The uniform samples ensure global coverage [S1, unofficial].
- **Limitations:** Computationally expensive due to the large number of surface points and nearest neighbor searches [S1, unofficial]. The sign determination can be ambiguous for non-manifold or self-intersecting meshes [S1, unofficial].
- **When to Use:** When training deep learning models for shape reconstruction (e.g., DeepSDF) where surface accuracy is paramount [S1, unofficial].
- **When Not to Use:** For real-time applications or when memory is constrained, due to the large number of query points [S1, unofficial].

## Background
The method is based on the sampling strategy proposed in the DeepSDF paper, which emphasizes sampling points near the surface to improve the learning of signed distance fields [S1, unofficial]. The implementation is part of the `mesh_to_sdf` library, which provides tools for converting meshes to SDFs for various graphics and machine learning applications [S1, unofficial].

## Sources
[S1] mesh_to_sdf, https://github.com/marian42/mesh_to_sdf, fetched 2026-10-28 (unofficial)
[S2] SVRecon: Sparse Voxel Rasterization for Surface Reconstruction, https://doi.org/10.48550/arxiv.2511.17364, fetched 2026-10-26 (unofficial)
