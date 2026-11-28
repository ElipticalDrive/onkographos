# L1-absolute Error (Accuracy)

Also called: Average of absolute differences between SDF values, Experimental Evaluation Metrics, L1-abs Error, Mean Absolute Error (Point Cloud), Mesh Sampling Error, TSDF Accuracy Measurement

## Overview
L1-absolute Error quantifies geometric accuracy by computing the average absolute difference between Signed Distance Field (SDF) values of an estimated reconstruction and a ground truth reference [S1, S2]. It operates directly in implicit field space rather than computing point-to-mesh Euclidean displacements, distinguishing it from metrics like [ROC Geometry Matching Evaluation](14003-roc-geometry-matching-evaluation.md) [S1].

## How it works
**Inputs and Outputs**
- **Input**: A high-accuracy reference mesh generated from the finest reconstruction (e.g., 2 mm grid) and an SDF function representing the estimated or coarser reconstruction [S1].
- **Output**: A scalar mean absolute error in length units (e.g., millimeters) [S1].

**Data Structures**
- **Reference Mesh**: Triangular mesh from the highest-resolution reconstruction [S1].
- **Sample Points**: Set $P_{est}$ of $N$ points randomly sampled on the reference mesh surface [S1].
- **SDF Function**: $SDF_{est}(p)$ returning the signed distance from point $p$ to the zero-level set of the estimated reconstruction [S1].

**Steps**
1. Generate a high-accuracy reference mesh from the finest available reconstruction (e.g., 2 mm grid) [S1].
2. Randomly sample $N$ points (e.g., 20 million) on the reference mesh surface to form $P_{est}$ [S1].
3. Evaluate the SDF of the estimated reconstruction at each point $p \in P_{est}$ to obtain $d_p = SDF_{est}(p)$ [S1].
4. Filter outliers by ranking points by their distance to the nearest ground truth point and retaining the 90% with the shortest distances [S2].
5. Compute the mean absolute SDF value over the filtered points [S1, S2].

**Formula**
The L1-absolute error is defined as:

$$
L1\text{-abs}(P_{est}, P_{gt}) = \frac{1}{|P_{filtered}|} \sum_{p \in P_{filtered}} |SDF_{est}(p)|
$$

Where $P_{filtered}$ is the set of points after outlier removal, and $SDF_{est}(p)$ is the signed distance value at $p$ [S1, S2]. The general form in [S2] is $\frac{1}{|P_{est}|} \sum_{p \in P_{est}} |P_{est}(p) - P_{gt}(p)|$, which simplifies to absolute SDF values when points lie on the ground truth zero-level set [S1, S2].

**Parameters**
- **Sampling count**: 20 million points [S1].
- **Outlier retention**: 90% of points with shortest distance to ground truth [S2].
- **Reference resolution**: 2 mm grid [S1].

**Edge Cases and Pitfalls**
- **Outliers**: Noisy reconstructions produce isolated large SDF values; the 90% filtering step mitigates this [S2].
- **Signed vs. Absolute**: The metric takes the absolute value $|SDF_{est}(p)|$, discarding inside/outside sign to measure pure magnitude of deviation from the zero-level set [S1, S2].
- **Ground truth distance calculation**: The method requires distances from sampled points to the nearest ground truth point for filtering, but the sources do not specify the algorithm used [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_l1_absolute_error(reference_mesh, sdf_function, gt_point_cloud=None, num_samples=20_000_000, keep_percentile=0.90):
    # Step 1 & 2: Sample points from the high-accuracy reference mesh
    points = reference_mesh.sample_uniform_surface_points(num_samples)
    
    # Step 3: Evaluate SDFs at these points
    sdf_values = [sdf_function(p.x, p.y, p.z) for p in points]
    
    # Step 4: Filter outliers
    # The sources do not specify the algorithm for computing distances to the ground truth [G].
    if gt_point_cloud is not None:
        # Standard Euclidean distance calculation for filtering
        distances_to_gt = [min((p.x - g.x)**2 + (p.y - g.y)**2 + (p.z - g.z)**2 for g in gt_point_cloud) for p in points]
        sorted_indices = sorted(range(len(distances_to_gt)), key=lambda k: distances_to_gt[k])
        num_keep = int(len(points) * keep_percentile)
        kept_indices = sorted_indices[:num_keep]
        kept_sdf_values = [sdf_values[i] for i in kept_indices]
    else:
        kept_sdf_values = sdf_values
        
    # Step 5: Compute mean absolute error
    if not kept_sdf_values:
        return 0.0
    return sum(abs(v) for v in kept_sdf_values) / len(kept_sdf_values)
```

## Subtopics

### Completeness (Cpl)
Measures the fraction of ground truth points covered by the reconstruction within a specified threshold (e.g., 0.2 m) [S2]. It quantifies coverage rather than precision and is typically reported alongside L1-absolute Error to evaluate both reconstruction fidelity and spatial extent [S2].

## Operating characteristics
- **Time complexity**: $O(N \cdot T)$ for $N$ samples and $T$ time per SDF query, plus $O(N \log N)$ for sorting during outlier filtering [S2].
- **Memory complexity**: $O(N)$ to store sampled points and SDF values [G].
- **Measured performance**: Relative to a 2 mm grid reference, 8 mm fixed-grid reconstructions yielded errors from 3.1 mm (Fire) to 9.5 mm (Stairs), while adaptive resolution reduced errors to 2.2 mm (Fire) and 7.6 mm (Stairs) [S1].
- **Strengths**: Directly evaluates implicit surface quality at the zero-level set; robust to noise when combined with percentile filtering [S1, S2].
- **Limitations**: Requires a high-quality reference mesh for sampling; SDF evaluation scales linearly with sample count and can be costly for large volumes [S1].

## Background
Introduced to evaluate hierarchical voxel block hashing for depth image integration, comparing fixed 8 mm and adaptive resolution TSDF reconstructions against a 2 mm baseline [S1]. It is used in aerial imagery reconstruction pipelines alongside completeness metrics to assess neural radiance field (NeRF) and voxel-based outputs [S2].

## Sources
[S1] Hierarchical Voxel Block Hashing for Efficient Integration of Depth Images, https://doi.org/10.1109/lra.2015.2512958, fetched 2026-11-08
[S2] Leveraging Neural Radiance Fields for Large-Scale 3D Reconstruction from Aerial Imagery, https://doi.org/10.3390/rs16244655, fetched 2026-10-26
