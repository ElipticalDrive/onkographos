# L1-absolute Error (Accuracy)

Also called: Average of absolute differences between SDF values, Experimental Evaluation Metrics, L1-abs Error, Mean Absolute Error (Point Cloud), Mesh Sampling Error, TSDF Accuracy Measurement

## Overview
L1-absolute Error quantifies the geometric accuracy of a reconstructed 3D model by measuring the average signed distance field (SDF) deviation of points sampled from a high-fidelity reference mesh against coarser or estimated reconstructions. It is specifically designed for evaluating TSDF (Truncated Signed Distance Field) or SDF-based volume representations, distinguishing itself from point-to-mesh Euclidean distance metrics by operating directly in the implicit field space [S1].

## How it works
**Inputs and Outputs**
- **Input**: A high-accuracy reference mesh (ground truth) and a set of reconstructed volume representations (estimated models) defined by their SDF functions.
- **Output**: A scalar value representing the mean absolute SDF error in length units (e.g., millimeters).

**Data Structures**
- **Reference Mesh**: A triangular mesh generated from the highest resolution reconstruction (e.g., 2 mm grid).
- **Sample Points**: A set of $N$ points $P_{est}$ sampled uniformly and randomly from the surface of the reference mesh.
- **SDF Function**: A function $SDF_{est}(p)$ that returns the signed distance from a 3D point $p$ to the zero-level set of the estimated reconstruction.

**Steps**
1. **Generate Reference Mesh**: Create a high-resolution mesh from the most accurate available reconstruction (e.g., using a 2 mm grid resolution) [S1].
2. **Sample Points**: Randomly sample $N$ points (e.g., 20 million) uniformly distributed on the surface of the reference mesh [S1]. Let this set be $P_{est}$.
3. **Evaluate SDFs**: For each point $p \in P_{est}$, evaluate the SDF value of the estimated/coarser reconstruction at that location: $d_p = SDF_{est}(p)$ [S1].
4. **Filter Outliers (Optional but Recommended)**: To exclude noise and outliers, rank the points by their distance to the nearest ground truth point and select only the 90% of points with the shortest distances [S2]. Let the filtered set be $P_{filtered}$. If this step is skipped, $P_{filtered} = P_{est}$ [G].
5. **Compute Mean Absolute Error**: Calculate the average of the absolute SDF values over the selected points.

**Formula**
The L1-absolute error is defined as:

$$
L1\text{-abs}(P_{est}, P_{gt}) = \frac{1}{|P_{filtered}|} \sum_{p \in P_{filtered}} |SDF_{est}(p)|
$$

Where:
- $P_{est}$ is the set of points sampled from the high-accuracy reference mesh.
- $P_{gt}$ represents the ground truth surface (implicitly defined by the zero-level set of the reference).
- $SDF_{est}(p)$ is the signed distance value of the estimated reconstruction at point $p$.
- $|P_{filtered}|$ is the number of points in the filtered set.

**Parameters**
- **Sampling Count**: 20 million points is used in the primary evaluation [S1].
- **Outlier Exclusion**: 90% of points with shortest distance to ground truth are retained [S2].
- **Reference Resolution**: 2 mm grid is used as the high-accuracy baseline in cited experiments [S1].

