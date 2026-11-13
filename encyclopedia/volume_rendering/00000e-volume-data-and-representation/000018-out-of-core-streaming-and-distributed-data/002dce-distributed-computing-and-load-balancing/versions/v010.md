# Distributed computing and load balancing

Also called: Parallel volume rendering distribution, Cluster-based volume rendering, Load balancing strategies

## Overview
Distributed computing and load balancing in volume rendering addresses the problem of rendering volumetric datasets that exceed the memory or computational capacity of a single machine by partitioning data and computation across a cluster of processors [G]. It solves the scalability bottleneck inherent in single-node rendering, enabling the visualization of massive unstructured grids, adaptive-resolution datasets, and large scalar fields [G]. Unlike sibling methods that focus on data streaming or hardware acceleration, this family focuses on the algorithmic partitioning of work and data to minimize the maximum execution time ($T_{max}$) while preserving the strict front-to-back compositing order required for correct alpha blending [G]. It sets itself apart from [Out-of-core, streaming and distributed data](8-out-of-core-streaming-and-distributed-data.md) by explicitly managing the trade-off between computational balance and communication overhead during the distributed compositing phase [G].

## Shared principles
All methods in this family share the following core mechanisms and constraints:

1.  **Decomposition Strategy**: The volume or dataset is partitioned into subdomains, bricks, or cells [G]. This partitioning can be:
    *   **Spatial**: Dividing the 3D space into rectangular blocks (e.g., 1D, 2D, or 3D grids) [G].
    *   **Data-Driven**: Dividing based on cell counts or resolution levels (e.g., adaptive grids) [G].
    *   **Topology-Aware**: Dividing unstructured meshes based on connectivity [G].

2.  **Work Assignment**: Each processor is assigned a subset of the partitioned data [G]. The goal is to minimize the maximum workload across all processors:
    $$ T_{max} = \max_{p \in Processors} (T_{compute, p} + T_{comm, p}) $$
    Where $T_{compute}$ is the rendering time and $T_{comm}$ is the communication/compositing time [G].

3.  **Compositing Order**: For correct alpha compositing, the relative depth order of rendered partitions must be preserved [G]. If partitions are spatial, the compositing order is often determined by the view direction and the spatial position of the partitions [G]. In distributed systems, partial images are often sent to a client or composed in a tree-like structure [S2, unofficial].

4.  **Communication Overhead**: Methods must account for the cost of transferring data to processors and transferring partial images for compositing [G]. Load balancing algorithms often trade off spatial locality (which reduces communication) against computational balance (which reduces idle time) [G].

## Subtopics

### [Scalable Distributed Volume Rendering (SDVR)](863-scalable-distributed-volume-rendering-sdvr.md)
A distributed rendering system for unstructured data targeting large PC clusters, using OpenGL and MPI. It partitions data into load-balanced brick-shaped chunks via modified k-d decomposition to ensure computational balance while minimizing communication overhead. Choose when rendering massive unstructured datasets that exceed single-machine memory.

### [MPI Volume Rendering (1D/2D Decomposition)](2249-mpi-volume-rendering-1d-2d-decomposition.md)
A distributed-memory parallel technique that partitions 3D scalar datasets into 1D or 2D spatial subdomains, keeping the full depth (Z-axis) intact for each process. Choose when direct compositing performance is prioritized over abstract partitioning flexibility.

### [Round-Robin Data Distribution](2667-round-robin-data-distribution.md)
A static load balancing strategy that assigns data cells to processors cyclically ($p_i = i \mod P$) to average out computational requirements. Choose when statistical averaging is sufficient and spatial locality is secondary to computational balance.

### [Cell Count Algorithm for Adaptive-Resolution Load Balancing](6937-cell-count-algorithm-for-adaptive-resolution-load-balancing.md)
A strategy for adaptive-resolution datasets that assigns blocks to processors based on cell counts ($C_{optimal} = \lceil C_{total} / p \rceil$) to prevent idle time caused by resolution variance. Choose when block sizes vary significantly and computational balance must be decoupled from spatial contiguity.

### [Dynamic load balancing using kd-trees](9365-dynamic-load-balancing-using-kd-trees.md)
Uses a hierarchical kd-tree to track workload statistics and enable dynamic redistribution of tasks, maintaining explicit spatial hierarchy for compositing order determination. Choose when non-uniform density or occlusion requires runtime adjustments.

### [Load Balanced Network Distribution (LBND)](9702-load-balanced-network-distribution-lbnd.md)
An optimization framework that dynamically groups data bricks based on render and network cost models to ensure the slowest node determines total time without violating compositing priority constraints. Choose when network cost and view-dependent precedence are critical bottlenecks.

