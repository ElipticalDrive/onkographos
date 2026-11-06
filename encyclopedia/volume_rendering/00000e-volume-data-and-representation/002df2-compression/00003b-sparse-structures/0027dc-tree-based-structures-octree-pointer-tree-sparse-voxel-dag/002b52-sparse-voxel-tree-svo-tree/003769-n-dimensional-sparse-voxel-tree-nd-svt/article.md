# n-Dimensional Sparse Voxel Tree (nD SVT)
Also called: nD SVT

## Overview
The n-Dimensional Sparse Voxel Tree (nD SVT) is a hierarchical data structure that generalizes Sparse Voxel Octrees (SVOs) to $n$-dimensional grids, where $n > 3$, to achieve a more compact binary representation of voxelized scenes [S1]. It solves the problem of storage efficiency in 3D volume rendering by upscaling the dimensionality of the scene from 3D to $m$D ($m > 3$), reducing the total number of nodes and pointers required to represent the geometry [S1]. Unlike its sibling [Morton-coded Sparse Voxel Tree](5212-morton-coded-sparse-voxel-tree.md) which remains strictly 3D, or [Hashed-Octree](7486-hashed-octree.md) which uses hashing for flat storage, the nD SVT trades increased per-node complexity (larger child masks) for a significantly reduced tree depth and node count [S1].

## How it works
### Inputs and Outputs
**Input:** A voxelized 3D scene defined on a regular grid of size $N \times N \times N$ voxels, where $N = 2^m$ for $m \ge 1$. Each voxel has coordinates $(X, Y, Z)$ where $X, Y, Z \in [0, 2^m - 1]$ [S1].
**Output:** An nD SVT data structure representing the scene in $D$ dimensions, where $D > 3$. The structure is linearized using Space Filling Curves (SFCs) such as Morton or Hilbert curves [S1].

### Data Structures and Layout
The nD SVT is defined by the following Backus-Naur Form (BNF) [S1]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```bnf
SVT ::= (n) <NODE>
NODE ::= <INODE> | <LNODE>
INODE ::= <CHNM>(p) <BIT><PTS>
LNODE ::= <CHNM>(q) <BIT>
CHNM ::= (2^D)<HT>
PTS ::= (1)*(2^D)<PT>
PT ::= (r)<BIT>
HT ::= <BIT>
BIT ::= "0" | "1"
```

Where:
*   $D$ is the dimensionality of the SVT.
*   $n$ is the number of internal nodes.
*   $m$ is the number of leaf nodes.
*   **CHNM** (Child Node Mask): A bitfield of size $2^D$ bits. It indicates which of the $2^D$ possible children in the $D$-dimensional hypercube are present.
*   **PTS** (Pointer Table): A variable-length array of pointers to child nodes. The number of pointers ranges from 1 to $2^D$, corresponding to the number of set bits in CHNM.
*   **PT** (Pointer): A pointer to a child node, size $r$ bits.
*   **HT** (Hash/Type): A bit indicating node type or hash information.

**Binary Representation Size:**
The total size of the data structure in bits is calculated as [S1]:

$$
\text{Size}_{nD SVT} = (n + m) \times \text{CHNM}_{size} + (n + m - 1) \times \text{PTS}_{size}
$$

Where:
*   $\text{CHNM}_{size} = 2^D$ bits.
*   $\text{PTS}_{size}$ is the total size of all pointers in the structure.

### Coordinate Transformation
To map $n$-dimensional coordinates to a 1D linear space for traversal and storage, the method uses bit interleaving (Morton code) [S1].

**1. Upscaling (3D to $D$-dimensional):**
To transform 3D coordinates $(X, Y, Z)$ into a $D$-dimensional coordinate system, the scene is conceptually mapped to a higher-dimensional grid. The specific geometric mapping of the 3D scene into $D$ dimensions is not detailed in the sources, but the linearization relies on the resulting $D$ coordinates [S1].

**2. Linearization (nD to 1D Morton Coordinate):**
To transform coordinates from $D$ dimensions to a 1D Morton coordinate $M$:
1.  Extract the binary representation of each of the $D$ coordinates.
2.  Interleave the bits of these $D$ coordinates to form $M$.
    *   If each coordinate has $k$ bits, $M$ will have $D \times k$ bits.
    *   $M \in [0, 2^{D \cdot k} - 1]$ [S1].

**3. De-linearization (1D Morton to nD):**
To transform from 1D Morton coordinate $M$ back to $D$-dimensional coordinates:
1.  De-interleave the bits of $M$ into $D$ separate coordinate values [S1].

### Parameters
*   **$D$ (Dimensionality):** The number of dimensions of the SVT. Typical values tested are 3, 4, 5, 6, 7, and 8 [S1].
    *   CHNM size for 3D SVT: $2^3 = 8$ bits.
    *   CHNM size for 4D SVT: $2^4 = 16$ bits.
    *   CHNM size for 8D SVT: $2^8 = 256$ bits [S1].
