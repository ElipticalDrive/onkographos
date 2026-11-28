# Load Balanced Network Distribution (LBND)


## Overview
Load Balanced Network Distribution (LBND) is an optimization problem in which volumetric data is partitioned across a distributed cluster to minimize end-to-end rendering time [S1, unofficial]. This time is determined by the slowest render-node, image composition time, and network communication time [S1, unofficial]. LBND partitions data to achieve good load balancing for parallel ray casting while strictly preserving the view-dependent precedence order required for correct alpha compositing [S1, unofficial]. It overcomes the granularity limitations of traditional static space-subdivision methods by using cost models rather than fixed spatial grids [S1, unofficial].

## Shared principles
This method inherits the decomposition, work assignment, compositing order, and communication overhead constraints defined in [Distributed computing and load balancing](5624-distributed-computing-and-load-balancing.md).

## Subtopics

### Brick Grouping
A dynamic programming solution that starts with a directed acyclic graph of segmented and cropped bricks [S1, unofficial]. It groups these bricks to minimize the end-to-end rendering cost while respecting image composition priority constraints [S1, unofficial]. Choose this method when the dataset contains large variations in sparsity and uniform empty space distribution cannot be assumed [S1, unofficial].

### Second DP Solution
The source mentions a second dynamic programming solution but does not provide specific details on its algorithmic structure or name [S1, unofficial].

## Background
The LBND problem was formulated to address the inefficiencies of static partitioning in distributed volume rendering [S1, unofficial]. Traditional space-subdivision methods suffer from load balancing issues due to the granularity required to force consecutive depth priorities; if empty space is not uniform, some render nodes are assigned large amounts of empty space, leading to disproportionate computation loads on other nodes [S1, unofficial]. The problem is defined as an optimization task where the input consists of a volumetric data set, along with render and network cost information [S1, unofficial]. The precedence order is defined as the relative distance of each voxel with respect to a given view direction [S1, unofficial]. The goal is to minimize end-to-end render time within resource and priority order constraints [S1, unofficial]. Dynamic programming (DP) was developed as a cost optimization technique to solve this problem [S1, unofficial]. The approach minimizes a cost function that reflects the end-to-end rendering cost on a cluster without violating image composition priority constraints [S1, unofficial]. The associative nature of the alpha compositing equation allows images rendered on a node to be composited into a single image prior to blending with images from other nodes, thereby minimizing the bandwidth required for transporting images between nodes [S1, unofficial].

## Sources

[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
