# Grouped vs Interleaved Macro Distribution
Also called: Grouped Distribution, Interleaved Distribution, Random Distribution
## Overview
This method classifies how macro-sized blocks of voxel data are assigned to processing nodes in a distributed memory system. It determines whether a node holds a spatially contiguous subset of the dataset (grouped) or a scattered subset (interleaved/random) [S1, unofficial]. The primary distinction lies in the trade-off between cache coherence and communication overhead during ray casting resampling [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: Total number of nodes $N$, total number of samples in the dataset $V$, macro size $M$ (samples per macro).
- Output: Assignment map defining which node holds which macros.

**Data Structures:**
- Macro: A block of $M$ contiguous voxels.
- Node: A processing unit with local memory.

**Steps:**
1. **Partitioning:** Divide the voxel array into macros of size $M$. Total macros $K = \lceil V/M \rceil$.
2. **Assignment:**
   - **Grouped:** Assign macros to nodes such that each node receives a spatially adjacent subset of the original voxel array. Node $i$ holds macros corresponding to voxel indices $[i \cdot \frac{V}{N}, (i+1) \cdot \frac{V}{N})$.
   - **Interleaved/Random:** Assign macros to nodes in a round-robin or random fashion. Node $i$ holds macros with indices $k$ where $k \mod N = i$.
3. **Boundary Handling:**
   - In **Grouped** distribution, nodes have clear spatial boundaries. During ray casting, resampling requires neighboring voxels. If a ray crosses a node boundary, the node must fetch macros from adjacent nodes [S1, unofficial].
   - In **Interleaved** distribution, neighboring voxels are likely on different nodes. Every resampling operation near a ray's path may require inter-node communication [S1, unofficial].

**Formulas:**
- Communication Cost: The cost of fetches is a function of macro size $M$ and grouping [S1, unofficial].
- Grouped distribution simplifies addressing during macro redistribution [S1, unofficial].

**Parameters:**
- Macro size $M$: Determines granularity. Larger $M$ reduces communication frequency but increases granularity of spatial locality.
- $N$: Number of nodes.

**Edge Cases:**
- **Ray Casting Resampling:** Requires data at neighboring points. This creates a redistribution problem at node data boundaries [S1, unofficial].
- **Cache Coherence:** Grouped macros have contiguously addressed data points, offering good implications for cache coherence during ray casting [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def assign_macros(grouping_type, N, V, M):
    """
    Assign macros to nodes.
    grouping_type: 'grouped' or 'interleaved'
    N: number of nodes
    V: total samples
    M: macro size
    """
    num_macros = (V + M - 1) // M
    assignment = {} # node_id -> list of macro_indices

    for node_id in range(N):
        assignment[node_id] = []

    for macro_idx in range(num_macros):
        # Determine node based on grouping
        if grouping_type == 'grouped':
            # Spatially adjacent: map macro index to node based on volume fraction
            # Each node gets roughly equal volume
            node_id = (macro_idx * N) // num_macros
        elif grouping_type == 'interleaved':
            # Round-robin
            node_id = macro_idx % N
        else:
            # Random: not deterministic, skip for deterministic assignment
            continue
        
        assignment[node_id].append(macro_idx)
    
    return assignment
```
## Subtopics

### [Data Replication for Adjacent Macros](14149-data-replication-for-adjacent-macros.md)
A strategy for grouped distribution where nodes replicate boundary voxels to avoid inter-node fetches.

## Operating characteristics
- **Grouped Distribution:**
  - **Strengths:** Good cache coherence due to contiguous data addressing [S1, unofficial]. Simplifies addressing during macro redistribution [S1, unofficial].
  - **Limitations:** Requires handling of boundary fetches when rays cross node boundaries. Cost of fetches depends on macro size [S1, unofficial].
- **Interleaved/Random Distribution:**
  - **Strengths:** Load balancing may be easier.
  - **Limitations:** Higher communication overhead for resampling as neighbors are scattered. Poor cache coherence for contiguous ray paths.
- **Scaling:** Communication cost is a function of macro size and grouping [S1, unofficial].
## Background
Originated in the context of volume rendering on multicomputers, specifically addressing data distribution strategies for shared-memory MIMD architectures [S1, unofficial].
## Sources
[S1] DTIC ADA236632: Taxonomy and Algorithms for Volume Rendering on Multicomputers, https://archive.org/details/DTIC_ADA236632, fetched 2026-10-26 (unofficial)
