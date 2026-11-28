# Large Scene Reconstruction via Chunking

Also called: Chunk-based Reconstruction, Scene Partitioning

## Overview
Large Scene Reconstruction via Chunking addresses the physical memory limits and scalability challenges inherent in 3D Gaussian Splatting (3DGS) when applied to massive environments [S1]. By partitioning the scene into spatially disjoint or distributed regions, it enables the reconstruction of datasets that would otherwise exceed GPU memory capacities [S1]. The method splits scenes into chunks based on camera distribution and SfM reconstruction, uniform grids, or dynamic hierarchies, and handles boundaries via skyboxes or pruning [S1, S2]. It supports distributed training across multiple GPUs with dynamic load balancing [S1].

## How it works
### Inputs and Outputs
*   **Inputs:** Input images with known or estimated poses (SfM data) and camera intrinsics [S1, G].
*   **Outputs:** A set of 3D Gaussians representing the scene, organized by chunk or distributed across devices [S1].

### Data Structures
The core data structure is the **Chunk**, representing a spatial region of the scene [S1]. Each chunk stores a subset of 3D Gaussians with attributes: position ($\mathbf{x}$), covariance ($\mathbf{\Sigma}$), opacity ($\alpha$), spherical harmonics coefficients ($\mathbf{c}$), and scale ($\mathbf{s}$) [G]. Gaussians are assigned to chunks based on spatial location or camera positions [S1].

### Steps
1.  **Scene Partitioning:** Divide the scene into chunks. Strategies include splitting based on camera distribution and SfM reconstruction [S1], using a uniform spatial grid [S1], or creating a dynamic hierarchy ranging from 1 to 8 levels depending on scene size [S2].
2.  **Background Handling:** Define a low-resolution skybox on a sphere surrounding each chunk to treat the surrounding scene as background [S1]. For uniform grids, prune chunks with low rendering contribution to reduce memory usage [S1].
3.  **Training:** Train each chunk independently on a single GPU, or distribute Gaussians across multiple GPUs [S1]. Assign pixels to GPUs in tiles and transfer Gaussians between them as needed [S1]. Apply dynamic load balancing to scale training efficiently [S1].
4.  **Merging:** If using disjoint chunks, merge the resulting Gaussian sets into a final scene representation [S1]. If using a hierarchy, manage levels to maintain memory constraints [S2].

### Parameters
*   **Maximum GPU Count:** Distributed training can scale to up to 32 GPUs [S1].
*   **Hierarchy Levels:** Ranges from 1 for small scenes to 8 for large scenes like CityWalk [S2].

