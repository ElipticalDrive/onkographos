# Voxel Attribute Decoupling

Also called: DAG-based attribute decoupling

## Overview
Voxel Attribute Decoupling is a technique that separates voxel attributes (such as color or normals) from the geometric topology of a Sparse Voxel Octree (SVO) to enable independent, specialized compression of each component [S1]. Unlike standard SVOs or naive colored DAGs that store attributes redundantly with every node, this method achieves significant memory reduction by storing attributes in a dense array and encoding geometric offsets within child pointers, allowing subtrees with identical topology to be merged into a Directed Acyclic Graph (DAG) [S1].

## How it works
**Inputs:** An initial Sparse Voxel Octree (SVO) with voxel attributes [S1].
**Outputs:** A DAG-based geometric structure and a dense attribute array [S1].

**Data Structures:**
1. **DAG Nodes:** Each node stores child pointers [S1]. Each child pointer contains an offset value instead of a direct index [S1].
2. **Attribute Array:** A dense 1D array storing voxel attributes in depth-first order [S1].

**Steps:**
1. **Index Assignment:** Virtually assign unique indices to all nodes in the initial SVO using a depth-first traversal order [S1].
2. **Offset Calculation:** For every child pointer in the SVO, calculate an offset equal to the difference between the child's index and the parent's index associated with that pointer [S1].
    $$ \text{offset} = \text{index}_{\text{child}} - \text{index}_{\text{parent}} $$
3. **Storage:** Store these calculated offsets together with each child pointer in the DAG structure [S1].
4. **Attribute Extraction:** Extract all node attributes from the SVO and store them in a dense attribute array, maintaining the same depth-first order used for index assignment [S1].
5. **Traversal and Retrieval:** During rendering or traversal, reconstruct a node's original index by summing the offsets along the path from the root to that node [S1]. Use this reconstructed index to retrieve the corresponding attribute from the dense attribute array [S1].

**Edge Cases and Pitfalls:**
* **Overhead:** The mapping introduces storage overhead in the form of an additional offset for every child pointer [S1].
* **Compression:** While offsets add overhead, they can be compressed very efficiently using entropy-based encoding, which amortizes the additional storage cost [S1].
* **DAG Merging:** This method allows subtrees with identical topology to be merged into a DAG because depth-first indexing automatically results in identical offsets for geometrically identical subtrees [S1].

## Subtopics

### [Decoupled SVO Compression](1872-decoupled-svo-compression.md)
Decoupled SVO Compression is a memory optimization technique for Sparse Voxel Octrees (SVOs) that separates the geometric topology from voxel attributes (such as color, normals, or reflectance) to allow independent, specialized compression of each component. Unlike standard SVOs or naive colored DAGs that store attributes redundantly with every node, this method achieves significant memory reduction by compressing the sparse geometry structure and applying lossless palette-based compression to the decoupled attribute array, outperforming existing methods like PSVOs, ESVOs, and CDAGs in both memory footprint and visual quality.

### Attribute Decoupling via Leaf Counter
A technique to decouple attributes (like color) from the DAG geometry to allow independent compression, using a leaf counter to map voxel indices to attribute data without altering DAG node uniqueness [S2].

* **Definition:** Uses a leaf counter to determine the position of a node's attributes within the dense attribute array based on the number of leaf nodes in preceding siblings [S2].
* **Steps:**
  1. For DAG resolutions larger than $1024^3$, store an offset for the upper levels into a separate 32-bit counter array [S2].
  2. Use the leaf counter to calculate the attribute index by summing the leaf counts of preceding child nodes during traversal [S2].
* **Layout:** The DAG node structure is stored in a 32-bit array consisting of an 8-bit child mask and 24-bit padding [S2]. The 24-bit padding is repurposed as the leaf counter [S2].
* **Parameters:** Leaf counter size is 24 bits, which is sufficient for DAG resolutions up to $1024^3$ [S2].
* **Pitfalls:** Using per-node counters requires reading preceding child counters for each node, resulting in extra memory reads compared to per-pointer counters [S2].
* **Comparison:** Per-pointer counter lookups are 1.2x-1.9x faster than per-node counter lookups [S2]. However, per-node counters have negligible memory overhead (<0.1%) compared to per-pointer counters, which result in a 1.5x-1.8x larger DAG [S2].

## Operating characteristics
* **Memory:** Introduces overhead for offsets in child pointers [S1]. Per-pointer counters increase DAG size by 1.5x-1.8x, while per-node counters add <0.1% overhead [S2].
* **Performance:** Per-pointer counter lookups are 1.2x-1.9x faster than per-node counter lookups due to reduced memory reads [S2].
* **Compression:** Offsets can be compressed efficiently using entropy-based encoding [S1]. Attributes can be compressed using palette-based methods after decoupling [S1].
* **Scalability:** Per-node counters require a separate 32-bit counter array for resolutions larger than $1024^3$, though the overhead remains minimal [S2].

## Background
The concept of decoupling attributes from geometry in voxel structures was introduced to address the inefficiency of storing redundant attributes in standard SVOs and the limitations of naive DAG compression [S1]. The method builds on the principles of space-filling curves, such as the Morton curve, which allow for efficient mapping between 1D arrays and N-dimensional voxel grids [S2]. The specific technique of using depth-first indexing to generate identical offsets for identical subtrees enables the re-application of DAG compression to decoupled structures [S1].

## Sources
[S1] Geometry and Attribute Compression for Voxel Scenes, https://timothykol.com/pub/compression.pdf, fetched 2026-10-27
[S2] On sparse voxel DAGs and memory efficient compression of surface ..., https://research.chalmers.se/publication/528795/file/528795_Fulltext.pdf, fetched 2026-10-27
