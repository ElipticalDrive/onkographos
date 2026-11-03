# Sparse Voxel Directed Acyclic Graph (SVDAG)
Also called: Sparse Voxel DAG

## Overview
A Sparse Voxel Directed Acyclic Graph (SVDAG) is a hierarchical data structure for representing 3D voxelized scenes that reduces memory footprint by identifying and merging identical subtrees, allowing multiple parent nodes to share the same child node instance [S5]. Unlike a Sparse Voxel Octree (SVO) which only encodes empty regions efficiently, the SVDAG additionally encodes identical regions of space by sharing pointers to identical subtrees [S2]. This allows for lossless compression without decompression overhead during traversal, making it suitable for real-time rendering and interactive editing [S3].

## How it works
**Inputs and Outputs:**
*   **Input:** A binary voxel grid $R$ of size $N^3$ voxels, where $N \ge 2$ and $N = 2^m$ [S1]. The grid contains active (filled) and passive (empty) voxels [S1].
*   **Output:** A root node of a Directed Acyclic Graph (DAG) representing the voxel grid, where identical subvolumes are merged into shared nodes [S2].

**Data Structures:**
The SVDAG consists of two node types: Internal Nodes (INODE) and Leaf Nodes (LNODE) [S1].

1.  **Internal Node (INODE):**
    *   **Child Node Mask (CHNM):** An 8-bit field where each bit is a Header Tag (HT) corresponding to one of the 8 potential children (octants) [S1].
        *   If $HT = 0$: The subDAG is homogenously filled with passive voxels (pruned); no pointer is needed [S1].
        *   If $HT = 1$: At least one voxel is active; a child node pointer is present [S1].
    *   **Reserved Bits (p):** Concatenated after CHNM for alignment [S1].
    *   **Pointers (PTS):** An array of 1 to 8 pointers (PT), each $r$ bits long, pointing to child nodes [S1]. The order of pointers corresponds to the linearization order (e.g., Morton order) [S1].
    *   **Layout:** `CHNM` (8 bits) + `p` (reserved) + `PTS` (variable). Standard alignment uses $p=24$ and $r=32$, ensuring 32-bit alignment [S1].

2.  **Leaf Node (LNODE):**
    *   Represents a homogeneous subvolume (all passive or all active) or the base resolution [S1].

**Algorithm (Bottom-Up Reduction):**
The construction transforms an SVO into a DAG by searching for common subtrees and referencing only unique instances, proceeding from leaves upward [S2].

1.  **Root Construction:** If $N=2$, the root is a leaf node. If $N>2$, the root is an internal node [S1].
2.  **Recursive Build:** For a given grid section:
    a.  Check if the section is homogeneous (all voxels identical) or if the base resolution is reached [S5].
    b.  If homogeneous/passive: Create a Leaf Node (LNODE). If active, it represents the filled volume; if passive, it is pruned [S5].
    c.  If not homogeneous: Create an Internal Node (INODE) [S5].
    d.  For each of the 8 octants:
        i.  Extract the sub-grid.
        ii. Recursively build the child node.
        iii. If the child is a leaf representing passive voxels, set the corresponding HT in CHNM to 0 and do not add a pointer to PTS [S1].
        iv. If the child is active, set HT to 1 and add the pointer to PTS [S1].
3.  **Common Subtree Merging (CSM):**
    a.  Generate a unique key for the current subtree configuration (e.g., based on child pointers and structure) [S5].
    b.  Check a cache of existing nodes. If a node with the same key exists, return the existing node's pointer instead of creating a new one [S5].
    c.  If not in cache, insert the new node into the cache and return its pointer [S5].
