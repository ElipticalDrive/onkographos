# EVERY-2 scheme with greedy optimization

Also called: greedy optimization, greedy selection algorithm

## Overview
The EVERY-2 scheme with greedy optimization is a parallel multiresolution volume rendering algorithm that uses a greedy strategy to select which wavelet tree nodes store pre-reconstructed data, trading storage space for rendering speed while maintaining load balance across processors [S1]. It builds on [Parallel Multiresolution Volume Rendering with Error-Guided Load Balancing](6624-parallel-multiresolution-volume-rendering-with-error-guided-.md) by introducing a selection algorithm that prioritizes nodes for reconstruction based on a benefit function, thereby reducing the runtime reconstruction overhead without significantly increasing communication or storage costs.

## How it works
The method operates on a wavelet tree distributed across $P$ processors using a hierarchical space-filling curve (specifically the Hilbert curve) and error-guided bucketization [S1]. The core innovation is the greedy selection of nodes to store pre-reconstructed data blocks.

### Inputs and Outputs
*   **Input:** A wavelet tree of 3D volume data, distributed across $P$ processors. Each processor holds a set of non-empty wavelet tree nodes. An error tolerance parameter $\epsilon$ (e.g., 1,000) and a storage budget percentage $\alpha$ (e.g., 0%, 20%, 40%) of the total non-empty nodes per processor [S1].
*   **Output:** A subset of wavelet tree nodes marked for pre-reconstruction, and the rendered image.

### Data Structures
The system relies on the following structures and constants, derived from benchmarking [S1]:

1.  **Performance Model Constants:**
    *   $T_1 = 0.0286$ s: Initialization/overhead constant.
    *   $T_4 = 0.3563$ s: Raycasting constant.
    *   $T_5 = 0.0018$ s: Additional overhead constant.
    *   Decompression Speed $S_{dec} = 6.542$ MB/s (including disk I/O) [S1].
2.  **Node Timing:** For each node $i$, $T_2(i)$ and $T_3(i)$ are calculated using the benchmark results. $T_2(i)$ represents the time to decompress/reconstruct node $i$, and $T_3(i)$ represents the time to raycast through node $i$ if pre-reconstructed data is not available (or the cost difference). *Note: The exact formula for $T_2(i)$ and $T_3(i)$ is not explicitly provided in the source text, but they are derived from the benchmark constants and node size/data [S1].*
3.  **Cache Allocation:** Each processor allocates 256 MB temporary disk space and 128 MB main memory for run-time caching of reconstructed data blocks [S1].

### Greedy Selection Algorithm
The algorithm selects nodes to store pre-reconstructed data to minimize total rendering time subject to the storage budget $\alpha$.

**Benefit Function:**
For each node $i$, a benefit score $B(i)$ is calculated. The source specifies a penalty heuristic $p = 0.8$ used in this function [S1]. While the exact mathematical formula for $B(i)$ is not provided in the text, it is defined as a function of the timing constants $T_2(i)$, $T_3(i)$, and the penalty $p$ [S1].

**Steps:**
1.  **Calculate Benefits:** For every non-empty node $i$ in the wavelet tree, calculate the benefit $B(i)$ of storing its pre-reconstructed data. This involves using the benchmark-derived timing constants $T_1, T_4, T_5$ and the node-specific times $T_2(i)$ and $T_3(i)$ [S1].
2.  **Sort Nodes:** Sort all non-empty nodes in descending order of their benefit $B(i)$ [S1].
3.  **Select Nodes:** Iterate through the sorted list and select nodes until the number of selected nodes reaches the storage budget. The budget is defined as $\alpha \times N_{non-empty}$, where $N_{non-empty}$ is the total number of non-empty nodes assigned to that specific processor [S1].
4.  **Store Data:** For each selected node, store the pre-reconstructed data block in the allocated cache space (disk and memory) [S1].
5.  **Render:** During rendering, if a node's pre-reconstructed data is available in the cache, use it directly. Otherwise, decompress and reconstruct the data on the fly [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def greedy_optimization(nodes, processor_id, total_processors, alpha, p=0.8):
    """
    Select nodes for pre-reconstruction using greedy optimization.
    
    Args:
        nodes: List of non-empty wavelet tree nodes assigned to this processor.
        processor_id: ID of the current processor.
        total_processors: Total number of processors P.
        alpha: Percentage of non-empty nodes to select for storage (e.g., 0.2 for 20%).
        p: Penalty heuristic for the benefit function (default 0.8).
        
    Returns:
        selected_nodes: List of nodes to pre-reconstruct and store.
    """
    # 1. Calculate benefit for each node
    # Note: T2(i) and T3(i) are derived from benchmark constants T1, T4, T5, S_dec
    # and node-specific properties (size, etc.) as per [S1].
    # The exact formula for B(i) is not provided, but it uses p=0.8.
    node_benefits = []
    for node in nodes:
        # Calculate timing constants for this node based on benchmarks [S1]
        t2_i = calculate_t2(node) # Decompression/Reconstruction time
        t3_i = calculate_t3(node) # Raycasting time without pre-reconstruction
        
        # Calculate benefit B(i) using penalty p [S1]
        # Formula not explicitly given, but involves T2, T3, and p
        benefit = calculate_benefit(t2_i, t3_i, p)
        node_benefits.append((node, benefit))
    
    # 2. Sort nodes by benefit descending
    node_benefits.sort(key=lambda x: x[1], reverse=True)
    
    # 3. Determine budget
    num_non_empty = len(nodes)
    budget = int(alpha * num_non_empty)
    
    # 4. Select top nodes
    selected_nodes = [node for node, _ in node_benefits[:budget]]
    
    return selected_nodes

