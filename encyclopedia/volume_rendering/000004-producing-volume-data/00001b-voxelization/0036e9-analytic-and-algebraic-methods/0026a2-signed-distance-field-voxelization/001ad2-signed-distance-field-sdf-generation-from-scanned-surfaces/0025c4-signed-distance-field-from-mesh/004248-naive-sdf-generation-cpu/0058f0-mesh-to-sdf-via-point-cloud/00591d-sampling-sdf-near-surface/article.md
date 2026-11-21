# Sampling SDF Near Surface

Also called: DeepSDF Sampling

## Overview
This method generates a set of query points for Signed Distance Field (SDF) evaluation by sampling points near the mesh surface and in the surrounding volume, following the strategy proposed in the DeepSDF paper [S1, unofficial]. It solves the problem of creating training data that is dense near the surface (where SDF values change rapidly) while maintaining coverage of the interior and exterior [S1, unofficial]. It is distinguished from uniform sampling by its focus on surface proximity, which improves reconstruction accuracy for thin structures and complex geometries.

## How it works
**Inputs:**
- `mesh`: The input triangle mesh.
- `number_of_points`: Total number of query points to generate. Default: 500,000 [S1, unofficial].
- `surface_point_method`: Method to generate initial surface points. Options: `'scan'` or `'sample'` [S1, unofficial].
- `sign_method`: Method to determine the sign of the SDF. Options: `'normal'` or `'depth'` [S1, unofficial].
- `scan_count`: Number of virtual camera scans if `surface_point_method` is `'scan'`. Default: 100 [S1, unofficial].
- `scan_resolution`: Resolution of each scan buffer. Default: 400 [S1, unofficial].
- `sample_point_count`: Number of points to sample per scan or uniformly. Default: 10,000,000 [S1, unofficial].
- `normal_sample_count`: Number of samples to average for normal estimation if `sign_method` is `'normal'`. Default: 11 [S1, unofficial].
- `min_size`: Minimum volume ratio threshold. If > 0, raises an exception if the mesh volume is too small [S1, unofficial].
- `return_gradients`: Boolean flag to return gradients. Default: False [S1, unofficial].

**Outputs:**
- A list of query points (positions) and their corresponding SDF values (and optionally gradients).

**Data Structures:**
- `surface_point_cloud`: A collection of 3D points representing the mesh surface, generated via scanning or uniform sampling.
- `query_points`: An array of shape `(number_of_points, 3)` containing the final query locations.
- `sdf_values`: An array of shape `(number_of_points,)` containing the signed distance values.

**Steps:**
1. **Scale Mesh:** Transform the input mesh to fit within a unit sphere centered at the origin. This is done by scaling the mesh vertices such that the maximum distance from the origin is 0.5 [S1, unofficial].
2. **Generate Surface Point Cloud:**
   - If `surface_point_method` is `'scan'`:
     - Generate `scan_count` virtual camera positions distributed around the mesh.
     - For each position, render a depth buffer and optionally a normal buffer.
     - Reproject the depth buffer pixels into 3D space to form a point cloud.
     - If `sign_method` is `'normal'` or `return_gradients` is True, calculate normals for these points.
   - If `surface_point_method` is `'sample'`:
     - Sample `sample_point_count` points uniformly on the mesh surface.
   - Store these points in `surface_point_cloud`.
3. **Generate Query Points:**
   - Initialize an empty list for query points.
   - **Surface Proximity Sampling (47/50 of total points):**
     - Select 47/50 of `number_of_points` from the `surface_point_cloud`.
     - For each selected point, add Gaussian noise with standard deviation 0.0025 to its position.
     - Add these noisy points to the query list.
     - Select another 47/50 of `number_of_points` from the `surface_point_cloud` (or reuse the same set, depending on implementation details not fully specified, but typically distinct or repeated sampling) [S1, unofficial].
     - For each selected point, add Gaussian noise with standard deviation 0.00025 to its position.
     - Add these finely noisy points to the query list.
     - *Note:* The source describes "Sample surface points (47/50 of total / 2)" and "Sample surface points again" with different noise scales. This implies two batches of surface-proximate points, totaling 47/50 of the total count, split between coarse and fine noise [S1, unofficial].
   - **Uniform Sampling (3/50 of total points):**
     - Calculate the remaining count: 3/50 of `number_of_points`.
     - Sample these points uniformly within the unit sphere.
     - Add these points to the query list.
4. **Compute SDF Values:**
   - For each query point, compute the distance to the closest point on the `surface_point_cloud`.
   - Determine the sign of the distance:
     - If `sign_method` is `'normal'`: Calculate the dot product between the vector from the closest surface point to the query point and the surface normal at the closest point. If the dot product is positive, the point is outside (positive SDF); if negative, inside (negative SDF) [S1, unofficial].
     - If `sign_method` is `'depth'`: Use the depth buffer information from the scans to determine if the point is inside or outside [S1, unofficial].
   - Store the signed distance.
