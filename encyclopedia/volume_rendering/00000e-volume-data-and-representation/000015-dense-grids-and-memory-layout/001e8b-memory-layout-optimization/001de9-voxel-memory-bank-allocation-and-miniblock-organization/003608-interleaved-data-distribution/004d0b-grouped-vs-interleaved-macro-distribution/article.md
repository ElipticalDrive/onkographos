# Grouped vs Interleaved Macro Distribution
Also called: Grouped Distribution, Interleaved Distribution, Random Distribution

## Overview
Grouped vs Interleaved Macro Distribution classifies how a node's allocated macros are spatially arranged within the global voxel dataset [S1, unofficial]. It addresses spatial locality to optimize cache coherence and manage inter-node communication during ray casting [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Global voxel dataset of size $V$ samples; number of processing nodes $N$; macro size and grouping strategy [S1, unofficial].
*   **Output:** Assignment of macros to nodes, categorized as grouped or interleaved/random [S1, unofficial].

**Data Structures:**
The dataset is partitioned into macros, and each macro is assigned to a specific node [S1, unofficial].

**Steps:**
1. Partition the global voxel array into macros [S1, unofficial].
2. Assign macros to nodes [S1, unofficial].
3. Classify the assignment: if the macros assigned to a single node form a spatially adjacent subset of the global dataset, the distribution is grouped; otherwise, it is interleaved or random [S1, unofficial].

**Formulas:**
*   Let $S_i$ be the set of spatial indices of voxels contained in macros assigned to node $i$ [S1, unofficial].
*   **Grouped Condition:** The union of spatial indices in $S_i$ forms a contiguous sub-volume [S1, unofficial].
*   **Interleaved Condition:** The spatial indices in $S_i$ are scattered or non-contiguous [S1, unofficial].

**Parameters:**
*   Macro size and grouping strategy [S1, unofficial]. Typical values: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Boundary Fetches:** Resampling requires data at neighboring points, creating a redistribution problem at node boundaries [S1, unofficial]. Nodes must request neighboring voxels by fetching the macros containing them [S1, unofficial]. The cost of these fetches is a function of the macro size and grouping [S1, unofficial]. An adjacent (grouped) distribution may avoid these fetches by replicating the neighboring data at each node [S1, unofficial].
*   **Cache Coherence:** Grouped distributions have contiguously addressed data points, which improves cache coherence during ray casting and simplifies addressing during macro redistribution [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# The source does not provide implementation pseudocode for this classification.
```

## Subtopics

### [Data Replication for Adjacent Macros](14149-data-replication-for-adjacent-macros.md)
Replicates boundary voxels to avoid inter-node communication during resampling. Sets itself apart by handling redundancy within grouped partitions rather than defining the partition unit itself. Choose when minimizing network traffic is critical and memory allows for redundancy.

### [Macro Distribution Granularity](16651-macro-distribution-granularity.md)
Defines the size of the macro unit. Sets itself apart by establishing the fundamental unit of data movement rather than the spatial arrangement. Choose finer granularity for load balancing, coarser for reduced addressing overhead.

## Operating characteristics
*   **Time Complexity:** Communication cost depends on the macro size and grouping strategy [S1, unofficial]. Not given in the sources.
*   **Memory Complexity:** Grouped distributions may replicate neighboring data at each node to avoid fetches [S1, unofficial]. Not given in the sources.
*   **Strengths:** Grouped distributions offer better cache coherence during ray casting and simplify addressing during macro redistribution [S1, unofficial].
*   **Limitations:** Interleaved or random distributions do not inherently avoid boundary fetches through replication [S1, unofficial].
*   **When to Use:** Not given in the sources. [S1, unofficial]

## Background
This classification appears in taxonomies of volume rendering algorithms on multicomputers [S1, unofficial].

## Sources
[S1] DTIC ADA236632: Taxonomy and Algorithms for Volume Rendering on Multicomputers, https://archive.org/details/DTIC_ADA236632, fetched 2026-10-26 (unofficial)