### Edge Cases and Pitfalls
*   **Out-of-Memory (OOM):** Processing large scenes without a hierarchy causes OOM errors on standard 24GB GPUs after processing a fraction of frames (e.g., 795 out of 4050 frames for CityWalk) with Gaussian counts exceeding 6.7 million [S2].
*   **Boundary Artifacts:** Improper boundary handling can cause artifacts, mitigated by the skybox method [S1].
*   **Pose Estimation:** Pure rotations prevent triangulation, a limitation shared with COLMAP and ORB-SLAM [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def reconstruct_large_scene(input_images, sfm_data, max_gpus=32, use_hierarchy=False):
    # Step 1: Partition Scene
    if use_hierarchy:
        chunks = []
        for level in range(1, 9):
            chunks.extend(split_scene_into_levels(sfm_data, level))
    else:
        chunks = split_by_camera_distribution(sfm_data)
    
    # Step 2: Initialize Background & Pruning
    for chunk in chunks:
        chunk.skybox = create_sphere_skybox(chunk.bounds, resolution=64)
        if not use_hierarchy:
            chunk.contribution = estimate_rendering_contribution(chunk.gaussians, input_images)
    
    # Step 3: Training
    if max_gpus > 1:
        gpu_workers = [Worker(device=i) for i in range(max_gpus)]
        for chunk in chunks:
            tile_assignments = assign_pixels_to_gpus(chunk, gpu_workers)
            gaussian_shards = distribute_gaussians(chunk.gaussians, gpu_workers)
            for worker in gpu_workers:
                worker.train(chunk, gaussian_shards[worker.id], tile_assignments[worker.id])
    else:
        for chunk in chunks:
            if chunk.contribution < 0.1:
                continue
            train_chunk_single(chunk, chunk.skybox)
            
    # Step 4: Merge Results
    final_gaussians = []
    for chunk in chunks:
        final_gaussians.extend(chunk.gaussians)
    return final_gaussians

def split_by_camera_distribution(sfm_data):
    return [Chunk(cameras) for cameras in group_cameras_by_region(sfm_data.cameras)]

def split_scene_into_levels(sfm_data, level):
    bounds = sfm_data.scene_bounds
    cells = 2 ** level
    return [Chunk(bounds, cells) for _ in range(cells ** 3)]

def create_sphere_skybox(bounds, resolution=64):
    return SphereTexture(bounds.center, bounds.radius, resolution)

def estimate_rendering_contribution(gaussians, images):
    return sum(g.coverage(img) for g in gaussians) / len(images)

def assign_pixels_to_gpus(chunk, workers):
    tiles = chunk.pixel_grid.split_into_tiles(len(workers))
    return [tiles[i % len(workers)] for i in range(len(workers))]

def distribute_gaussians(gaussians, workers):
    shards = [[] for _ in range(len(workers))]
    for g in gaussians:
        shards[g.worker_id % len(workers)].append(g)
    return shards

def train_chunk_single(chunk, skybox):
    optimize_gaussians(chunk.gaussians, chunk.images, skybox)
```

## Subtopics

### Progressive Hierarchical Memory Management
A variant that applies a lightweight progressive hierarchy to dynamically create levels throughout reconstruction [S2]. It ranges from 1 level for small scenes to 8 levels for large scenes like CityWalk [S2]. This method maintains peak trained Gaussians below 3.7 million and allocated memory below 12.8 GB, preventing OOM errors that occur without hierarchy (e.g., 6.7 million Gaussians causing OOM on 24GB GPU) [S2]. It trades a small quality gap versus offline approaches for orders-of-magnitude faster processing and enables scalability to large scenes [S2].

## Operating characteristics
*   **Memory Complexity:** Scales with the number of chunks and the hierarchy depth [S2]. Without chunking or hierarchy, memory usage grows linearly with scene size, leading to OOM on standard GPUs (e.g., 24GB) for large scenes [S2]. With chunking, memory is bounded by the chunk size and hierarchy limits (e.g., <12.8 GB with hierarchy) [S2].
*   **Time Complexity:** Distributed training across multiple GPUs (up to 32) improves training speed [S1]. Dynamic load balancing further optimizes this [S1]. Hierarchical approaches offer orders-of-magnitude faster processing compared to offline methods [S2].
*   **Strengths:** Enables reconstruction of massive scenes that exceed single-GPU memory [S1, S2]. Supports distributed training [S1].
*   **Limitations:** Potential quality gap compared to monolithic offline approaches [S2]. Boundary artifacts if skybox or pruning is not handled well [S1]. Pose estimation limitations for pure rotations [S2].
*   **When to Use:** Large-scale scenes where memory is a constraint [S1, S2]. When distributed hardware is available [S1]. When processing speed is prioritized over absolute offline optimality [S2].
*   **When Not to Use:** Small scenes where monolithic reconstruction is sufficient and faster [S2]. Scenes with pure rotations where pose estimation fails [S2].

## Background
The challenge of scaling 3D Gaussian Splatting to large scenes was highlighted by Kerbl et al. [18], who noted the lack of huge datasets and the memory limitations of the original method [S1]. Lin et al. [19] proposed partitioning scenes based on camera distribution and SfM reconstruction [S1]. Liu et al. [20] introduced uniform chunking with pruning [S1]. Kerbl et al. [18] also introduced the skybox approach for background handling [S1]. Zhao et al. [21] explored multi-GPU distribution with dynamic load balancing [S1]. More recently, Meulemann et al. [22] introduced on-the-fly processing for unposed data [S1]. The Progressive Hierarchical Memory Management approach was introduced to address OOM issues in immediate reconstruction of unordered input [S2].

## Sources
[S1] Virtual memory for 3D Gaussian Splatting, https://zenodo.org/records/19690968, fetched 2026-10-26
[S2] Immediate 3D Gaussian Splat Reconstruction of Unordered Input with Global Consistency, https://doi.org/10.1145/3799902.3811167, fetched 2026-10-26
