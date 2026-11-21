# Macro Distribution Granularity

Also called: medium granularity, macro-based distribution

## Overview
Macro distribution granularity classifies parallel volume rendering data distribution by defining the minimum redistribution quantum as a fixed-size chunk of data, termed a "macro" [S1, unofficial]. It solves the trade-off between the high addressing overhead of fine-grained arbitrary voxel access and the communication waste of coarse-grained node-level transfers [S1, unofficial]. It sets itself apart from its siblings by establishing the fundamental unit of data movement, whereas [Grouped vs Interleaved Macro Distribution](13385-grouped-vs-interleaved-macro-distribution.md) determines the spatial arrangement of these units, and [Data Replication for Adjacent Macros](14149-data-replication-for-adjacent-macros.md) handles redundancy within them.

## Shared principles
The core mechanism relies on partitioning the dataset into fixed-size chunks called macros [S1, unofficial].

**Definition of a Macro:**
A macro is a data chunk of medium granularity [S1, unofficial]. Typically, macros are spatially adjacent cubes of data addressed by a single coordinate, specifically the lowest $x, y, z$ voxel coordinate in the macro [S1, unofficial].

**Granularity Spectrum:**
Data distribution is classified by the minimum redistribution quantum [S1, unofficial]:
1.  **Coarse Granularity:** A node may only send all or none of its data to another node [S1, unofficial]. This is potentially wasteful if only one voxel is needed from a node, as an entire node's data set must be transmitted [S1, unofficial].
2.  **Fine Granularity:** Allows any number of arbitrarily connected samples to be transmitted [S1, unofficial]. This has very high addressing overhead for arbitrary regions of the 3-D sample grid [S1, unofficial].
3.  **Medium Granularity (Macro):** Data is transmitted in fixed-size chunks, each some fraction of the total data at a node [S1, unofficial]. This is a good compromise between addressing complexity and waste [S1, unofficial].

**Optimization Criteria:**
The optimal size and aspect ratio of a macro is a function of the communication network and the algorithm [S1, unofficial].

**Addressing Efficiency:**
Using Peano addressing of the data set [Laurini ’85] produces a high probability that spatially close points are also close in a linear address space [S1, unofficial]. Peano addressed macros have contiguously addressed data points, which has good implications for cache coherence during ray casting and simplifies addressing during macro redistribution [S1, unofficial].

## Subtopics

### [Grouped vs Interleaved Macro Distribution](13385-grouped-vs-interleaved-macro-distribution.md)
Classifies whether a node's macros form a spatially adjacent subset (grouped) or an interleaved/random subset (interleaved) of the dataset. Choose grouped for better spatial locality and cache coherence; choose interleaved for load balancing in irregular workloads.

### [Data Replication for Adjacent Macros](14149-data-replication-for-adjacent-macros.md)
Handles the redundancy of macro data to minimize communication latency for adjacent samples. Choose when communication latency dominates computation time for neighboring voxels.

## Background
The concept was introduced in the taxonomy of volume rendering algorithms for multicomputers [S1, unofficial]. The work highlights that holding the entire data set at each node via simple replication is not practical [S1, unofficial]. The classification of granularity addresses the inefficiencies of pure coarse or fine distributions in parallel environments [S1, unofficial].

## Sources
[S1] DTIC ADA236632: Taxonomy and Algorithms for Volume Rendering on Multicomputers, https://archive.org/details/DTIC_ADA236632, fetched 2026-10-26 (unofficial)