### Distributed Data Loader Creation
This variant provides a programmatic interface for creating distributed data loaders in deep learning contexts, often used for training volume rendering models [G]. It evaluates the dataset type, attempts to create a dataset-specific sampler, and falls back to standard PyTorch distributed samplers (e.g., `DistributedSampler`, `RandomSampler`) if a custom sampler is not available [S1, unofficial].
*   **Step**:
1.  Evaluate dataset string if dataset is a string: if `isinstance(dataset, str)`: `dataset = eval(dataset)` [S1, unofficial].
2.  Get world size and rank from distributed environment: `world_size = get_world_size(); rank = get_rank()` [S1, unofficial].
3.  Attempt to create a dataset-specific sampler: `try: sampler = dataset.make_sampler(batch_size, shuffle=shuffle, world_size=world_size, rank=rank, drop_last=drop_last)` [S1, unofficial].
4.  If dataset-specific sampler fails, fall back to torch distributed sampler if distributed is initialized, else random or sequential sampler: `except (AttributeError, NotImplementedError): if torch.distributed.is_initialized(): sampler = torch.utils.data.DistributedSampler(...) elif shuffle: sampler = torch.utils.data.RandomSampler(...) else: sampler = torch.utils.data.SequentialSampler(...)` [S1, unofficial].
5.  Create and return `torch.utils.data.DataLoader` with the selected sampler, `batch_size`, `num_workers`, `pin_memory`, and `drop_last` [S1, unofficial].

### MapReduce Rendering
This variant utilizes the MapReduce API to handle communication and computation in parallel for volume rendering tasks [G]. It employs an asynchronous streaming interface that allows network communication, CPU/GPU data transfers, disk access, and GPU kernel execution to happen concurrently, facilitating efficient processing on large-scale GPU clusters [S2, unofficial].

### Region-of-Interest Based Distribution
A workload distribution method where focus regions (defined by 2D/3D shapes like magic lenses or fixed 3D segmentations) are raycasted with best quality on the fast GPU, while context is established by image warping from frames produced on the slow GPU [S3]. This approach introduces the disadvantage of not being able to predict frame rates during a user session, as the user's input influences the sampling behavior [S3].

### Load Balancing Strategy for Multicomputer Volume Rendering
Uses dynamic tile assignments to distribute work across nodes, implements preferred tile-node pairing to optimize data locality, and constrains view positions to maintain effective load balancing [S4, unofficial].

### Virtual Node Mapping for Large Dataset Rendering
Rendering a dataset larger than individual node memory by mapping multiple virtual nodes onto physical nodes, exploiting memory hierarchy and load balancing without data replication [S5]. Maps eight virtual nodes onto four physical Power Challenge nodes to handle the 7.1 GB dataset, constrained by a 2 GB per process OS memory limit [S5]. Uses no replication and a low load balancing factor (.65) to minimize communication overhead while distributing the large dataset across 64 physical processors [S5]. Applies dataset-specific optimizations: opacity clipping and discarding of completely empty volume blocks to improve frame rate [S5]. Rendering 7.1 GB dataset on 64 processors (8 virtual nodes on 4 physical nodes) at 800x600: 3.4s/frame baseline, 2.1s/frame with optimizations [S5].

## Background
The need for distributed volume rendering arose with the growth of scientific datasets exceeding single-machine memory limits [G]. Early approaches focused on static spatial decomposition using MPI [S2, S4, unofficial]. Scalable Distributed Volume Rendering (SDVR) was developed to handle unstructured grids on PC clusters, addressing the limitations of structured grid methods [G]. The Cell Count Algorithm and Round-Robin strategies emerged to handle specific imbalances in unstructured and adaptive-resolution data [G]. Dynamic load balancing using kd-trees was introduced to handle non-uniform density and occlusion, allowing for runtime adjustments [G]. MapReduce-based approaches were adapted from big data processing to leverage asynchronous GPU execution on clusters [S2, unofficial].

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
[S3] Multi-Frame Rate Volume Rendering, https://doi.org/10.2312/egpgv.egpgv10.019-026, fetched 2026-10-29
[S4] DTIC ADA236632: Taxonomy and Algorithms for Volume Rendering on Multicomputers, https://archive.org/details/DTIC_ADA236632, fetched 2026-10-26 (unofficial)
[S5] Exploiting deep parallel memory hierarchies for ray casting volume rendering, https://doi.org/10.1145/266638.266645, fetched 2026-10-26
