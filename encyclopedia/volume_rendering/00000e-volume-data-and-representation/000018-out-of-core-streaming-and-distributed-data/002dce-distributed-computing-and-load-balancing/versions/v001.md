# Distributed computing and load balancing
Also called: Parallel volume rendering, Distributed volume rendering

## Overview
This family encompasses techniques for rendering volumetric datasets that exceed the memory or processing capacity of a single machine by distributing the workload across multiple nodes or processes [S1][S2, S4, unofficial]. It solves the scalability bottleneck inherent in single-node rendering by partitioning data or computation [S1][S2, unofficial]. What sets this family apart from [Out-of-core, streaming and distributed data](8-out-of-core-streaming-and-distributed-data.md) is the explicit focus on simultaneous multi-node processing and load balancing, whereas out-of-core methods primarily address single-node memory limits via disk swapping [G].

## Shared principles
All methods in this family share the following core mechanisms:
1.  **Decomposition:** The input volume or mesh is partitioned into subsets. This can be spatial (sub-volumes) or topological (mesh cells) [S1][S2, S4, unofficial].
2.  **Distribution:** Subsets are assigned to distinct processing units (MPI ranks, cluster nodes) [S1][S2, S4, unofficial].
3.  **Local Rendering:** Each unit renders its assigned subset independently, typically using ray casting or projection [S1][S2, S4, unofficial].
4.  **Compositing/Assembly:** Results from all units are combined to form the final image. This occurs either via pixel-based accumulation over the network (e.g., SDVR) or image-based assembly by a root process (e.g., MPI 1D/2D) [S1][S2, S4, unofficial].

## Subtopics

### [Scalable Distributed Volume Rendering (SDVR)](863-scalable-distributed-volume-rendering-sdvr.md)
Targets unstructured data using out-of-core k-d decomposition and OpenGL/MPI. It accumulates subimages over the interconnect. Choose when rendering large unstructured grids on a cluster where hardware acceleration is optional but beneficial.

### [MPI Volume Rendering (1D/2D Decomposition)](2249-mpi-volume-rendering-1d-2d-decomposition.md)
Uses static spatial decomposition of regular 3D scalar grids along X or X/Y axes. Processes send sub-images to a root for assembly. Choose for regular structured data where simple spatial partitioning suffices.

### [Round-Robin Data Distribution](2667-round-robin-data-distribution.md)
Assigns mesh cells cyclically to processors to average load based on cell count rather than spatial locality. Choose when mesh cell complexity varies significantly and spatial coherence is less critical than load balancing.

## Background
Distributed volume rendering emerged in the late 1990s and early 2000s as a response to the growing size of scientific datasets (e.g., CFD, medical imaging) exceeding single-processor capabilities [S1][S2, S4, unofficial]. Early work focused on MPI-based decomposition of structured grids [S1, S2, S4, unofficial], while later approaches addressed unstructured meshes using graph partitioning or round-robin strategies to handle irregular computational costs [S1, unofficial][S2]. The field has since integrated with hardware-assisted streaming and MapReduce paradigms [G].

## Sources
[S1] Scalable Distributed Volume Rendering (SDVR) notes
[S2] SDVR and Round-Robin notes (unofficial)
[S4] MPI Volume Rendering notes (unofficial)
