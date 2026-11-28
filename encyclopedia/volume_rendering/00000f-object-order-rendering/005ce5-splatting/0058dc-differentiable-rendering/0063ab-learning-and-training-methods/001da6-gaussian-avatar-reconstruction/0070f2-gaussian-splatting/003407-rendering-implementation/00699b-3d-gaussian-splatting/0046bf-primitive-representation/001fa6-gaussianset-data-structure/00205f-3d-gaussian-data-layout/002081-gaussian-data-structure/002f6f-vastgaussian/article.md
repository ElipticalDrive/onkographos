# VastGaussian


## Overview
VastGaussian is a large-scale 3D Gaussian Splatting method that partitions a scene into spatially-disjoint regions to enable scalable training and rendering of massive environments [S2, S3, unofficial]. It solves the memory and computational bottlenecks of global optimization by training independent models per region and merging them at render time using feathered overlap [S2, unofficial]. It distinguishes itself from naive spatial splitting by incorporating decoupled appearance handling for varying lighting conditions and an iterative viewpoint addition strategy based on visibility criteria [S3, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A set of images with known camera poses covering a large-scale scene.
- Output: A set of 3D Gaussian primitives partitioned into spatially-disjoint regions, each with associated parameters (position, covariance, opacity, spherical harmonics coefficients, and appearance embeddings).

**Data Structures:**
The scene is divided into $N$ spatially-disjoint regions (blocks). Each region $i$ contains a subset of Gaussians $\mathcal{G}_i$.
- **Region Partitioning:** Regions are defined based on the distribution of training cameras projected onto the ground plane [S3, unofficial].
- **Gaussian Parameters:** Standard 3DGS parameters: mean $\mu$, covariance $\Sigma$, opacity $\alpha$, spherical harmonics coefficients $c_{lm}$, and an additional optimizable appearance embedding $e_v$ for each view $v$ [S3, unofficial].

**Steps:**
1. **Partitioning:** Divide the scene into spatially-disjoint regions based on camera distribution projected onto the ground [S3, unofficial].
2. **Initialization:** Initialize Gaussians for each region independently.
3. **Iterative Training:**
   - Start with a subset of viewpoints.
   - Train the Gaussians in each region using the current subset of viewpoints [S3, unofficial].
   - Iteratively add more viewpoints into the training set based on visibility criteria to ensure sufficient supervision [S3, unofficial].
4. **Appearance Modeling:** For each view $v$, optimize an appearance embedding $e_v$ to model appearance changes due to varying lighting conditions [S3, unofficial].
5. **Merging:** At render time, merge the Gaussians from all regions. Use feathered overlap at region boundaries to handle discontinuities [S2, unofficial].

**Formulas:**
- **Appearance Embedding:** The color $C$ for a view $v$ is modulated by the appearance embedding $e_v$. The exact modulation function is not specified in the sources [S3, unofficial].
- **Feathered Overlap:** The blending at boundaries is not explicitly defined, but described as "feathered" [S2, unofficial].

**Parameters:**
- Number of regions: Not given in the sources.
- Viewpoint addition criteria: Based on visibility, but specific thresholds are not given [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Discontinuities:** Handled by feathered overlap [S2, unofficial].
- **Under-supervision:** Addressed by visibility-aware viewpoint addition [S3, unofficial].

## Subtopics

### [Progressive Partitioning for Large-Scale 3DGS](8057-progressive-partitioning-for-large-scale-3dgs.md)
Progressive Partitioning is a data distribution strategy for training 3D Gaussian Splatting on large-scale scenes by dividing the environment into spatially-disjoint cells, each containing a subset of training cameras and 3D Gaussians. It solves the memory and time bottlenecks of global optimization by enabling parallel, lower-memory training per cell. It distinguishes itself from naive spatial splitting by using an airspace-aware visibility criterion to ensure sufficient supervision of the volume above the surface, preventing floaters and under-supervision.

### [Large Scene Reconstruction via Chunking](8353-large-scene-reconstruction-via-chunking.md)
Large Scene Reconstruction via Chunking addresses the physical memory limits and scalability challenges inherent in 3D Gaussian Splatting (3DGS) when applied to massive environments. By partitioning the scene into spatially disjoint or distributed regions, it enables the reconstruction of datasets that would otherwise exceed GPU memory capacities. The method splits scenes into chunks based on camera distribution and SfM reconstruction, uniform grids, or dynamic hierarchies, and handles boundaries via skyboxes or pruning. It supports distributed training across multiple GPUs with dynamic load balancing.

### [Page Assignment and Merging](8355-page-assignment-and-merging.md)
Page Assignment and Merging is a spatial partitioning strategy used in VastGaussian to organize millions of 3D Gaussians into fixed-size memory pages for efficient virtual memory management and rendering. It solves the problem of mapping unstructured Gaussian clusters to a structured, mesh-aligned page hierarchy, ensuring that Gaussians are grouped by spatial locality while respecting hardware page size constraints. Unlike standard grid-based voxelization, this method derives its topology from a proxy triangular mesh, allowing it to handle surface-aligned geometry with varying lighting and appearance. It distinguishes itself by combining mesh-based subdivision with greedy merging and explicit page linking to handle overlapping ellipsoids that cross page boundaries.

### [Similarity Guided Partition](14399-similarity-guided-partition.md)
Similarity Guided Partition is a graph-based clustering algorithm that groups 3D Gaussians into spatially-disjoint blocks of bounded size based on spatial proximity and appearance similarity [1, unofficial]. It solves the problem of inefficient joint pruning by ensuring that potentially redundant Gaussians are evaluated together within the same optimization block [1, unofficial]. It sets itself apart from naive spatial partitioning by incorporating a scale-aware affinity metric that accounts for both geometric overlap and appearance consistency [1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Scales with the number of regions and Gaussians per region, reducing peak memory usage compared to global optimization [S2, S3, unofficial].
- **Measured Performance:** Not given in the sources.
- **Strengths:** Scalable to large scenes; handles varying lighting via appearance embeddings; reduces memory bottlenecks [S2, S3, unofficial].
- **Limitations:** Requires careful handling of region boundaries; iterative viewpoint addition may increase training time [S2, S3, unofficial].
- **When to Use:** Large-scale urban or outdoor scenes with varying lighting conditions [S2, S3, unofficial].
- **When Not to Use:** Small-scale scenes where global optimization is feasible [S2, unofficial].

## Background
VastGaussian was introduced in CVPR'24 [S2, unofficial]. It addresses the scalability challenges of 3D Gaussian Splatting for large-scale environments by partitioning the scene into spatially-disjoint regions [S2, unofficial]. The method builds on the divide-and-conquer strategy used in other large-scale reconstruction methods [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
