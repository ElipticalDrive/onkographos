# Hash Grid with Multiple Hash Functions

Also called: Instant-NGP Multi-Hash, Leaf-conditioned Hashing, Multi-Hash Hash Grid, Multi-hash Grid

## Overview
This method extends the standard Hash Grid by assigning a distinct hash function to each leaf node in the spatial hierarchy, thereby reducing hash collisions that occur when different regions of space map to the same bucket indices [S1]. It solves the problem of feature vector interference in sparse volume representations, where standard single-hash schemes cause artifacts due to conflicting features from distant geometry [S1]. Unlike its parent [Hash Grids](936-hash-grids.md), which uses a uniform multiplicative hash for all vertices, this variant conditions the hash parameters on the leaf node index, ensuring that geometrically distinct regions maintain independent feature spaces [S1].

## How it works
**Inputs and Outputs:**
- Input: A 3D point $\mathbf{x}$ in world space, the leaf node index $i$ containing $\mathbf{x}$, and the warp function $F_i$ mapping world space to warp space [S1].
- Output: A feature vector $\mathbf{f}$ at $\mathbf{x}$, derived via trilinear interpolation from the hash grid [S1].

**Data Structures:**
- **Hash Table:** An array of size $L$ storing feature vectors [S1].
- **Hash Parameters:** For each leaf node $i$, three pairs of random large prime numbers $(\pi_{i,k}, \Delta_{i,k})$ for $k \in \{1, 2, 3\}$ are stored [S1]. These are fixed constants for the lifetime of the representation [S1].

**Steps:**
1. **Warping:** Map the input point $\mathbf{x}$ to warp space using the leaf-specific function $F_i$ to obtain $\mathbf{z} = F_i(\mathbf{x})$ [S1].
2. **Neighbor Identification:** Identify the eight neighboring grid vertices $\hat{\mathbf{v}}$ of $\mathbf{z}$ in warp space. These vertices have integer coordinates $\hat{\mathbf{v}}_j$ where $j \in \{0, \dots, 7\}$ [S1].
3. **Hash Computation:** For each neighbor $\hat{\mathbf{v}}$, compute the hash index using the leaf-specific parameters:
   $$ \text{Hash}_i(\hat{\mathbf{v}}) = \left( ((\hat{\mathbf{v}}_1 \oplus \pi_{i,1}) + \Delta_{i,1}) \oplus ((\hat{\mathbf{v}}_2 \oplus \pi_{i,2}) + \Delta_{i,2}) \oplus ((\hat{\mathbf{v}}_3 \oplus \pi_{i,3}) + \Delta_{i,3}) \right) \bmod L $$
   where $\oplus$ denotes bitwise XOR, $\hat{\mathbf{v}}_k$ is the $k$-th coordinate of the vertex, $\pi_{i,k}$ and $\Delta_{i,k}$ are the random primes for leaf $i$ and coordinate $k$, and $L$ is the hash table length [S1].
