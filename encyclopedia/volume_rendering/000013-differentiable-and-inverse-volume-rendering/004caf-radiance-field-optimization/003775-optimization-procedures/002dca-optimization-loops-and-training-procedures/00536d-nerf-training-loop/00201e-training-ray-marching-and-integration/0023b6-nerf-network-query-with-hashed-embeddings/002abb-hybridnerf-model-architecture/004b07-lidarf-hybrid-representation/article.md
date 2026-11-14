# LidaRF Hybrid Representation

Also called: LidaRF, Lidar-NeRF with Hybrid Representation

## Overview
LidaRF is a NeRF variant for street scenes that fuses explicit Lidar point cloud features with an implicit hash-based grid representation to improve geometry and rendering quality under sparse and collinear camera trajectories [S1]. It differs from prior Lidar-only NeRF approaches by avoiding low-resolution and incomplete rendering caused by sparse Lidar coverage and limited field of view; instead, it combines explicit Lidar geometry with a high-resolution implicit grid [S1].

## How it works
**Inputs:**
1.  Lidar point clouds from all frames of the entire sequence [S1].
2.  Camera poses and intrinsics for the sequence [S1].
3.  Hash-based feature grid parameters (resolution, hash table size, MLP architecture) [S1].

**Outputs:**
1.  Color and density for sampled points along rays [S1].
2.  Denser Lidar points accumulated across frames used as depth supervision [S1].

**Data Structures:**
*   **Voxel Grid:** A 3D grid where each cell stores a 3-dimensional feature vector representing the averaged spatial position of all Lidar points assigned to that cell [S1].
*   **Lidar Embeddings ($P$):** A set $P = \{(p_i, f_i) | i = 1, ..., N\}$, where $p_i$ is the 3D location of point $i$ and $f_i$ is the $n$-dimensional neural feature vector of the voxel cell it resides in [S1].
*   **Hash-based Feature Grid:** Implicit grid storing features for continuous 3D coordinates, fused with Lidar features [S1].

**Steps:**
1.  **Aggregate Lidar Points:** Aggregate Lidar point clouds from all frames of the entire sequence to construct a denser point cloud set [S1].
2.  **Voxelize and Average:** Voxelize the aggregated point clouds into a voxel grid. For each voxel cell, average the spatial positions of all points falling into that cell to produce a single 3-dimensional feature vector per voxel [S1].
3.  **Encode Geometry:** Encode the scene geometry features using a 3D sparse UNet operating on the voxel grid to capture local and global contextual information [S1]. The 3D sparse UNet takes the voxel grid along with its 3-dim features as input and outputs neural volumetric features, consisting of $n$-dim feature for each occupied voxel [S1].
4.  **Fuse Features:** Fuse the extracted Lidar features with the hash-based feature grid. Feed the hybrid features into MLPs to decode color and density [S1].
5.  **Render:** Apply standard volume rendering using the decoded color and density [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lidaRF_pipeline(lidar_points_all_frames, camera_poses, grid_params):
    # Step 1: Aggregate Lidar Points
    aggregated_points = aggregate_all_frames(lidar_points_all_frames)
    
    # Step 2: Voxelize and Average
    voxel_grid = create_voxel_grid(aggregated_points)
    voxel_features = {}
    for point in aggregated_points:
        voxel_id = get_voxel_id(point.position, voxel_grid.resolution)
        if voxel_id not in voxel_features:
            voxel_features[voxel_id] = []
        voxel_features[voxel_id].append(point.position)
        
    averaged_features = {}
    for voxel_id, positions in voxel_features.items():
        avg_pos = sum(positions) / len(positions)
        averaged_features[voxel_id] = avg_pos # 3-dim feature vector
    
    # Step 3: Encode Geometry with 3D Sparse UNet
    # Input: voxel grid structure + averaged_features
    # Output: neural volumetric features (n-dim) for occupied voxels
    lidar_embeddings = sparse_unet_encode(voxel_grid, averaged_features)
    
    # Step 4: Fuse Features and Decode
    # For each sample point x along a ray:
    def query_and_decode(x, ray):
        # Query hash-based grid for base features
        hash_features = query_hash_grid(x, grid_params)
        
        # Query Lidar features if available
        lidar_feature = query_lidar_feature(x, lidar_embeddings)
        
        # Fuse features
        hybrid_feature = fuse(hash_features, lidar_feature)
        
        # Decode color and density via MLP
        color, density = mlp_decode(hybrid_feature)
        return color, density

    # Step 5: Standard Volume Rendering
    rendered_images = []
    for ray in rays:
        samples = sample_along_ray(ray)
        colors = []
        densities = []
        for sample in samples:
            c, d = query_and_decode(sample, ray)
            colors.append(c)
            densities.append(d)
        image_color = volume_render(colors, densities)
        rendered_images.append(image_color)
        
    return rendered_images
```

**Parameters:**
*   **Voxel Grid Resolution:** Not given in the sources.
*   **Hash Grid Resolution/Size:** Not given in the sources.
*   **UNet Channels ($n$):** Not given in the sources.
*   **Loss Weights:** The total loss $L$ includes $L_{nerfacto}$, $L_{ds}$ (depth supervision), and $L_{aug}$ (augmentation) [S1]. Specific weights are not given in the sources.

**Edge Cases and Pitfalls:**
*   **Sparse Lidar Coverage:** Relying on Lidar features alone results in low-resolution rendering [S1].
*   **Limited Field of View:** Lidar does not cover the entire scene (e.g., building surfaces above a certain height), yielding blank rendering in those regions [S1]. The hybrid approach mitigates this by using the hash grid for uncovered regions [S1].
*   **Occlusion:** Depth supervision accounts for occlusion [S1].

## Subtopics

### Shading Replacement for Relighting
A method for relighting scenes by replacing shading components. It differs from LidaRF by focusing on material and lighting manipulation rather than geometry reconstruction from Lidar. [Shading Replacement for Relighting](6346-shading-replacement-for-relighting.md)

### Hybrid Multi-Space Module
A module for combining features from multiple spatial representations. It differs from LidaRF by being a general module rather than a specific Lidar-NeRF fusion architecture. [Hybrid Multi-Space Module](11712-hybrid-multi-space-module.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Combines complementary benefits of explicit Lidar geometry and high-resolution implicit grid for high-quality and complete scene rendering [S1].
*   **Limitations:** Requires Lidar data. Performance depends on the quality of Lidar aggregation and voxelization.
*   **When to Use:** Street scenes with sparse and collinear camera trajectories where Lidar data is available.
*   **When Not to Use:** Scenes without Lidar data or where high-resolution implicit grid alone is sufficient.

## Background
LidaRF was introduced in the paper "LidaRF: Delving into Lidar for Neural Radiance Field on Street Scenes" [S1]. It addresses the limitations of prior Lidar-only NeRF approaches, which suffer from low-resolution rendering due to sparse Lidar points and incomplete coverage due to limited field of view [S1]. The method leverages denser Lidar points accumulated across frames as depth supervision while accounting for occlusion [S1].

## Sources
[S1] LidaRF: Delving into Lidar for Neural Radiance Field on Street Scenes, https://doi.org/10.1109/cvpr52733.2024.01850, fetched 2026-10-26