5. **Size Check (Optional):**
   - If `min_size` > 0, calculate the volume ratio by counting the number of query points with negative SDF values (inside the mesh) relative to the total number of uniform sphere samples.
   - If this ratio is less than `min_size`, raise a `BadMeshException` [S1, unofficial].

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
        mesh, 
        surface_point_method, 
        bounding_radius=1, 
        scan_count=scan_count, 
        scan_resolution=scan_resolution, 
        sample_point_count=sample_point_count, 
        calculate_normals=(sign_method == 'normal' or return_gradients)
    )
    
    # Step 3: Generate query points
    query_points = []
    
    # 47/50 of points are surface-proximate
    surface_proximate_count = int(number_of_points * 47 / 50)
    half_surface_count = surface_proximate_count // 2
    
    # Coarse noise batch
    coarse_points = sample_points_from_cloud(surface_point_cloud, half_surface_count)
    coarse_noisy = add_gaussian_noise(coarse_points, scale=0.0025)
    query_points.extend(coarse_noisy)
    
    # Fine noise batch
    fine_points = sample_points_from_cloud(surface_point_cloud, surface_proximate_count - len(coarse_noisy))
    fine_noisy = add_gaussian_noise(fine_points, scale=0.00025)
    query_points.extend(fine_noisy)
    
    # 3/50 of points are uniform in sphere
    uniform_count = number_of_points - len(query_points)
    uniform_points = sample_uniform_in_sphere(uniform_count)
    query_points.extend(uniform_points)
    
    # Step 4: Compute SDF values
    sdf_values = []
    for point in query_points:
        closest_point, distance = find_closest_point(surface_point_cloud, point)
        
        if sign_method == 'normal':
            normal = get_normal(surface_point_cloud, closest_point, normal_sample_count)
            vector = point - closest_point
            sign = 1 if dot(vector, normal) > 0 else -1
        else:
            # Depth-based sign determination
            sign = determine_sign_depth(point, surface_point_cloud)
            
        sdf_values.append(sign * distance)
        
    # Step 5: Size check
    if min_size > 0:
        inside_count = sum(1 for v in sdf_values if v < 0)
        # Note: The source mentions calculating ratio based on uniform sphere samples, 
        # but the implementation snippet checks against all points or requires specific tracking.
        # Assuming the ratio is calculated over the uniform subset for accuracy.
        uniform_sdf_values = sdf_values[-uniform_count:]
        inside_uniform = sum(1 for v in uniform_sdf_values if v < 0)
        ratio = inside_uniform / uniform_count
        if ratio < min_size:
            raise BadMeshException("Mesh volume is too small")
            
    return query_points, sdf_values
```

**Parameters with Typical Values:**
- `number_of_points`: 500,000 [S1, unofficial].
- `scan_count`: 100 [S1, unofficial].
- `scan_resolution`: 400 [S1, unofficial].
- `sample_point_count`: 10,000,000 [S1, unofficial].
- `normal_sample_count`: 11 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Incompatible Methods:** If `surface_point_method` is `'sample'` and `sign_method` is `'depth'`, the method automatically switches `sign_method` to `'normal'` [S1, unofficial].
- **Precision:** The noise scales (0.0025 and 0.00025) are small relative to the unit sphere, requiring floating-point precision to avoid quantization artifacts near the surface.
- **Mesh Quality:** The size check helps detect degenerate meshes with negligible volume, which can lead to incorrect SDF signs or training instability.

## Subtopics

### [Normal-Based Sign Determination](16359-normal-based-sign-determination.md)
Normal-Based Sign Determination assigns the sign of a Signed Distance Field (SDF) value at a query point by evaluating the dot product between the point's position relative to the closest surface point and the surface normal at that point. It is distinguished from Depth Buffer Sign Determination by its ability to handle meshes with holes, though it is prone to cone-shaped artifacts on meshes with inconsistent triangle orientations.

## Operating characteristics
- **Time Complexity:** Dominated by the surface point cloud generation (scanning or sampling) and the nearest neighbor search for each query point. Scanning is $O(scan\_count \times scan\_resolution^2)$, while sampling is $O(sample\_point\_count)$. SDF computation is $O(number\_of\_points \times log(surface\_point\_cloud\_size))$ with efficient data structures.
- **Memory Complexity:** Proportional to the size of the surface point cloud and the number of query points.
- **Strengths:** Provides high-density samples near the surface, which is crucial for learning fine geometric details. The uniform samples ensure global coverage.
- **Limitations:** Computationally expensive due to the large number of surface points and nearest neighbor searches. The sign determination can be ambiguous for non-manifold or self-intersecting meshes.
- **When to Use:** When training deep learning models for shape reconstruction (e.g., DeepSDF) where surface accuracy is paramount.
- **When Not to Use:** For real-time applications or when memory is constrained, due to the large number of query points.

## Background
The method is based on the sampling strategy proposed in the DeepSDF paper, which emphasizes sampling points near the surface to improve the learning of signed distance fields [S1, unofficial]. The implementation is part of the `mesh_to_sdf` library, which provides tools for converting meshes to SDFs for various graphics and machine learning applications [S1, unofficial].

## Sources
[S1] mesh_to_sdf, https://github.com/marian42/mesh_to_sdf, fetched 2026-10-28 (unofficial)