4. **Lookup:** Retrieve the feature vectors $\mathbf{f}_{\hat{\mathbf{v}}}$ from the hash table at the computed indices [S1].
5. **Interpolation:** Trilinearly interpolate the feature vector $\mathbf{f}_z$ from the 8 retrieved vertex feature vectors based on the fractional part of $\mathbf{z}$ [S1].
6. **MLP Evaluation:** Feed $\mathbf{f}_z$ and the view direction $\mathbf{d}$ into a tiny MLP to produce color and density [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_hash(v_hat, pi, delta, L):
    # v_hat: tuple of 3 integer coordinates
    # pi, delta: tuples of 3 random large primes for leaf node i
    term1 = (v_hat[0] ^ pi[0]) + delta[0]
    term2 = (v_hat[1] ^ pi[1]) + delta[1]
    term3 = (v_hat[2] ^ pi[2]) + delta[2]
    return (term1 ^ term2 ^ term3) % L

def get_feature(x, leaf_index, F_i, hash_table, pi_params, delta_params, L):
    z = F_i(x)
    # Find 8 neighboring grid vertices with integer coordinates
    v_hats = []
    for dx in (0, 1):
        for dy in (0, 1):
            for dz in (0, 1):
                v_hats.append((int(z[0]) + dx, int(z[1]) + dy, int(z[2]) + dz))
    # Retrieve features
    features = []
    for v in v_hats:
        idx = compute_hash(v, pi_params[leaf_index], delta_params[leaf_index], L)
        features.append(hash_table[idx])
    # [G] Standard trilinear interpolation of 8 features based on fractional coordinates
    frac = z - floor(z)
    f_z = interpolate_trilinear(features, frac)
    return f_z
```

**Parameters:**
- $L$: Length of the hash table [S1].
- $\pi_{i,k}$: Random large prime numbers, fixed for leaf node $i$ and coordinate $k$ [S1].
- $\Delta_{i,k}$: Random large prime numbers (offsets), fixed for leaf node $i$ and coordinate $k$ [S1].

**Edge Cases and Pitfalls:**
- **Collision:** If $L$ is too small, different leaf nodes may map to the same bucket, causing feature bleeding [S1].

## Subtopics

### [Adapted Cuckoo Hashing for Voxel Labels](10183-adapted-cuckoo-hashing-for-voxel-labels.md)
Adapted Cuckoo Hashing for Voxel Labels is a sparse data structure method for caching voxel labels ($\lambda$) and their corresponding keys ($k$) in 4D space (voxel coordinate $\mathbf{v}$ and level of detail $l$) during volume rendering. It solves the problem of efficiently retrieving pre-computed labels for sparse visible regions while minimizing memory footprint and lookup latency. Unlike standard Cuckoo hashing which guarantees insertion via complex reorganization, this variant is adapted to handle cases where an element cannot be moved back to its first position by removing it instead, and limits ejection recursion depth to 1 to reduce thread divergence in parallel renderers.

### [Voxel Caching with Cuckoo Hashing](10184-voxel-caching-with-cuckoo-hashing.md)
An optional voxel cache using Cuckoo hashing to store labels $\lambda$ for tuples of voxel coordinates $v \in \mathbb{N}^3$ and requested LODs $l$, allowing direct access without one-frame latency. It differs from its parent [Hash Grid with Multiple Hash Functions](7678-hash-grid-with-multiple-hash-functions.md) by using a single combined table with a compact 32-bit key and specific rounding to handle multi-resolution queries, and from its sibling [Adapted Cuckoo Hashing for Voxel Labels](10183-adapted-cuckoo-hashing-for-voxel-labels.md) by its specific 4D tuple handling and PCG hash integration.

### Dimension-Wise Context Model
The Dimension-Wise Context Model leverages cross-dimensional relations by projecting 3D voxel features onto 2D planes to create Projected Voxel Features (PVF), which serve as context for decoding 2D tri-plane embeddings [S3]. It reconstructs the entire 3D voxel using the spatial hashing function, then projects the 3D voxel along three axes (xy, xz, yz) to record the frequency of +1s along each direction, obtaining 2D PVFs [S3]. During projection, the occupancy grid prior is used; if a vertex's AOE is 0, it is omitted [S3]. The PVFs serve as an additional "previous level" context to estimate the probability $p_i$ for each 2D feature $\theta_i$, enabling coarse-to-fine decoding of 2D embeddings [S3]. This method utilizes 3D feature embeddings at the largest resolution to generate PVFs [S3].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Mitigates feature interference in sparse scenes by reducing hash collisions between leaf nodes [S1].
- **Limitations:** Requires storage of hash parameters ($\pi, \Delta$) for each leaf node [S1].

## Background
The method originates from F2-NeRF, which introduced multi-hash functions to alleviate the conflict problem in Instant-NGP's hash grid [S1]. It builds on the [Hash Grids](936-hash-grids.md) technique by conditioning the hash function on the leaf node index [S1].

## Sources
[S1] F2-NeRF: Fast Neural Radiance Field Training with Free Camera Trajectories, https://doi.org/10.1109/cvpr52729.2023.00404, fetched 2026-10-26
[S2] F2-NeRF: Fast Neural Radiance Field Training with Free Camera Trajectories, https://openaccess.thecvf.com/content/CVPR2023/html/Wang_F2-NeRF_Fast_Neural_Radiance_Field_Training_With_Free_Camera_Trajectories_CVPR_2023_paper.html, fetched 2026-10-27 (unofficial)
[S3] How Far can we Compress Instant-NGP-Based NeRF?, https://doi.org/10.1109/cvpr52733.2024.01921, fetched 2026-10-26
