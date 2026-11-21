# Multi-Plane Projection Voxelization and Aggregation

Also called: MPP Pre-processing

## Overview
Multi-Plane Projection (MPP) Voxelization and Aggregation is a method for converting multi-view RGB-D images into a 3D volume representation by projecting point clouds onto a set of parallel planes and aggregating the data via distance-weighted blending [S1, unofficial]. It solves the problem of efficiently initializing volume data for neural rendering without the memory overhead of dense 3D voxel grids or the complexity of iterative reconstruction [S1, unofficial]. It is distinguished from its siblings by its explicit use of a fixed number of parallel planes to discretize space and its two-stage weighting scheme (horizontal and vertical distance) to handle occlusions and depth uncertainty [S1, unofficial].

## How it works
**Inputs:**
- A set of $N$ RGB-D images with known camera poses.
- For each image $i$: intrinsic matrix $K_i$, extrinsic matrix $E_i$, color image $I_i$, and depth image $D_i$.
- Hyperparameters: `num_planes` (number of parallel planes, typically 32), `a` (horizontal weight exponent, typically 1), `b` (vertical weight exponent, typically 1) [S1, unofficial].

**Outputs:**
- Voxelization data: `u`, `v`, `d` (coordinates and depths), `select_index`, `group_belongs`, `index_in_each_group`, `distance`, `each_split_max_num`, `delta` [S1, unofficial].
- Aggregation data: `weight_average`, `distance_to_depth_min` [S1, unofficial].

**Data Structures:**
The method uses NumPy arrays stored in `.npz` format.
- `point_clouds`: Shape `[4, M]`, homogeneous coordinates of projected points.
- `point_clouds_colors`: Shape `[3, M]`, RGB colors.
- `depth_masks`: Shape `[1, M]`, validity mask.
- Voxelization output keys: `u`, `v`, `d` (sorted coordinates), `select_index` (indices of valid points), `group_belongs` (plane index for each point), `index_in_each_group` (local index within a plane group), `distance` (vertical distance to depth minimum), `each_split_max_num` (max points per plane), `delta` (plane spacing) [S1, unofficial].
- Aggregation output keys: `weight_average` (normalized weights), `distance_to_depth_min` (vertical distances) [S1, unofficial].

**Steps:**

1. **Preprocessing and Projection:**
   For each image $i$:
   a. Read intrinsic matrix $K_i$ and extrinsic matrix $E_i$ [S1, unofficial].
   b. Check for invalid poses (e.g., $E_i[0,0] < -10^{10}$). Skip if invalid to prevent numerical errors [S1, unofficial].
   c. Optionally subsample pixels using a random mask with proportion `random_select_proportion` (typically 1/10) to reduce resource usage [S1, unofficial].
   d. Compute the projection matrix: $P_i = K_i E_i^{-1}$ [S1, unofficial].
   e. For each pixel $(u, v)$ in the image, compute homogeneous image coordinates $p_{img} = [u, v, 1, 1]^T$.
   f. Project to 3D: $p_{3d} = P_i^{-1} p_{img}$ (or equivalent back-projection using depth). The source code notes: `position_image = transform_matrix.dot(point_clouds)` where `transform_matrix` is derived from intrinsics and extrinsics [S1, unofficial].
   g. Extract depth $d = p_{3d}[2]$ (z-coordinate in camera space) [S1, unofficial].
   h. Accumulate all 3D points, colors, and depths into global arrays `point_clouds`, `point_clouds_colors`, `depth_masks` [S1, unofficial].

2. **Voxelization (Plane Assignment):**
   a. Define `num_planes` parallel planes along the viewing direction (typically Z-axis in world space or a canonical direction). The spacing `delta` is determined by the scene bounds [S1, unofficial].
   b. For each 3D point, determine which plane it belongs to based on its depth/position.
   c. Sort points by plane index, then by horizontal distance (u, v) within each plane.
   d. Store sorted coordinates `u_sorted`, `v_sorted`, depth `d_position_sorted`, and group indices [S1, unofficial].
   e. Compute `select_index` to identify valid points and `group_belongs` to map points to planes [S1, unofficial].

