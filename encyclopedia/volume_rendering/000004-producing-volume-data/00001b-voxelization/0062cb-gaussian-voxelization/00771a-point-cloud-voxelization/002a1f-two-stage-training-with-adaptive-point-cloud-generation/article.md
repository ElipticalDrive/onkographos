# Two-stage training with adaptive point cloud generation

Also called: Adding extra points, Supplemental point cloud generation

## Overview
This method expands a sparse scene representation by generating a supplemental point cloud based on the bounding volume of existing points [S1]. It splits training into two stages, using noise-injected UDF gradient descent to densify the point cloud after the first stage [S2, unofficial].

## How it works
**Inputs:**
- Initial point cloud positions $P_{init} \in \mathbb{R}^{N \times 3}$ from stage 1 [S2, unofficial].
- Ground truth points $P_{gt}$ [S2, unofficial].
- Hyperparameters: stability constant $c=1.1$, maximum supplemental points $N_{max}=200,000$, noise scaling factor `low_range`, UDF threshold `df_filter`, outlier distance `outlier` [S1][S2, unofficial].

**Outputs:**
- Combined point cloud $P_{combined}$ for stage 2 training [S2, unofficial].

**Data Structures:**
- Bounding box defined by $Q_{min}$ and $Q_{max}$, the element-wise minimum and maximum coordinates of $P_{init}$ [S1].
- `cKDTree` for efficient nearest-neighbor queries during stage 2 [S2, unofficial].

**Step 1: Calculate Supplemental Point Count**
Determine the number of points $N_p$ to generate based on the volume of the bounding box of the initial point cloud [S1]:
$$
N_p = \text{min}\left(\frac{\text{prod}(Q_{max} - Q_{min})}{c^3}, N_{max}\right)
$$
where $\text{prod}(\cdot)$ returns the product of all values in the vector $(Q_{max} - Q_{min})$ [S1].

**Step 2: Generate Supplemental Points**
The process samples points, computes offsets and standard deviation, generates noise scaled by `low_range`, adds noise to points, computes UDF and gradient, projects samples to surface, filters by `udf_sample < df_filter`, and repeats until `num_points` is reached [S2, unofficial].

**Step 3: Concatenate and Rebuild**
Concatenate ground truth points with generated points, build `cKDTree`, and update dataset for stage 2 training [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gen_extra_pointcloud(P_init, P_gt, N_max=200000, c=1.1, low_range=0.01, df_filter=0.05, outlier=0.1, num_points=None, max_iterations=1000, learning_rate=0.01):
    # Step 1: Calculate N_p
    Q_min = P_init.min(axis=0)
    Q_max = P_init.max(axis=0)
    bbox_volume = np.prod(Q_max - Q_min)
    N_p = min(bbox_volume / (c ** 3), N_max)
    if num_points is None:
        num_points = int(N_p)

    # Step 2: Generate points via UDF gradient descent
    # [G] Initialize samples on a grid within the bounding box
    grid_dim = int(np.ceil(num_points ** (1/3)))
    coords = np.linspace(Q_min, Q_max, grid_dim)
    X, Y, Z = np.meshgrid(coords[:, 0], coords[:, 1], coords[:, 2], indexing='ij')
    samples = np.stack([X.ravel(), Y.ravel(), Z.ravel()], axis=1)

    generated_points = []
    for _ in range(max_iterations):
        # [Comment: UDF computation and gradient descent details are implementation-specific and not provided in the sources]
        udf_values = np.zeros(len(samples))  # Placeholder for actual UDF values
        gradients = np.zeros_like(samples)    # Placeholder for actual gradients
        noise = np.random.normal(0, low_range, samples.shape)
        samples = samples - gradients * learning_rate + noise
        valid_mask = udf_values < df_filter
        samples = samples[valid_mask]
        if len(samples) >= num_points:
            break
    generated_points = samples[:num_points]

    # Step 3: Remove outliers
    # [G] Compute distances to nearest neighbor in P_gt and filter
    if len(P_gt) > 0 and len(generated_points) > 0:
        dists = np.linalg.norm(generated_points[:, np.newaxis, :] - P_gt[np.newaxis, :, :], axis=2)
        min_dists = dists.min(axis=1)
        generated_points = generated_points[min_dists < outlier]

    # Concatenate and rebuild KDTree
    P_combined = np.concatenate([P_gt, generated_points], axis=0)
    # [Comment: cKDTree construction for stage 2 sampling is implementation-specific]
    return P_combined
```

## Operating characteristics
Memory complexity increases by $N_p$ points, which is capped at 200,000 [S1].

## Background
The method was introduced in "Deblurring 3D Gaussian Splatting" by B. Lee et al. [S1]. It addresses sparse point clouds in Gaussian Splatting by supplementing points to improve reconstruction quality [S1].

## Sources
[S1] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
[S2] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
