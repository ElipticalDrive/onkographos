# Distributed computing and load balancing

## Overview
This family encompasses techniques for rendering volumetric datasets that exceed the memory or processing capacity of a single machine by distributing the workload across multiple nodes or processes. It solves the scalability bottleneck inherent in single-node rendering by partitioning data or computation. What sets this family apart from [Out-of-core, streaming and distributed data](8-out-of-core-streaming-and-distributed-data.md) is the explicit focus on simultaneous multi-node processing and load balancing, whereas out-of-core methods primarily address single-node memory limits via disk swapping.

## Shared principles
This family inherits the core mechanisms of decomposition, distribution, local rendering, and compositing defined in [Out-of-core, streaming and distributed data](8-out-of-core-streaming-and-distributed-data.md).

## Subtopics

### [Scalable Distributed Volume Rendering (SDVR)](863-scalable-distributed-volume-rendering-sdvr.md)
Targets unstructured data using out-of-core k-d decomposition and OpenGL/MPI. It accumulates subimages over the interconnect. Choose when rendering large unstructured grids on a cluster where hardware acceleration is optional but beneficial.

### [MPI Volume Rendering (1D/2D Decomposition)](2249-mpi-volume-rendering-1d-2d-decomposition.md)
Uses static spatial decomposition of regular 3D scalar grids along X or X/Y axes. Processes send sub-images to a root for assembly. Choose for regular structured data where simple spatial partitioning suffices.

### [Round-Robin Data Distribution](2667-round-robin-data-distribution.md)
Assigns mesh cells cyclically to processors to average load based on cell count rather than spatial locality. Choose when mesh cell complexity varies significantly and spatial coherence is less critical than load balancing.

### [Cell Count Algorithm for Adaptive-Resolution Load Balancing](6937-cell-count-algorithm-for-adaptive-resolution-load-balancing.md)
Distributes blocks of varying resolutions among processors to balance workload. Choose when adaptive-resolution data requires equal cell counts per processor while maintaining consistent resolution within assigned blocks.

### Distributed Data Loader Creation
Evaluates dataset strings, retrieves world size and rank, and attempts to create a dataset-specific sampler. If that fails, it falls back to a PyTorch distributed sampler, random sampler, or sequential sampler. It then creates and returns a `torch.utils.data.DataLoader` with the selected sampler, batch size, number of workers, pin memory, and drop last parameters [S1, unofficial].

## Background
Not given in the sources.

## Sources
[S1] NVlabs/InstantSplat, https://github.com/NVlabs/InstantSplat, fetched 2026-10-27 (unofficial)
