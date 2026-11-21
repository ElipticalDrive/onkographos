# Distributed computing and load balancing

Also called: Parallel volume rendering distribution, Cluster-based volume rendering, Load balancing strategies

## Overview
This family addresses the scalability limits of single-node volume rendering by distributing both volumetric data and rendering computation across multiple processors or nodes in a cluster. It solves the problem of rendering datasets that exceed the memory capacity or computational throughput of a single machine by partitioning the volume space or data cells among processors [S1, S2, S4, unofficial]. What sets this family apart from its siblings is the specific focus on **load balancing**—ensuring that all processors finish their assigned work simultaneously to minimize idle time—rather than just parallel execution. Unlike [Hardware-Assisted Volume Streaming](2119-hardware-assisted-volume-streaming.md) which optimizes for bandwidth on a single machine, or [Chunk-based spatial partitioning](3862-chunk-based-spatial-partitioning.md) which defines the data structure, these methods define how work is assigned to processors to handle non-uniform data density and view-dependent occlusion [S1, S2, unofficial].

## Shared principles
All methods in this family share the following core mechanisms and constraints:

1.  **Decomposition Strategy**: The volume or dataset is partitioned into subdomains, bricks, or cells. This partitioning can be:
    *   **Spatial**: Dividing the 3D space into rectangular blocks (e.g., 1D, 2D, or 3D grids) [S2, S4, unofficial].
    *   **Data-Driven**: Dividing based on cell counts or resolution levels (e.g., adaptive grids) [S1, unofficial].
    *   **Topology-Aware**: Dividing unstructured meshes based on connectivity [S2, unofficial].

2.  **Work Assignment**: Each processor is assigned a subset of the partitioned data. The goal is to minimize the maximum workload across all processors:
    $$ T_{max} = \max_{p \in Processors} (T_{compute, p} + T_{comm, p}) $$
    Where $T_{compute}$ is the rendering time and $T_{comm}$ is the communication/compositing time [S1, unofficial].

3.  **Compositing Order**: For correct alpha compositing, the relative depth order of rendered partitions must be preserved. If partitions are spatial, the compositing order is often determined by the view direction and the spatial position of the partitions [S1, S2, unofficial]. In distributed systems, partial images are often sent to a root node or composed in a tree-like structure [S2, unofficial].

4.  **Communication Overhead**: Methods must account for the cost of transferring data to processors and transferring partial images for compositing. Load balancing algorithms often trade off spatial locality (which reduces communication) against computational balance (which reduces idle time) [S2, S4, unofficial].

## Subtopics

### [Scalable Distributed Volume Rendering (SDVR)](863-scalable-distributed-volume-rendering-sdvr.md)
SDVR targets large unstructured datasets on PC clusters using OpenGL and MPI. It uses an out-of-core modified k-d decomposition to create load-balanced "brick-shaped chunks" during preprocessing. It is chosen when scalability is prioritized over peak performance, as it can run with or without graphics hardware. Unlike fixed-grid methods, it adapts to the data structure via k-d decomposition.

### [MPI Volume Rendering (1D/2D Decomposition)](2249-mpi-volume-rendering-1d-2d-decomposition.md)
This method partitions a 3D scalar volume into 1D (row-based) or 2D (block-based) subdomains along the X and/or Y axes, keeping the full Z-depth for each process. It is chosen for regular structured data where ray casting along the Z-axis allows local compositing without per-ray communication. It differs from 3D decomposition by eliminating inter-process communication during ray marching, but may suffer from load imbalance if density varies significantly along the partitioned axes.

### [Round-Robin Data Distribution](2667-round-robin-data-distribution.md)
A static load balancing strategy for unstructured grids where cells are assigned to processors cyclically ($p_i = i \mod P$). It is chosen to average out computational costs caused by varying cell sizes and opacities. It sets itself apart by prioritizing computational balance over spatial locality, which destroys spatial relationships and requires additional structures (like spatial partitioning trees) for efficient compositing.

### [Cell Count Algorithm for Adaptive-Resolution Load Balancing](6937-cell-count-algorithm-for-adaptive-resolution-load-balancing.md)
This algorithm balances load for adaptive-resolution datasets by ensuring each processor handles approximately the same total number of voxels (cells). It is chosen when data resolution varies significantly across the volume. It sets itself apart by decoupling block assignment from spatial contiguity, using a greedy sorting approach to assign blocks to processors until their cell count quota is met.

### [Dynamic load balancing using kd-trees](9365-dynamic-load-balancing-using-kd-trees.md)
This method uses a hierarchical kd-tree to dynamically redistribute rendering work during rendering. It is chosen when load imbalance is caused by non-uniform data density or view-dependent occlusion that static methods cannot handle. It sets itself apart by allowing runtime adjustment of partitions based on load statistics stored in inner tree nodes, managing both spatial partitioning and compositing order.

### [Load Balanced Network Distribution (LBND)](9702-load-balanced-network-distribution-lbnd.md)
LBND is an optimization framework that partitions data to minimize the maximum of render, compose, and network times. It is chosen when network bandwidth is a bottleneck or when data sparsity is non-uniform. It sets itself apart by explicitly guiding partition boundaries based on empty space distribution and render costs while maintaining strict view-dependent precedence constraints.

#### Distributed Data Loader Creation
This variant focuses on the software infrastructure for loading data in a distributed manner. It evaluates the dataset string, attempts to create a dataset-specific sampler, and falls back to standard distributed or random/sequential samplers if unavailable. It is used in deep learning-based volume rendering pipelines to ensure efficient data distribution across workers [S1, unofficial].

#### MapReduce Rendering
This variant utilizes the MapReduce API to handle communication and computation in parallel. It employs an asynchronous streaming interface that allows network communication, CPU/GPU data transfers, disk access, and GPU kernel execution to happen concurrently. It is chosen for systems where overlapping I/O and computation is critical for performance [S2, unofficial].

## Background
Distributed volume rendering emerged as a solution to the "memory wall" and "compute wall" encountered when visualizing teravoxel-scale datasets from scientific simulations and medical imaging. Early approaches relied on static spatial decomposition (grids) [S2, unofficial]. As datasets became larger and more heterogeneous, load imbalance became a critical issue, leading to the development of dynamic load balancing techniques using kd-trees and cell-count algorithms [S1, unofficial]. The use of MPI and cluster computing became standard for high-performance volume rendering [S1, S2, unofficial]. More recently, MapReduce paradigms have been adapted for volume rendering to handle massive parallelism and asynchronous I/O [S2, unofficial].

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
[S2] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
