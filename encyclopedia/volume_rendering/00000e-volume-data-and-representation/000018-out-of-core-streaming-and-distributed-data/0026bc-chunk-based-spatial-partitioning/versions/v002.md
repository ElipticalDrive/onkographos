# Chunk-based spatial partitioning


## Overview
Chunk-based spatial partitioning decomposes a volume dataset or 3D world into fixed-size spatial subsets (chunks) to enable efficient memory management, parallel processing, and streaming [S1, S2, unofficial]. It differs from [Sectioning](1636-sectioning.md) by using fixed 3D blocks rather than planar slices, and from [VolumeLoader](2045-volumeloader.md) by explicitly managing the lifecycle and spatial locality of these blocks rather than just loading data on demand.

## Shared principles
All methods in this family share the following core mechanisms:
1.  **Decomposition:** The input volume or mesh is partitioned into subsets. This can be spatial (sub-volumes) or topological (mesh cells) [S1, S2, S4, unofficial].
2.  **Distribution:** Subsets are assigned to distinct processing units (MPI ranks, cluster nodes) [S1, S2, S4, unofficial].
3.  **Local Rendering:** Each unit renders its assigned subset independently, typically using ray casting or projection [S1, S2, S4, unofficial].
4.  **Compositing/Assembly:** Results from all units are combined to form the final image. This occurs either via pixel-based accumulation over the network (e.g., SDVR) or image-based assembly by a root process (e.g., MPI 1D/2D) [S1, S2, S4, unofficial].

## Subtopics

### [MPI-based Volume Partitioning](1093-mpi-based-volume-partitioning.md)
MPI-based Volume Partitioning is a data-parallel strategy where a volume dataset is partitioned into contiguous chunks of slices, with each MPI process reading and rendering only its assigned subset. It differs from image-based parallelism (e.g., [Chunk-based World Streaming](2369-chunk-based-world-streaming.md)) by distributing the raw data access and local rendering workload rather than compositing final pixel buffers, and from adaptive methods (e.g., [Uniform Block Partitioning with Acceleration](6098-uniform-block-partitioning-with-acceleration.md)) by relying on fixed spatial slice boundaries rather than workload-balanced adaptive blocks.

### [Chunk-based World Streaming](2369-chunk-based-world-streaming.md)
Chunk-based World Streaming divides a 3D voxel world into fixed-size spatial partitions (chunks) to manage memory, generation, and rendering efficiently by loading only the chunks near the player. It differs from [Chunk-based Training](3168-chunk-based-training.md) by focusing on runtime spatial management rather than data preparation for machine learning, and from [Chunk Trees](4050-chunk-trees.md) by using a flat hash map of active chunks rather than a hierarchical tree structure for spatial queries.

### [Chunk-based Training](3168-chunk-based-training.md)
Chunk-based Training is a distributed optimization strategy for training large-scale neural volume representations, where the input volume is partitioned into spatial chunks to enable parallel processing across multiple GPUs. It differs from single-GPU training by scaling the coordinate space and training units according to the number of chunks, allowing datasets like Hemibrain to be trained on 4–5 GPUs instead of a single device.

### [Chunk Trees](4050-chunk-trees.md)
Chunk Trees are a planet-wide octree-based spatial partitioning method for voxel terrain rendering, designed to replace the legacy "span columns" approach. The method eliminates visible refinement artifacts and memory fragmentation by organizing data into a hierarchical structure of chunk trees, ensuring seamless visual continuity from ground level to orbit. It differs from sibling methods by enforcing a screen-space cut that covers the entire world to prevent checker-hole artifacts, while using motion-aware detail levels to maintain performance during high-speed camera movement.

### [Object-space decomposition with local clustering](4156-object-space-decomposition-with-local-clustering.md)
Object-space decomposition with local clustering partitions an unstructured volume grid into topological subsets (clusters) to enable parallel rendering across distributed processors. It differs from spatial partitioning methods by grouping cells based on mesh connectivity rather than geometric bounding boxes, which minimizes inter-processor communication during ray casting. This method serves as the static preprocessing foundation for the Remapping Phase, which dynamically adjusts these clusters based on viewing parameters.

### [Uniform Block Partitioning with Acceleration](6098-uniform-block-partitioning-with-acceleration.md)
Uniform Block Partitioning with Acceleration is a technique that subdivides a volume dataset into uniform-sized blocks and applies acceleration techniques like early ray termination, empty space skipping, and occlusion clipping. It differs from its siblings by focusing on uniform spatial decomposition combined with ray-casting specific optimizations for interactive display, whereas [Chunk-based World Streaming](2369-chunk-based-world-streaming.md) focuses on memory management and [Chunk Trees](4050-chunk-trees.md) uses hierarchical spatial indexing.

### VOGUE Parallel Operation
#### step
- [S1] Divide dataset into subvolumes distributed across different units. Each unit processes a ray while it traverses its subvolume. On exit, send ray properties to the neighboring unit.
#### parameter
- [S1] Inter-unit communication: Multi-master bus for up to 8 engines (640MB/s bandwidth). Ring-connected cubic network for larger numbers (e.g., 4x4x4 units), requiring ~27MByte/s per link, total ~5.2GByte/s.
#### pitfall
- [S1] Gaps between subvolumes must be removed to avoid visual artifacts. This is done by replicating a certain set of boundary voxels.

## Background
The concept of dividing datasets into subvolumes for parallel processing was implemented in the COLOSSUS architecture, which used a multi-master bus or ring-connected cubic network to distribute ray processing across engines [S1]. The COLOSSUS system implemented the unrestricted Phong illumination model for a single point light source and performed depth-cueing, with a compositing unit (COMET) summing intensities to pass pixel colors to the frame buffer [S1].

## Sources
[S1] Three Architectures for Volume Rendering, https://vcg.seas.harvard.edu/publications/19950801-three-architectures-for-volume-rendering/paper, fetched 2026-10-29
