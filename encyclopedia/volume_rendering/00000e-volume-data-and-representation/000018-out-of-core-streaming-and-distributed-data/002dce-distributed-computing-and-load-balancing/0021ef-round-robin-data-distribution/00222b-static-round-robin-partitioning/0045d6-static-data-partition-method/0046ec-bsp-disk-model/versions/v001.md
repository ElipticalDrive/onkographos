# BSP-Disk Model

Also called: BSP-Disk parallel computation model

## Overview
The BSP-Disk model is a distributed memory parallel computation model designed to describe PC clusters where each node possesses its own CPU, local memory, and local disk [S1]. It combines the Bulk Synchronous Parallel (BSP) model with the Parallel Disk Model (PDM) to account for local computation, local disk I/O, and inter-processor communication [S1]. It is primarily used to analyze and optimize isocontour visualization algorithms for massive datasets, where static data partitioning is chosen to avoid the scalability bottlenecks of dynamic load redistribution in low-bandwidth cluster environments [S1].

## Shared principles
The model characterizes a system of $P$ interconnected processors, where each processor $p$ has associated local memory and local disk storage [S1]. Unlike the PDM model, where a single processor has equal access to all parallel disks, the BSP-Disk model associates disks with specific processors as local disks [S1]. A common configuration is $P = D$, where every processor has one associated local disk, typical of PC clusters [S1]. More generally, $P/D$ processors can be assigned to every disk [S1]. Accessing data on a remote disk requires extra communication time [S1].

The total execution time $T$ for an algorithm in this model is defined as the maximum time across all processors $P$, comprising local computation time $T_w$, local disk I/O time $T_{io}$, and communication time $T_c$ [S1]:

$$
T = \max_{P}(T_w + T_{io} + T_c) \leq \max_{P}(T_w) + \max_{P}(T_{io}) + \max_{P}(T_c)
$$

The components are measured as follows:
- $T_w$ and $T_c$ are measured using the BSP model parameters [S1].
- $T_{io}$ is measured as $A \times N_d$, where $N_d$ is the number of parallel I/O operations and $A$ is the time to read or write one disk block on the local disk [S1].

The model uses the following parameters [S1]:
- $N$: Total number of atomic units of the problem.
- $M$: Total number of atomic units that can fit in one processor's main memory.
- $P$: Number of processors.
- $D$: Number of Disks.
- $B$: Number of atomic units that fits in one disk block.
- $A$: Time to read or write one disk block on the local disk.
- $L$: Minimal synchronization time of the BSP model.
- $g$: Gap parameter of the BSP model, characterizing the communication bandwidth.

For large datasets, the design conditions are $N > P \cdot M$ and $M \gg B$ [S1].

## Subtopics

### Static Data Partition Method
A data partitioning strategy chosen for isocontour visualization in the BSP-Disk model to minimize communication overhead. It partitions data among multiple computational nodes at the preprocessing stage and builds an external interval tree as the index structure for blocks on each disk. See [Static Data Partition Method](11605-static-data-partition-method.md).

## Background
The BSP-Disk model was introduced to address the challenges of visualizing massive datasets on commodity off-the-shelf clusters [S1]. It was proposed in the paper "Scalable isosurface visualization of massive datasets on commodity off-the-shelf clusters" [S1]. The model was developed to bridge the gap between theoretical parallel computation models (BSP and PDM) and the practical constraints of PC clusters, specifically the limited bandwidth and high gap factor $g$ of inter-node communication [S1]. This led to the preference for static data partitioning over dynamic load redistribution, as dynamic methods require transmitting data blocks between processors, which is inefficient in low-bandwidth environments [S1].

## Sources
[S1] Scalable isosurface visualization of massive datasets on commodity off-the-shelf clusters., https://doi.org/10.1016/j.jpdc.2008.07.006, fetched 2026-11-08
