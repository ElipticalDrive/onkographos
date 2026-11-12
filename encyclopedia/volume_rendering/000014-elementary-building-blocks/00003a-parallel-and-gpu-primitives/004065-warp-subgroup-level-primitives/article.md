# Warp/Subgroup Level Primitives
## Overview
Warp/Subgroup Level Primitives enable direct communication and aggregation among threads within a single GPU execution unit (warp, wavefront, or subgroup) using registers and fast on-chip logic, bypassing global memory latency [G]. This family solves synchronization and memory bandwidth bottlenecks inherent in traditional shared-memory or atomic-based reductions [G]. Unlike sibling methods that rely on global memory barriers or atomic operations, these primitives enable fast, lockless aggregation of data within the hardware execution unit, as detailed in [Subgroup Operations](3683-subgroup-operations.md), [Warp Shuffling](4949-warp-shuffling.md), and [Warp-level Reduction](10224-warp-level-reduction.md).
## Shared principles
The shared principles for this family are defined in [Parallel and GPU primitives](771-parallel-and-gpu-primitives.md).
## Subtopics

### [Subgroup Operations](3683-subgroup-operations.md)
Subgroup operations are hardware-synchronized collective computations performed across a subgroup (NVIDIA warp, AMD wave) of threads. They solve the problem of high-latency global memory synchronization by allowing threads within the subgroup to exchange data and compute aggregates (sums, ballots) using fast on-chip registers. Unlike atomic operations, which serialize access to global memory, subgroup operations are significantly faster and do not involve global memory synchronization. See [Subgroup Operations](3683-subgroup-operations.md) for implementation details.

### [Warp Shuffling](4949-warp-shuffling.md)
Warp Shuffling (specifically Single Buffer Shuffling, SBShuffle) is a warp-level reduction technique that performs parallel reductions by communicating register values directly among threads within a warp, eliminating the need for shared memory allocation. It addresses the memory bandwidth and allocation overhead of shared memory-based reductions by leveraging hardware shuffle instructions available on Kepler architecture and newer GPUs. Unlike shared memory approaches, it allocates a single `float4` buffer from the register space of each thread to store intermediate results. See [Warp Shuffling](4949-warp-shuffling.md) for implementation details.

### [Warp-level Reduction](10224-warp-level-reduction.md)
Warp-level Reduction is an optimization using GPU warp/subgroup primitives to reduce the number of `atomicMin` calls by combining results from threads within the same warp that target the same pixel. It is chosen when rendering dense point clouds or zoomed-out views where multiple threads in a warp write to the same framebuffer location, as it reduces contention on global atomic counters. Detailed algorithmic steps and pseudocode are not given in the sources.

## Background
Not given in the sources.
## Sources
[S1] Rendering Point Clouds with Compute Shaders and Vertex Order Optimization, https://doi.org/10.1111/cgf.14345, fetched 2026-10-26