3. **Aggregation (Weighting):**
   a. Load voxelization data: `select_index`, `index_in_each_group`, `distance` [S1, unofficial].
   b. For each group (plane), compute the minimum depth: $d_{min} = \min(d_{selected})$ [S1, unofficial].
   c. Compute vertical distance for each point: $dist_{vert} = d - d_{min}$ [S1, unofficial].
   d. Compute horizontal distance weight: $w_1 = (1 - dist_{horiz})^a$. Note: The source code uses `1-distance` where `distance` likely refers to a normalized horizontal distance or a placeholder for horizontal proximity. Given the context of "horizontal distance weighting", this term penalizes points far from the center or reference point in the u-v plane [S1, unofficial].
   e. Compute vertical distance weight: $w_2 = \frac{1}{1 + dist_{vert}^b}$ [S1, unofficial].
   f. Combine weights: $w_{renew} = w_1 \times w_2$ [S1, unofficial].
   g. Normalize weights per group: $w_{norm} = \frac{w_{renew}}{\sum w_{renew}}$ [S1, unofficial].
   h. Return `weight_average` and `distance_to_depth_min` [S1, unofficial].

4. **Volume Rendering (Accumulation):**
   The aggregated weights are used in the volume rendering loop. For sample $i$ along a ray:
   $$ C = \sum_{i=0}^{N-1} T_i \cdot (1 - \exp(-\sigma_i \cdot \delta_i)) \cdot c_i $$
   where $T_i = \exp(-\sum_{j=0}^{i-1} \sigma_j \cdot \delta_j)$ is the transmittance, $\sigma_i$ is the density, $\delta_i$ is the step size, and $c_i$ is the color [S1, unofficial].
   The code implements this as:
   ```python
   final = final + (tf.exp(-tf.reduce_sum(sigma[:, 0:i, :, :, :] * delta[i], axis=1))) * (1 - tf.exp(-(sigma[:, i, :, :, :] * delta[i]))) * conv12[:, i, :, :, :]
   ```
   [S1, unofficial].

**Pitfalls:**
- Invalid camera poses (e.g., containing `-inf`) must be skipped [S1, unofficial].
- Data types must be managed carefully; `np.uint32` is resource-consuming for storage [S1, unofficial].
- Random subsampling reduces accuracy but saves memory; balance `random_select_proportion` based on available resources [S1, unofficial].

## Subtopics

### [Texture Spreading Backprojection](16610-texture-spreading-backprojection.md)
A GPU-accelerated variant that maps projection data onto volume slices via texture mapping, eliminating the need for a final merge step but requiring all projections to remain in memory.

## Operating characteristics
- **Time Complexity:** Dominated by the projection of all pixels from all views to 3D space and the sorting/aggregation steps. Subsampling reduces this significantly.
- **Memory Complexity:** High, as it stores all projected points and intermediate arrays. `np.uint32` usage increases memory footprint [S1, unofficial].
- **Performance:** Efficient for scenes with moderate complexity and limited memory compared to dense voxel grids. Scaling is linear with the number of pixels and views.
- **Strengths:** Simple implementation, GPU-friendly aggregation, handles occlusions via depth-based weighting.
- **Limitations:** Requires accurate depth maps; sensitive to noise in depth; memory-intensive for high-resolution images without subsampling [S1, unofficial].
- **When to use:** When initializing NeRF-like models from RGB-D data with limited memory.
- **When not to use:** When high-precision geometry is required without depth sensors, or when memory is extremely constrained and subsampling degrades quality too much.

## Background
The method originates from the PCMP-NeRF project, which proposes a pre-processing pipeline for neural radiance fields using multi-plane projection [S1, unofficial]. It is designed to handle Matterport3D and ScanNet datasets, leveraging undistorted RGB-D images and camera parameters.

## Sources
[S1] PCMP-NeRF, https://github.com/Mayxmu/PCMP-NeRF, https://github.com/Mayxmu/PCMP_NeRF, fetched 2026-10-30 (unofficial)