4.  **Propagation:** Merging opportunities propagate from leaf nodes upward [S2]. Identical subtrees at any level are merged into a single instance, and all parents point to this single instance [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VoxelNode:
    is_leaf: bool
    value: int  # 0 for passive, 1 for active
    children: list[VoxelNode] # 8 elements, None if leaf
    chnm: int # 8-bit header tag mask
    pts: list[VoxelNode] # pointers to active children

def get_octant(grid: list[list[list[int]]], octant_idx: int, size: int) -> list[list[list[int]]]:
    half = size // 2
    x0 = (octant_idx & 1) * half
    y0 = (octant_idx & 2) * half // 2
    z0 = (octant_idx & 4) * half // 4
    return [[[grid[z0+z][y0+y][x0+x] for x in range(half)] for y in range(half)] for z in range(half)]

def is_homogeneous(grid: list[list[list[int]]]) -> bool:
    val = grid[0][0][0]
    for z in grid:
        for y in z:
            for x in y:
                if x != val: return False
    return True

def build_dag(grid: list[list[list[int]]], depth: int, size: int, cache: dict) -> VoxelNode:
    # Generate unique key from grid content [G]
    key = str(tuple(grid[z][y][x] for z in range(size) for y in range(size) for x in range(size)))
    
    if key in cache:
        return cache[key]
        
    node = VoxelNode()
    if depth == 0 or is_homogeneous(grid):
        node.is_leaf = True
        node.value = grid[0][0][0]
        node.children = [None]*8
        node.chnm = 0
        node.pts = []
    else:
        node.is_leaf = False
        node.children = [None]*8
        node.chnm = 0
        node.pts = []
        new_size = size // 2
        for i in range(8):
            octant = get_octant(grid, i, new_size)
            child = build_dag(octant, depth - 1, new_size, cache)
            node.children[i] = child
            if child.is_leaf and child.value == 0:
                pass # HT=0, pruned
            else:
                node.chnm |= (1 << i)
                node.pts.append(child)
                
    cache[key] = node
    return node

def trace_ray(node: VoxelNode, origin: list[float], direction: list[float], bounds: list[float]) -> tuple[int, int, int]:
    # Standard SVDAG ray traversal [G]
    # 1. Compute ray-box intersection with current node bounds
    # 2. If no intersection, return None
    # 3. If leaf: return voxel coordinates if active, else None
    # 4. If internal: iterate i in 0..7
    #    if (node.chnm >> i) & 1:
    #        res = trace_ray(node.pts[i], origin, direction, child_bounds[i])
    #        if res: return res
    # 5. Return None
    pass
```

**Parameters:**
*   $N$: Grid dimension size, $N \ge 2$, $N=2^m$ [S1].
*   $p$: Reserved bits for alignment. Typical value: 24 bits (for 32-bit alignment) [S1].
*   $r$: Length of child node pointer. Typical value: 32 bits (for 32-bit alignment) [S1].
*   $gridDepth$: Maximum recursion depth. Default 4 (implies $16^3$ base, but typically scaled to higher resolutions like $128K^3$ via depth 16-17) [S5].

**Edge Cases and Pitfalls:**
*   **Attributes:** Adding extra bits of information (e.g., color) to leaf nodes reduces the number of identical elements, abolishing compression because previously identical leaves become unique [S4].
*   **Memory Layout:** Identical pointers exist because nodes can have multiple parents. Rearranging nodes could potentially share pointers and amortize their cost, but this is an open issue [S3].
*   **Key Generation:** The caching mechanism relies on a unique key for subtree content. Incorrect key generation leads to failure to merge identical subtrees [S5].

## Subtopics

### [Symmetry-aware Sparse Voxel DAG (SSVDAG)](1285-symmetry-aware-sparse-voxel-dag-ssvdag.md)
SSVDAG is a compact, lossless volume representation that merges subtrees identical through reflective symmetries (planar reflections along principal grid axes) and employs variable bit-rate encoding for child pointers to minimize memory footprint while maintaining GPU-friendly traversal performance. It differs from its parent [Sparse Voxel Directed Acyclic Graph (SVDAG)](962-sparse-voxel-directed-acyclic-graph-svdag.md) by recognizing geometric similarity via transformations rather than requiring exact node identity, and from [Symmetry-aware Sparse Voxel DAG (SSVDAG)](1285-symmetry-aware-sparse-voxel-dag-ssvdag.md) by explicitly detailing the binary layout and tagged pointer encoding [1349-ssvdag-node-layout-and-tagged-pointer-encoding.md].

### [PSVDAG–SVDAG Conversion Algorithm](1727-psvdag-svdag-conversion-algorithm.md)
The PSVDAG–SVDAG Conversion Algorithm transforms a Pointerless Sparse Voxel Directed Acyclic Graph (PSVDAG) into a Sparse Voxel Directed Acyclic Graph (SVDAG). While PSVDAGs use variable-length Labels/Callers and intermittent binary encoding to save space, SVDAGs use constant-length 32-bit pointers and uninterrupted binary sequences to enable efficient rendering and editing. This conversion replaces the pointerless structure with a pointer-based one, allowing for Child Subtree Merging (CSM) and random access, at the cost of increased memory usage (inflation factor 3.01x–3.70x).

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** The DAG consumes 8 to 36 bytes per node [S2]. It reduces total memory consumption by up to 38x compared to ESVO and up to 26x compared to a minimal pointer-less SVO [S2].
*   **Node Reduction:** In the CRYSPONZA scene, a 576x reduction in node count was achieved [S2]. In the HAIRBALL scene, a 28x reduction was achieved despite lacking obvious regularities [S2].
*   **Build Time:** Building the DAG for CRYSPONZA at $512^3$ takes 8.4ms total (1.9ms sorting) [S2]. At $8K^3$, it takes 4.5s total (0.6s sorting) [S2]. For HAIRBALL at $8K^3$, it takes 40.6s total (4.5s sorting) [S2].

**Measured Performance:**
*   In the EPICCITADEL scene voxelized to $128K^3$ resolution, the SVDAG stores 19 billion voxels in 945MB of GPU memory [S2]. An SVO would require 5.1GB (excluding pointers) [S2].
*   On an NVIDIA GTX680, ray tracing in the sparse voxel DAG achieves 170 MRays/sec for ambient occlusion and 240 MRays/sec for shadows [S2].
*   The new encoding (Modisett & Billeter) reduces geometric data size by 13%-18% compared to the original SVDAG [S3].
*   The new encoding improves rendering time by around 10%-12% for static models and by up to 20%-25% for editable voxel models [S3].

**Strengths and Limitations:**
*   **Strengths:** No decompression overhead; efficient traversal; handles extremely high resolutions (up to $128K^3$) in real-time [S3].
*   **Limitations:** Higher memory usage per node compared to pointerless structures due to pointers [S1]. Adding attributes to leaves destroys compression benefits [S4].

## Background
High Resolution Sparse Voxel Directed Acyclic Graphs (SVDAGs) were proposed in 2013 by Kämpe, Sintorn, and Assarsson [S1, S3]. The method was introduced to address the limitations of Sparse Voxel Octrees (SVOs) and Efficient Sparse Voxel Octrees (ESVOs) by exploiting geometric similarity (identical subtrees) in addition to sparsity (empty regions) [S2, S3]. It allows for the storage and rendering of voxel geometry at resolutions as high as $(128k)^3$ in real time [S3].

## Sources
[S1] Transforming Hierarchical Data Structures – a PSVDAG–SVDAG Conversion Algorithm, https://doi.org/10.12700/aph.18.8.2021.8.3, fetched 2026-10-30
[S2] PDF High Resolution Sparse Voxel DAGs - Chalmers, https://www.cse.chalmers.se/~uffe/HighResolutionSparseVoxelDAGs.pdf, fetched 2026-10-27
[S3] (PDF) High resolution sparse voxel DAGs - ResearchGate, https://www.researchgate.net/publication/262367808_High_resolution_sparse_voxel_DAGs, fetched 2026-10-27
[S4] PDF On sparse voxel DAGs and memory efficient compression of surface ..., https://research.chalmers.se/publication/528795/file/528795_Fulltext.pdf, fetched 2026-10-27
[S5] Building a High-Performance Voxel Engine in Unity: A Step ..., https://medium.com/@adamy1558/building-a-high-performance-voxel-engine-in-unity-a-step-by-step-guide-part-6-high-resolution-e0217efd56d2, fetched 2026-10-27 (unofficial)
[S6] PSVDAG: Compact Voxelized Representation of 3D Scenes Using Pointerless Sparse Voxel Directed Acyclic Graphs, https://doi.org/10.31577/cai_2020_3_587, fetched 2026-10-26
