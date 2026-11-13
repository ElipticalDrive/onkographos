# Data Replication for Adjacent Macros

Also called: Boundary Voxel Replication, Ghost Zone Replication in Distributed Macros

## Overview
Data Replication for Adjacent Macros is a distributed memory optimization strategy for volume rendering that eliminates inter-node communication during the resampling phase by replicating boundary voxels from neighboring nodes [S1, unofficial]. It solves the problem of fetching remote data when a processing node needs to resample a ray at a position near the edge of its local data partition [S1, unofficial]. Unlike non-adjacent distributions, which require fetching macros for every boundary access, this method assumes a spatially adjacent macro distribution where each node holds a contiguous sub-volume, allowing a fixed, small amount of neighbor data to be replicated locally [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: Total number of samples in the dataset ($V$), number of processing nodes ($N$), and the assumption that macros at each node form a cubic shape [S1, unofficial].
- **Outputs**: Local voxel storage layout at each node, including unique voxels and replicated boundary voxels [S1, unofficial].

**Data Structures**
Each node maintains a local 3D grid of voxels. The grid consists of:
1.  **Unique Voxels**: The core partition of the dataset assigned to the node.
2.  **Replicated Voxels**: Copies of voxels from adjacent nodes' boundaries, stored in specific layers surrounding the unique data to facilitate local resampling [S1, unofficial].

**Numbered Steps**
1.  Partition the total dataset of $V$ samples among $N$ nodes such that each node stores $V/N$ unique samples [S1, unofficial].
2.  Assume the local data at each node forms a cube. Calculate the number of samples on one face of this cube as $(V/N)^{2/3}$ [S1, unofficial].
3.  Identify the boundary voxels required to ensure that any arbitrary resampling point in the global domain can be resolved using only local data [S1, unofficial].
4.  Replicate the following neighbor voxels at each node:
    -   3 faces of neighbor voxels [S1, unofficial].
    -   3 edges of neighbor voxels [S1, unofficial].
    -   1 corner of neighbor voxels [S1, unofficial].
    This replication ensures that the 8 nearest samples for any arbitrary position are present at some node [S1, unofficial].
5.  Store the replicated voxels in the local memory structure. Note that replication is not needed for the three outer faces of the entire dataset (the global boundaries) [S1, unofficial].

**Formula**
The total number of voxels stored at each node ($D_{node}$) is given by:

$$ D_{node} = \frac{V}{N} + 3\left(\frac{V}{N}\right)^{2/3} + 3\left(\frac{V}{N}\right)^{1/3} + 1 $$

Where:
-   $V$: Total samples in the data set [S1, unofficial].
-   $N$: Number of processing nodes [S1, unofficial].
-   $\frac{V}{N}$: Unique voxels per node [S1, unofficial].
-   $3\left(\frac{V}{N}\right)^{2/3}$: Replicated voxels from the 3 faces [S1, unofficial].
-   $3\left(\frac{V}{N}\right)^{1/3}$: Replicated voxels from the 3 edges [S1, unofficial].
-   $1$: Replicated voxel from the 1 corner [S1, unofficial].

**Parameters and Typical Values**
-   **Macro Shape**: Cubic [S1, unofficial].
-   **Replication Depth**: 1 voxel layer for faces, edges, and corners [S1, unofficial].

**Edge Cases and Pitfalls**
-   **Global Boundaries**: No replication is needed for the three faces of the dataset that are on the exterior of the global volume [S1, unofficial].
-   **Non-Adjacent Distributions**: If the macro distribution is not adjacent (interleaved or random), this replication scheme fails to eliminate fetches. Replication would be required for each individual macro, resulting in significantly higher total cost [S1, unofficial].
-   **Scalability**: Memory efficiency decreases as $N$ increases because the overhead of replicated boundary voxels grows relative to the shrinking unique voxel count [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_replication_overhead(V, N):
    """
    Calculate the total voxels stored per node using Data Replication for Adjacent Macros.
    
    Args:
        V (int): Total number of samples in the dataset.
        N (int): Number of processing nodes.
        
    Returns:
        int: Total voxels stored at each node (D_node).
    """
    # Step 1: Calculate unique voxels per node
    unique_voxels = V / N
    
    # Step 2: Calculate dimensions of the local cubic partition
    # Side length of the local cube
    side_length = unique_voxels ** (1/3)
    
    # Step 3: Calculate replicated voxels
    # 3 faces: each face has side_length^2 voxels
    face_replication = 3 * (side_length ** 2)
    
    # 3 edges: each edge has side_length voxels (excluding corners already counted in faces? 
    # Note: The formula adds 3*(V/N)^(1/3) directly. This implies edges are counted separately 
    # from faces in the additive model provided in the source.
    edge_replication = 3 * side_length
    
    # 1 corner: 1 voxel
    corner_replication = 1
    
    # Step 4: Sum total voxels
    # Formula: D_node = V/N + 3(V/N)^(2/3) + 3(V/N)^(1/3) + 1
    total_voxels = unique_voxels + face_replication + edge_replication + corner_replication
    
    return total_voxels

def get_replication_stats(V, N):
    """
    Calculate and print replication statistics for a given dataset size and node count.
    """
    unique = V / N
    total = calculate_replication_overhead(V, N)
    replicated = total - unique
    percentage = (replicated / total) * 100
    
    return {
        "unique_voxels_per_node": unique,
        "replicated_voxels_per_node": replicated,
        "total_voxels_per_node": total,
        "replication_percentage": percentage
    }
```

## Subtopics

### Grouped vs Interleaved Macro Distribution
The parent classification. Adjacent macros (grouped) enable this replication strategy, whereas interleaved macros do not [13385-grouped-vs-interleaved-macro-distribution.md].

### Voxel Memory Bank Allocation and Miniblock Organization
A lower-level memory optimization that may work in conjunction with macro distribution strategies to optimize cache locality [13384-voxel-memory-bank-allocation-and-miniblock-organization.md].

## Operating characteristics
-   **Time Complexity**: Resampling is $O(1)$ locally per voxel access because no inter-node communication is required for boundary cases [S1, unofficial].
-   **Memory Complexity**: $O(V/N + V^{2/3}N^{-2/3})$ per node. The replication overhead scales with the surface area of the local partition relative to the volume [S1, unofficial].
-   **Measured Performance**: For a $128^3$ dataset:
    -   8 nodes: 4.76% replication overhead [S1, unofficial].
    -   64 nodes: 9.67% replication overhead [S1, unofficial].
    -   512 nodes: 19.9% replication overhead [S1, unofficial].
    -   4,096 nodes: 42.4% replication overhead [S1, unofficial].
    -   32,768 nodes: 95.3% replication overhead [S1, unofficial].
-   **Strengths**: Eliminates inter-node fetches at data boundaries for adjacent distributions, reducing communication latency [S1, unofficial].
-   **Limitations**: Memory usage efficiency decreases significantly as the number of nodes increases [S1, unofficial]. Not applicable to non-adjacent (interleaved) macro distributions [S1, unofficial].
-   **When to Use**: When using an adjacent macro distribution and minimizing communication latency is critical, and the number of nodes is small enough that replication overhead is acceptable [S1, unofficial].
-   **When Not to Use**: When using interleaved macro distributions or when the number of nodes is very large, leading to excessive memory overhead [S1, unofficial].

## Background
This method was described in the context of volume rendering on multicomputers, specifically addressing redistribution problems at node data boundaries [S1, unofficial]. It is part of a broader taxonomy of algorithms for distributed volume rendering, considering factors like memory usage, CPU task organization, and inter-node communication [S1, unofficial]. The approach builds on the assumption that adjacent macro distributions can avoid fetches by replicating neighboring data [S1, unofficial].

## Sources
[S1] DTIC ADA236632: Taxonomy and Algorithms for Volume Rendering on Multicomputers, https://archive.org/details/DTIC_ADA236632, fetched 2026-10-26 (unofficial)