def calculate_t2(node):
    """
    Calculate reconstruction time T2(i) for node i.
    Based on decompression speed S_dec = 6.542 MB/s [S1].
    """
    size_mb = node.size_bytes / (1024 * 1024)
    return size_mb / 6.542

def calculate_t3(node):
    """
    Calculate raycasting time T3(i) without pre-reconstruction.
    Derived from benchmark constants T1, T4, T5 [S1].
    """
    # Placeholder for the specific formula involving T1, T4, T5
    # The source states these are calculated from benchmarks but does not give the formula.
    # We assume a linear combination or similar model based on node properties.
    return 0.0 # Not given in the sources.

def calculate_benefit(t2, t3, p):
    """
    Calculate benefit B(i) using penalty heuristic p.
    Formula not given in sources.
    """
    # Placeholder. The source mentions p=0.8 is used in B(i) but does not define B(i).
    return 0.0 # Not given in the sources.
```

### Parameters and Edge Cases
*   **Penalty Heuristic ($p$):** Set to 0.8 [S1].
*   **Storage Budget ($\alpha$):** Set to percentages of non-empty nodes (0%, 20%, 40%) rather than fixed byte sizes for intuitive comparison [S1].
*   **Cache Limits:** 256 MB disk, 128 MB memory per processor [S1]. If the cache is exceeded, data shuffling between memory and disk occurs, increasing time (e.g., 898.303s vs 840.838s for single processor with 128MB vs 1GB cache) [S1].
*   **Load Balancing:** Distribution units are assigned such that no processor receives more units if the total non-empty nodes assigned to it exceeds the average limit ($N_{total\_non\_empty} / P$) [S1].

## Operating characteristics
*   **Time Complexity:** The greedy selection step is $O(N \log N)$ due to sorting, where $N$ is the number of non-empty nodes. Rendering time is reduced by avoiding reconstruction for selected nodes.
*   **Memory Complexity:** Requires $O(N)$ storage for the wavelet tree and cache. The cache size is fixed per processor (128 MB RAM, 256 MB disk) [S1].
*   **Performance:**
    *   On 32 processors with error tolerance 1,000:
        *   0% nodes pre-reconstructed: 53.021s total (31.466s reconstruction, 15.603s raycasting) [S1].
        *   20% nodes pre-reconstructed: 35.430s total (21.567s reconstruction, 12.486s raycasting) [S1].
        *   40% nodes pre-reconstructed: 27.701s total (14.527s reconstruction, 11.413s raycasting) [S1].
    *   Parallel Efficiency: Achieves ~94.86% parallel CPU utilization, with a speedup of 30.36x on 32 processors [S1].
    *   Single Processor Comparison: 40% pre-reconstruction on 1 processor with 1GB cache took 840.838s, compared to 898.303s with 128MB cache [S1].
*   **Strengths:** Effective load balancing, significant reduction in reconstruction time, high scalability.
*   **Limitations:** Requires additional storage for pre-reconstructed data. Performance is sensitive to cache size; insufficient cache leads to disk shuffling overhead.

## Background
The EVERY-2 scheme with greedy optimization was proposed in the paper "A Parallel Multiresolution Volume Rendering Algorithm for Large Data Sets" by C. Wang et al., presented at the International Conference on Parallel Processing (ICPP) in 2005 [S1]. It was supported by NSF ITR grant ACI-0325934, DOE Early Career Principal Investigator Award DE-FG02-03ER25572, and NSF Career Award CCF-034688 [S1]. The work extends previous hierarchical data distribution methods by introducing a greedy optimization strategy to further improve rendering speed through selective pre-reconstruction.

## Sources
[S1] A Parallel Multiresolution Volume Rendering Algorithm for Large Data ..., https://academicweb.nd.edu/~cwang11/papers/pc05-pmvr.pdf, fetched 2026-10-26