*   **$n$:** Number of internal nodes.
*   **$m$:** Number of leaf nodes [S1].

### Algorithm: Traversal
Traversal is performed from the root to leaf nodes for all active voxels. The process is parallelized on GPU hardware [S1].

1.  **Initialize:** Start at the root node of the nD SVT.
2.  **Compute Morton Code:** Calculate the 1D Morton coordinate $M$ for the target voxel based on its $D$-dimensional position.
3.  **Iterate Bits:** For each level of the tree, extract the relevant bits from $M$ to determine the child index.
4.  **Check Mask:** Use the CHNM of the current node to check if the target child exists.
    *   If the bit corresponding to the child index is 0, the path is invalid or the voxel is empty.
    *   If the bit is 1, follow the corresponding pointer in the PTS array.
5.  **Advance:** Move to the child node.
6.  **Termination:** Repeat until a leaf node (LNODE) is reached.

*Note: The sources state that the number of operations per node remains constant regardless of dimensionality, but the size of the CHNM and PTS arrays grows [S1].*

### Edge Cases and Pitfalls
*   **Alignment:** Node components must be aligned to 32 bits for GPU-friendliness [S1].
    *   In 3D and 4D SVTs, reserved bits are required for alignment, increasing the binary size.
    *   From 5D SVT onwards, components are naturally aligned to 32 bits. This reduces the binary representation size by 24 bits compared to 3D SVTs and 16 bits compared to 4D SVTs for each node [S1].
*   **Cache Misses:** As dimensionality increases, the CHNM size grows exponentially. In 7D and 8D, the CHNM length exceeds 64 bits, leading to a higher probability of cache misses and negatively impacting traversal speed [S1].
*   **Optimal Dimensionality:** There is no single optimal $D$ for all scenes. The optimal dimensionality balances the sharp increase in CHNM size against the reduction in the total number of nodes and pointers. For the tested models, 6D and 7D provided the most compact representations [S1].

## Operating characteristics
### Time and Memory Complexity
*   **Time Complexity:** Traversing an nD SVT has linear time complexity $O(n)$, where $n$ is the number of nodes to be traversed [S1].
*   **Memory Complexity:** The memory footprint is determined by the binary size formula. The CHNM size doubles for every increase in dimensionality by 1 [S1].

### Measured Performance
Tests were conducted on an NVIDIA GeForce RTX 3060 with scenes voxelized at $1024^3$ resolution [S1].

**Traversal Times (µs):**

| Model | 3D SVT | 4D SVT | 5D SVT | 6D SVT | 7D SVT | 8D SVT |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Angel Lucy | 99.4 | 89.1 | 75.8 | 73.7 | 158.8 | 244.7 |
| Porsche | 257.0 | 225.3 | 198.7 | 190.5 | 428.0 | 654.3 |
| Skull | 313.4 | 275.5 | 240.7 | 231.4 | 486.4 | 777.2 |

The 6D SVT proved to be the most efficient in traversal speed, offering a speedup ratio of 1.35 compared to 3D SVT for all tested models [S1]. Performance degrades significantly in 7D and 8D due to CHNM sizes exceeding 64 bits [S1].

**Compression Ratio (Binary Size):**
The ratio of the binary representation size of the nD SVT to the 3D SVT (without 32-bit alignment optimization) ranged from 2.02 (6D SVT, Skull model, $4096^3$ resolution) to 2.77 (7D SVT, Porsche model, $128^3$ resolution) [S1].

When node components are aligned to 32 bits (optimal for GPU), the nD SVT achieves a 3.23 to 4.44 times more compact binary representation compared to the classical 3D SVO [S1].

### Strengths and Limitations
*   **Strengths:** Significantly reduced binary size (2x to 4.4x smaller than 3D SVO); reduced tree depth; optimal performance at 6D dimensionality for traversal speed.
*   **Limitations:** Increased per-node complexity; poor performance in 7D/8D due to cache misses from large CHNM (>64 bits); requires dimensionality upscaling logic [S1].

## Background
The n-Dimensional Sparse Voxel Tree was introduced by B. Madoš et al. in the paper "n-Dimensional Sparse Voxel Trees for Compact Representation of Dimensionally Upscaled Voxelized Three-Dimensional Scenes," published in *Acta Polytechnica Hungarica*, Vol. 22, No. 9, 2025 [S1]. The work explores the concept of dimensionality upscaling, transforming 3D voxelized scenes into higher-dimensional representations to leverage the sparsity of the data in higher-dimensional space, thereby reducing the overall storage footprint [S1].

## Sources
[S1] n-Dimensional Sparse Voxel Trees for Compact Representation of Dimensionally Upscaled Voxelized Three-Dimensional Scenes, https://doi.org/10.12700/aph.22.9.2025.9.1, fetched 2026-10-27
