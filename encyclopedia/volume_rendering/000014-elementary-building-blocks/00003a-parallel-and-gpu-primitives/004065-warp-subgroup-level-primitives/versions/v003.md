# Warp/Subgroup Level Primitives
## Overview
Warp/Subgroup Level Primitives enable direct communication and aggregation among threads within a single GPU execution unit (warp, wavefront, or subgroup) using registers and fast on-chip logic, bypassing global memory latency [G]. This family solves synchronization and memory bandwidth bottlenecks inherent in traditional shared-memory or atomic-based reductions, enabling fast, lockless aggregation as detailed in [Subgroup Operations](3683-subgroup-operations.md), [Warp Shuffling](4949-warp-shuffling.md), and [Warp-level Reduction](10224-warp-level-reduction.md) [G].
## Shared principles
The shared principles for this family are defined in [Parallel and GPU primitives](771-parallel-and-gpu-primitives.md) [G].
## Subtopics

### [Subgroup Operations](3683-subgroup-operations.md)
Subgroup operations are hardware-synchronized collective computations performed across a subgroup of threads. They solve the problem of high-latency global memory synchronization by allowing threads to exchange data and compute aggregates using fast on-chip registers. Unlike atomic operations, which serialize access to global memory, subgroup operations are significantly faster and do not involve global memory synchronization. Choose this method when aggregating data (sums, ballots) within a subgroup without incurring global memory latency.

### [Warp Shuffling](4949-warp-shuffling.md)
Warp Shuffling is a warp-level reduction technique that performs parallel reductions by communicating register values directly among threads within a warp, eliminating the need for shared memory allocation. It addresses the memory bandwidth and allocation overhead of shared memory-based reductions by leveraging hardware shuffle instructions. Unlike shared memory approaches, it allocates a single buffer from the register space of each thread to store intermediate results. Choose this method for efficient parallel reductions (e.g., color blending) where memory bandwidth is a bottleneck.

### [Warp-level Reduction](10224-warp-level-reduction.md)
Warp-level Reduction is an optimization using GPU warp/subgroup primitives to reduce the number of `atomicMin` calls by combining results from threads within the same warp that target the same pixel. It minimizes global atomic contention by aggregating per-warp before a global write. Choose this method for dense point clouds or zoomed-out views where multiple threads write to the same framebuffer location.

## Background
Not given in the sources.
## Sources
[S1] Rendering Point Clouds with Compute Shaders and Vertex Order Optimization, https://doi.org/10.1111/cgf.14345, fetched 2026-10-26
