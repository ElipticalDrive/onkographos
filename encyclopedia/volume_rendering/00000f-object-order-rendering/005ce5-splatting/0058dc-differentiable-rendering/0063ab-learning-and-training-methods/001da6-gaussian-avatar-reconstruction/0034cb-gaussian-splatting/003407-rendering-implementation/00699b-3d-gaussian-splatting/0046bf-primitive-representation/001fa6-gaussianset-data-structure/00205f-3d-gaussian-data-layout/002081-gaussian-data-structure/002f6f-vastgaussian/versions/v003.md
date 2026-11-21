# VastGaussian


## Overview
VastGaussian is a large-scale 3D Gaussian Splatting method introduced in CVPR'24 that partitions a scene into spatial regions to handle physical memory limits and varying lighting conditions [S2, unofficial][S4]. It addresses scalability by training independent 3DGS models per block and merging them at rendering time with feathered overlap [S1, unofficial][S2, unofficial]. Unlike standard grid-based voxelization, it uses a block-based partitioning strategy that allows for decoupled appearance handling across regions [S2, unofficial].

## How it works
**Inputs:** Training images with camera poses (SfM reconstruction) and an initial sparse point cloud [S4].
**Outputs:** Optimized 3D Gaussian parameters (position, covariance, opacity, spherical harmonics) organized into spatial blocks, plus a merging structure for rendering [S1, unofficial][S4].

**Data Structures:** The scene is partitioned into spatial blocks, described as either spatially-disjoint [S2, unofficial] or overlapping [S1, unofficial]. Each block stores a subset of 3D Gaussians and their parameters [S4]. Overlap regions between adjacent blocks are handled by a merging structure [S2, unofficial]. Exact field layouts, sizes, and bit-packing schemes are Not given in the sources.

**Steps:**
1. **Partitioning:** Divide the scene into spatial blocks or chunks. Partitioning can be based on the distribution of training camera positions and the respective SfM reconstruction [S4]. Alternatively, uniform chunks can be used [S4].
2. **Training:** Train an independent 3DGS model for each block. This optimizes Gaussian parameters (position, scale, rotation, opacity, color) for Gaussians assigned to that block using visible training images [S1, S2, unofficial].
3. **Appearance Decoupling:** Handle varying lighting conditions by decoupling appearance handling across blocks, allowing each block to optimize its own lighting characteristics independently [S2, unofficial].
4. **Merging:** At rendering time, merge results from adjacent blocks using a feathered overlap technique to ensure smooth transitions and avoid visible seams [S2, unofficial]. Overlap regions are blended based on distance to block boundaries [S2, unofficial].

**Formulas:** Not given in the sources.

**Parameters:**
* **Block Size:** Not given in the sources.
* **Overlap Width:** Not given in the sources.
* **Maximum GPU Count:** Up to 32 GPUs for distributed training [S4].

**Edge Cases and Pitfalls:**
* **Boundary Artifacts:** Incorrect feathered overlap implementation can cause visible seams between blocks [S2, unofficial].
* **Memory Usage:** Training independent models per block increases memory usage during training compared to a single global model [S4].
* **Gaussian Distribution:** Uneven distribution of Gaussians across blocks can lead to imbalanced training times and rendering performance [S4].

## Subtopics

### Progressive Partitioning for Large-Scale 3DGS
A strategy to divide a large scene into multiple cells where training cameras and 3D Gaussians are properly distributed [S3]. It uses an airspace-aware visibility criterion to distribute training cameras and 3D Gaussians, ensuring that each cell contains relevant data for its region [S3]. This method improves the efficiency of large-scale reconstruction by balancing the load across cells [S3].

### Large Scene Reconstruction via Chunking
Partitioning a 3D Gaussian Splatting scene into manageable chunks to handle physical memory limits and scalability challenges in large scenes [S4]. Various approaches exist: Lin et al. split scenes into chunks based on the distribution of camera positions and the respective Structure from Motion (SfM) reconstruction [S4]. Liu et al. split scenes into uniform chunks and prune chunks with low contribution to rendering [S4]. Kerbl et al. define a low-resolution skybox on a sphere surrounding each chunk, defining the surrounding scene as background [S4]. Zhao et al. distribute Gaussians across multiple GPUs, assigning pixels to be rendered to GPUs in tiles, and transferring the Gaussians between them as needed [S4].

## Operating characteristics
* **Time Complexity:** Training time is distributed across blocks, potentially reducing total training time if parallelized [S4]. Rendering time depends on the number of visible Gaussians and the overhead of merging blocks [S1, S2, unofficial].
* **Memory Complexity:** Memory usage is reduced per block during training, allowing for larger scenes to be processed [S4]. However, the total memory footprint may increase due to the need to store multiple block models and merging structures [S4].
* **Measured Performance:** Scales to large scenes with millions of Gaussians [S1, S2, unofficial]. Distributed training can scale to up to 32 GPUs [S4].
* **Strengths:** Handles large-scale scenes, decouples appearance for varying lighting, reduces memory pressure during training [S2, unofficial][S4].
* **Limitations:** Potential for seams if merging is not handled well, increased complexity in implementation [S2, unofficial].
* **When to Use:** Large outdoor scenes, city-scale reconstructions, scenes with varying lighting conditions [S2, unofficial].
* **When Not to Use:** Small scenes where the overhead of partitioning is not justified, scenes with uniform lighting where a single global model suffices [G].

## Background
VastGaussian was introduced in CVPR'24 [S2, unofficial]. It addresses the challenges of scaling 3D Gaussian Splatting to large-scale environments, such as cities, where traditional methods struggle with memory and computational constraints [S4]. The method builds on the divide-and-conquer principle, partitioning the scene into manageable blocks for independent training and merging at render time [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S4] Virtual memory for 3D Gaussian Splatting, https://zenodo.org/records/19690968, fetched 2026-10-26
