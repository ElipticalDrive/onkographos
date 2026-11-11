# Supplementary Point Addition for Sparse Point Clouds

Also called: Supplementary Point Addition

## Overview
A densification technique designed to compensate for the sparsity of 3D Gaussians at the far end of scenes in defocus-blurry environments, where Structure-from-Motion (SfM) extraction is typically poor [S1]. It supplements the initial point cloud with synthetic points to ensure adequate coverage in regions where SfM fails to extract features, thereby improving reconstruction fidelity in large-depth-of-field scenarios [S1].

## How it works
**Inputs:**
- Existing 3D Gaussian point cloud (positions and colors).
- Hyperparameters: $N_{st}$ (start iteration), $N_p$ (number of points), $K$ (neighbors), $t_d$ (distance threshold).

**Outputs:**
- Updated point cloud including newly added supplementary points.

**Data Structures:**
- Point Cloud: A list of points, each containing a 3D position vector $\mathbf{p}$ and a color vector $\mathbf{c}$.

**Parameters:**
- $N_{st}$: Iteration number at which addition begins. Value: 2,500 [S1].
- $N_p$: Number of supplementary points. Value: Proportional to the extent of the point cloud, capped at 200,000 [S1].
- $K$: Number of neighbors for KNN interpolation. Value: 4 [S1].
- $t_d$: Minimum distance threshold for discarding points. Value: 2 [S1].

**Steps:**
1. **Check Iteration:** If the current training iteration $i < N_{st}$, terminate. Otherwise, proceed [S1].
2. **Determine Bounds:** Calculate $\alpha$ and $\beta$, the minimum and maximum values across all dimensions of the positions of the existing point cloud [S1].
3. **Sample Positions:** Generate $N_p$ candidate points by sampling uniformly from the distribution $U(\alpha, \beta)$ for each coordinate dimension [S1].
4. **Interpolate Colors:** For each candidate point $\mathbf{p}$, identify the $K$ nearest neighbors $P_{knn}$ from the existing point cloud based on Euclidean distance [S1].
5. **Assign Color:** Compute the color $\mathbf{c}_p$ for $\mathbf{p}$ by interpolating the colors of the points in $P_{knn}$ [S1].
6. **Filter by Distance:** Calculate the distance $d$ from $\mathbf{p}$ to its nearest neighbor in the existing point cloud. If $d > t_d$, discard $\mathbf{p}$. Otherwise, add $\mathbf{p}$ with color $\mathbf{c}_p$ to the point cloud [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def add_supplementary_points(existing_points, Nst, current_iteration, Np, K, td):
    """
    Adds supplementary points to the existing point cloud if the current iteration
    has reached Nst.
    
    Args:
        existing_points: List of dicts with 'pos' (vec3) and 'color' (vec3).
        Nst: Start iteration threshold.
        current_iteration: Current training iteration.
        Np: Number of points to sample.
        K: Number of neighbors for KNN.
        td: Distance threshold for filtering.
        
    Returns:
        Updated list of points.
    """
    if current_iteration < Nst:
        return existing_points

    # Step 2: Determine bounds
    # Extract all positions
    positions = [p['pos'] for p in existing_points]
    
    # Calculate min and max for each dimension
    # Assuming positions are 3D vectors [x, y, z]
    mins = [min(p[d] for p in positions) for d in range(3)]
    maxs = [max(p[d] for p in positions) for d in range(3)]
    
    new_points = []
    
    # Step 3: Sample Np points
    for _ in range(Np):
        # Sample uniform distribution U(alpha, beta) for each dimension
        # alpha is min, beta is max
        sample_pos = [random.uniform(mins[d], maxs[d]) for d in range(3)]
        
        # Step 4: Find K nearest neighbors
        # Calculate distances to all existing points
        distances = []
        for i, p in enumerate(existing_points):
            dist = euclidean_distance(sample_pos, p['pos'])
            distances.append((dist, i))
        
        # Sort by distance and take K nearest
        distances.sort(key=lambda x: x[0])
        k_nearest_indices = [idx for _, idx in distances[:K]]
        k_nearest_points = [existing_points[idx] for idx in k_nearest_indices]
        
        # Step 5: Interpolate color
        # Simple average of K nearest neighbors' colors
        avg_color = [0.0, 0.0, 0.0]
        for np_point in k_nearest_points:
            for d in range(3):
                avg_color[d] += np_point['color'][d]
        avg_color = [c / K for c in avg_color]
        
        # Step 6: Filter by distance
        # Distance to nearest neighbor is the first element in sorted distances
        nearest_dist = distances[0][0]
        
        if nearest_dist <= td:
            new_points.append({'pos': sample_pos, 'color': avg_color})
            
    # Append new points to existing
    updated_points = existing_points + new_points
    return updated_points

def euclidean_distance(p1, p2):
    return sum((a - b) ** 2 for a, b in zip(p1, p2)) ** 0.5
```

**Edge Cases and Pitfalls:**
- **Empty Point Cloud:** If the existing point cloud is empty, bounds $\alpha$ and $\beta$ are undefined. The method assumes a non-empty initialization from SfM [S1].
- **Precision:** Uniform sampling over large extents may result in points far from any existing geometry, which are filtered by $t_d$. If $t_d$ is too small, few points are added; if too large, noise may be introduced [S1].
- **Boundary Conditions:** The bounds $\alpha$ and $\beta$ are derived strictly from the existing point cloud. If the scene extends beyond the SfM reconstruction, these points will not be covered unless the bounds are manually expanded or the method is iterated [S1].

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N_p \cdot N)$, where $N$ is the number of existing points, due to the KNN search for each of the $N_p$ sampled points.
- **Memory Complexity:** $O(N + N_p)$ to store the updated point cloud.
- **Performance:** Effective in defocus-blurry scenes where SfM fails to extract points at the far end of the scene [S1].
- **Strengths:** Simple to implement; does not require depth estimation or learned models; improves fidelity in sparse regions [S1].
- **Limitations:** Adds a fixed number of points regardless of local geometry density; may introduce noise if $t_d$ is not tuned correctly; does not adapt to dynamic scene content [S1].
- **When to Use:** Defocus-blurry scenes with large depth of field where SfM extraction is poor at the far end [S1].
- **When Not to Use:** Scenes with dense SfM initialization; real-time applications where the overhead of KNN search is prohibitive [S1].

## Background
Originated in the context of deblurring 3D Gaussian Splatting, specifically to address the issue of sparse point clouds produced by blurry images in the DeblurNeRF dataset [S1]. The method was introduced to handle scenes with large depth of field, where SfM hardly extracts points on the far end of the scene [S1]. It builds on the standard 3D Gaussian Splatting framework, which manages Gaussian count through adaptive density control, but adds a proactive supplementation step [S1].

## Sources
[S1] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