**Edge Cases and Pitfalls**
- **Outliers**: Raw SDF evaluations on noisy reconstructions can produce large errors for isolated noise points. The 90% filtering step is critical to mitigate this [S2].
- **Signed vs. Absolute**: The metric uses the absolute value $|SDF_{est}(p)|$, ignoring whether the point is inside or outside the volume, focusing purely on magnitude of deviation from the zero-level set [S1, S2].
- **Sampling Bias**: Uniform random sampling on the mesh surface is required to avoid biasing the error towards large or small triangles [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_l1_absolute_error(reference_mesh, sdf_function, num_samples=20_000_000, outlier_percentile=0.90):
    """
    Computes L1-absolute error between a reference mesh and an SDF-based reconstruction.
    
    Args:
        reference_mesh: Object supporting uniform random surface point sampling.
        sdf_function: Callable that takes a 3D point (x, y, z) and returns the SDF value.
        num_samples: Number of points to sample from the reference mesh.
        outlier_percentile: Fraction of points to keep (e.g., 0.90 for top 90% closest to GT).
        
    Returns:
        float: The mean absolute SDF error.
    """
    # Step 1 & 2: Sample points from the high-accuracy reference mesh
    points = reference_mesh.sample_uniform_surface_points(num_samples)
    
    # Step 3: Evaluate SDFs at these points
    sdf_values = []
    for p in points:
        # Evaluate SDF of the estimated/coarser reconstruction at point p
        val = sdf_function(p.x, p.y, p.z)
        sdf_values.append(val)
    
    # Step 4: Filter outliers
    # Rank points by distance to nearest ground truth point.
    # Note: In practice, this requires a ground truth point cloud or mesh for distance calculation.
    # If ground truth distance is not available, this step is skipped [G].
    # Here we assume we have a function to get distance to GT for each point.
    distances_to_gt = [distance_to_ground_truth(p) for p in points]
    
    # Sort indices by distance
    sorted_indices = sorted(range(len(distances_to_gt)), key=lambda k: distances_to_gt[k])
    
    # Select top percentile
    num_keep = int(len(points) * outlier_percentile)
    kept_indices = sorted_indices[:num_keep]
    
    kept_sdf_values = [sdf_values[i] for i in kept_indices]
    
    # Step 5: Compute mean absolute error
    if not kept_sdf_values:
        return 0.0
        
    mean_abs_error = sum(abs(v) for v in kept_sdf_values) / len(kept_sdf_values)
    
    return mean_abs_error

def distance_to_ground_truth(point):
    """
    Placeholder for calculating distance from a point to the nearest point on the ground truth surface.
    In the cited work, this is used to filter outliers [S2].
    """
    # Implementation depends on the specific ground truth representation (e.g., KD-tree search)
    # Not given in the sources. [G]
    pass
```

## Subtopics

### Completeness (Cpl)
Measures the fraction of ground truth points covered by the reconstruction within a threshold. It complements accuracy by measuring coverage rather than precision [S2].

### Chamfer Distance
A different metric that averages the nearest-neighbor distances between two point clouds, often used when SDFs are not available [S2].

## Operating characteristics
- **Time Complexity**: Dominated by the SDF evaluation for each sampled point. If $N$ is the number of samples and $T$ is the time per SDF query, complexity is $O(N \cdot T)$. Sorting for outlier removal adds $O(N \log N)$ [S2].
- **Memory Complexity**: $O(N)$ to store the sampled points and SDF values.
- **Measured Performance**: In evaluations relative to a 2 mm grid reference, L1 errors for various scenes using an 8 mm fixed grid ranged from 3.1 mm (Fire) to 9.5 mm (Stairs). Adaptive resolution methods reduced these errors significantly (e.g., Stairs from 9.5 mm to 7.6 mm) [S1].
- **Strengths**: Directly measures the quality of the implicit surface representation (zero-level set). Robust to noise when combined with outlier filtering [S2].
- **Limitations**: Requires a high-quality reference mesh for sampling. SDF evaluation can be computationally expensive for large volumes.

## Background
The metric was introduced in the context of evaluating hierarchical voxel block hashing for efficient integration of depth images, where it was used to compare fixed-resolution and adaptive-resolution TSDF reconstructions against a high-accuracy baseline [S1]. It has since been adopted in broader 3D reconstruction evaluations, including those involving Neural Radiance Fields (NeRFs) and aerial imagery, often alongside completeness metrics [S2].

## Sources
[S1] Hierarchical Voxel Block Hashing for Efficient Integration of Depth Images, https://doi.org/10.1109/lra.2015.2512958, fetched 2026-11-08
[S2] Leveraging Neural Radiance Fields for Large-Scale 3D Reconstruction from Aerial Imagery, https://doi.org/10.3390/rs16244655, fetched 2026-10-26
