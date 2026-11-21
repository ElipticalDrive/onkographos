# Hash Grid with Multiple Hash Functions

Also called: Instant-NGP Multi-Hash, Multi-Hash Hash Grid

## Overview
This method extends the standard Hash Grid by assigning a distinct hash function to each leaf node in the spatial hierarchy, thereby reducing hash collisions that occur when different regions of space map to the same warp space coordinates [S1]. While the parent [Hash Grids](936-hash-grids.md) uses a single global hash function for all vertices, this variant conditions the hashing parameters on the leaf node index, allowing points from different leaf nodes that occupy the same voxel in the shared warp space to be stored in different buckets [S1].

## How it works
**Inputs:**
- A 3D point $\mathbf{x}$ located within a specific leaf node $i$ [S1].
- A warp function $F_i(\cdot)$ mapping local coordinates to a shared warp space [S1].
- A hash table of length $L$ [S1].
- Precomputed random large prime numbers $\pi_{i,k}$ and $\Delta_{i,k}$ for each leaf node $i$ and coordinate $k \in \{1, 2, 3\}$ [S1].

**Outputs:**
- A feature vector for point $\mathbf{x}$, obtained by trilinear interpolation of feature vectors from the 8 neighboring grid vertices in the warp space [S1].

**Data Structures:**
- **Hash Table:** An array of size $L$ storing feature vectors [S1].
- **Hash Parameters:** For each leaf node $i$, a set of 6 constants: $\pi_{i,1}, \pi_{i,2}, \pi_{i,3}$ (primes) and $\Delta_{i,1}, \Delta_{i,2}, \Delta_{i,3}$ (offsets/primes) [S1].

**Steps:**
1. **Warping:** Map the input point $\mathbf{x}$ from the $i$-th leaf node to the warp space using the leaf-specific warp function: $\mathbf{z} = F_i(\mathbf{x})$ [S1].
2. **Neighbor Identification:** Identify the 8 neighboring grid vertices $\hat{\mathbf{v}}$ of $\mathbf{z}$ in the warp space. These vertices have integer coordinates $\hat{\mathbf{v}}_k = \lfloor z_k \rfloor$ or $\lceil z_k \rceil$ for $k=1,2,3$ [S1].
3. **Hashing:** For each neighbor vertex $\hat{\mathbf{v}}$, compute the hash index using the leaf-node-specific hash function [S1]:
   $$ \text{Hash}_i(\hat{\mathbf{v}}) = \left( ((\hat{\mathbf{v}}_1 \oplus \pi_{i,1}) + \Delta_{i,1}) \oplus ((\hat{\mathbf{v}}_2 \oplus \pi_{i,2}) + \Delta_{i,2}) \oplus ((\hat{\mathbf{v}}_3 \oplus \pi_{i,3}) + \Delta_{i,3}) \right) \bmod L $$
   where $\oplus$ denotes bitwise XOR, $+$ denotes integer addition, and $L$ is the hash table length [S1]. The operations are grouped per coordinate dimension before being combined via XOR [S1]. Note: The source text contains a typo stating "$L$ denotes the bitwise xor operation"; $L$ is the table length, and $\oplus$ is the XOR operator [S1].
4. **Lookup:** Use the computed hash index to retrieve the feature vector for vertex $\hat{\mathbf{v}}$ from the hash table [S1].
5. **Interpolation:** Trilinearly interpolate the feature vector of point $\mathbf{z}$ from the 8 retrieved vertex feature vectors [S1].
6. **MLP Evaluation:** Feed the interpolated feature vector and the view direction $\mathbf{d}$ into a tiny MLP network to produce color and density [S1].

**Parameters:**
- $\pi_{i,k}$: Random large prime numbers, fixed for leaf node $i$ and coordinate $k$ [S1].
- $\Delta_{i,k}$: Random large prime numbers (offsets), fixed for leaf node $i$ and coordinate $k$ [S1].
- $L$: Length of the hash table [S1].

**Edge Cases:**
- **Collision:** If two different vertices map to the same hash index, the hash table must handle collisions (e.g., via chaining or open addressing), though the multi-hash design aims to minimize this [S1].
- **Precision:** Warp space coordinates $\mathbf{z}$ may be floating-point; neighbor identification requires careful handling of boundaries (e.g., negative coordinates) if the warp space allows them [G].

## Subtopics

### [Adapted Cuckoo Hashing for Voxel Labels](10183-adapted-cuckoo-hashing-for-voxel-labels.md)
A hash table method for caching voxel labels ($\lambda$) and keys ($k$) in 4D space (voxel $\mathbf{v}$, LOD $l$). It uses two hash functions to determine positions, allowing elements to be stored in either a first or second position. This supports constant-time lookups and handles collisions via ejection, but is adapted to handle cases where an element cannot be moved back to its first position.

### [Voxel Caching with Cuckoo Hashing](10184-voxel-caching-with-cuckoo-hashing.md)
An optional voxel cache using Cuckoo hashing to store labels $\lambda$ for tuples of voxel coordinates $v \in \mathbb{N}^3$ and requested LODs $l$, allowing direct access without one-frame latency.

## Operating characteristics
- **Time Complexity:** Lookup is $O(1)$ per vertex, with 8 lookups and trilinear interpolation per point. The multi-hash adds negligible overhead compared to single-hash as parameters are precomputed [S1].
- **Memory Complexity:** $O(L)$ for the hash table, where $L$ is the table length [S1].
- **Strengths:** Reduces hash conflicts between points from different leaf nodes that map to the same warp space location, improving feature distinctiveness [S1].
- **Limitations:** Requires storing separate hash parameters ($\pi, \Delta$) for each leaf node, increasing metadata size slightly compared to a single global hash [S1].

## Background
This method was introduced in F2-NeRF to address the conflict problem in the original Instant-NGP [S1]. By using different hash functions for different leaf nodes, it alleviates the issue where points from different spatial regions collide in the hash table after warping [S1].

## Sources
[S1] F2-NeRF: Fast Neural Radiance Field Training with Free Camera Trajectories, https://doi.org/10.1109/cvpr52729.2023.00404, fetched 2026-10-26
