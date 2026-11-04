# Real-time KD Tree Construction

Also called: BFS-based KD Tree Construction, GPU-based KD Tree Construction

## Overview
Real-time KD Tree Construction is a technique for building k-d tree spatial partitioning structures on the GPU to enable immediate use in rendering pipelines, specifically addressing the latency of traditional CPU-based construction. It sets itself apart from its parent method, [K-D Tree Hierarchy Construction](2300-k-d-tree-hierarchy-construction.md), by utilizing GPU parallelism and breadth-first search (BFS) to achieve real-time performance, whereas the parent implies a general hierarchical construction often assumed to be offline or CPU-bound. It differs from [GPU K-D Tree Ray Traversal](4575-gpu-k-d-tree-ray-traversal.md) by focusing on the *construction* phase rather than the *traversal* phase.

## How it works
The method constructs k-d tree nodes in real-time using GPU programming via a breadth-first search approach [S1].

**Inputs and Outputs:**
- Input: A set of primitives or voxel data to be partitioned.
- Output: A k-d tree structure stored in GPU memory, ready for traversal.

**Data Structures:**
The specific memory layout is not detailed in the provided sources, but the structure follows the standard k-d tree node definition implied by the parent article: a split plane (axis and position) and child node indices. The tree is built in prefix-order traversal order as per the parent [S1][S2, unofficial].

**Steps:**
1. Initialize the GPU with the input data.
2. Perform a breadth-first search (BFS) to determine the tree structure and split planes [S1].
3. Store the resulting node information in GPU texture memory [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for BFS-based KD Tree Construction on GPU
# Note: Specific implementation details for the BFS kernel are not provided in the sources.
# The following outlines the high-level logic based on the description.

def build_kd_tree_bfs_gpu(data):
    # 1. Allocate GPU memory for tree nodes
    # The tree size must be determined or bounded.
    tree_nodes = allocate_gpu_memory(num_nodes)
    
    # 2. Launch BFS kernel
    # The kernel processes nodes level by level.
    # Each thread or thread block handles one node split.
    launch_bfs_kernel(data, tree_nodes)
    
    # 3. Return the constructed tree
    return tree_nodes

def launch_bfs_kernel(data, tree_nodes):
    # This function represents the GPU kernel logic.
    # It is not fully specified in the sources [S1].
    # It would typically involve:
    # - Determining the best split axis and position for the current node's bounding box.
    # - Partitioning the data into left and right subsets.
    # - Writing the split information to tree_nodes.
    # - Enqueuing child nodes for further processing in subsequent BFS levels.
    pass
```

**Parameters:**
- Not given in the sources.

**Edge Cases and Pitfalls:**
- The original parallel construction scheme by Zellmann et al. was designed for multi-core CPU architectures [S2, unofficial]. Adapting it to the GPU requires careful handling of memory coalescing and synchronization, which is not detailed in the provided notes [S2, S2, unofficial].
- Precision issues in split plane calculation on GPU may arise if not carefully managed, though specific mitigation strategies are not cited [S1].

## Subtopics

### Parallel KD-Tree Construction based on Summed-Volume Tables
This variant, proposed by Zellmann et al., uses summed-volume tables to accelerate the construction of k-d trees for sparse volumes [S2, S2, unofficial]. While the original work [ZSL18] targeted multi-core CPUs, a GPU-based adaptation [ZSL21] allows for real-time construction [S2, S2, unofficial]. It is chosen when dealing with sparse volumetric data where traditional partitioning methods are inefficient. It differs from the general BFS approach by leveraging the spatial coherence and sparsity information encoded in summed-volume tables to determine optimal splits [S2, S2, unofficial].

## Operating characteristics
- **Time Complexity:** Real-time, as the method is designed for immediate construction on the GPU [S1].
- **Memory Complexity:** Depends on the tree size and data structure, but stored in GPU texture memory [S1].
- **Performance:** Specific measured performance metrics are not provided in the sources.
- **Strengths:** Enables real-time spatial partitioning, suitable for dynamic scenes or interactive applications [S1].
- **Limitations:** The method is primarily effective for surface rendering according to one source [S1], though its application to volume rendering is implied by the context. The adaptation from CPU to GPU requires specific optimizations not detailed here [S2, S2, unofficial].

## Background
The technique was first proposed by Zhou et al. [S1]. It was developed to address the need for real-time spatial partitioning in rendering applications, moving away from offline CPU-based construction. The approach was later adapted by Zellmann et al. for sparse volumes using summed-volume tables, with a subsequent GPU-based implementation [S2, S2, unofficial].

## Sources
[S1] Real-time volume rendering for three-dimensional fetal ultrasound using volumetric photon mapping., https://doi.org/10.1186/s42492-024-00177-4, fetched 2026-10-26
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
