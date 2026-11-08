# Empty Space Caching with Bit Vector E

Also called: Empty block skipping

## Overview
Empty Space Caching with Bit Vector E is a fine-granular optimization for sparse volume rendering that reduces memory pressure and lookup latency by marking regions of invisible data. It complements the parent method, [Adapted Cuckoo Hashing for Voxel Labels](10183-adapted-cuckoo-hashing-for-voxel-labels.md), by introducing a compact bit vector $E$ that allows the renderer to skip hash table lookups for entire blocks of voxels known to be invisible. While the parent hash table $M$ stores explicit labels for visible and cached invisible voxels, $E$ provides a 1-bit-per-8-voxel summary, reducing the storage cost for fully invisible regions from 64 bits per voxel in $M$ to 1/8 bits per voxel in $E$ [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Voxel coordinate $v$ (in finest LOD), Level of Detail $l$, visibility transfer function.
*   **Output:** Label $\lambda$ or `INVISIBLE`.

**Data Structures**
*   **Bit Vector $E$:** A bit array where each bit corresponds to an "empty block" of $2^3 = 8$ voxels on the finest LOD.
    *   Size: $N_{blocks} = \lceil N_{voxels} / 8 \rceil$ bits.
    *   Layout: $E[e] = 1$ if all 8 voxels in block $e$ have invisible labels; $E[e] = 0$ otherwise.
    *   Initialization: All zeros $(000...000)_2$. Reset whenever the visibility transfer function changes [S1].
*   **Hash Table $M$:** Provided by the parent method [Adapted Cuckoo Hashing for Voxel Labels](10183-adapted-cuckoo-hashing-for-voxel-labels.md). Stores pairs $(k, \lambda)$.

**Parameters**
*   **Empty Block Size:** $2^3$ voxels (8 voxels) [S1].
*   **Brick Size $b$:** Used for coarse skipping (from parent context/CSGV) [S1].

**Algorithm Steps**
The voxel access method integrates brick skipping, empty block checking, hash lookup, and dynamic caching:

1.  **Brick Skipping:** Check if voxel $v$ belongs to a brick marked as invisible. If so, return `INVISIBLE` [S1].
2.  **Empty Block Check:**
    *   Calculate the empty block index $e$ for voxel $v$.
    *   If $E[e] == 1$, return `INVISIBLE` [S1].
3.  **Hash Lookup:** Perform Cuckoo hash lookup in $M$ for key $k = h(v, l)$ [S1].
    *   If a match is found, return the cached label $\lambda$ [S1].
4.  **Decode and Verify:**
    *   If hash miss, decode the label $\lambda$ using random access (e.g., CSGV-R) [S1].
    *   Check if $\lambda$ is invisible.
    *   If $\lambda$ is visible, cache $(k, \lambda)$ in $M$ and return $\lambda$ [S1].
    *   If $\lambda$ is invisible, check all 8 voxels $v'$ in the empty block $e$ at finest LOD ($l=0$):
        *   Decode label for each $v'$.
        *   If any $v'$ has a visible label, set `allInvisible = False`.
        *   If `allInvisible` is True, set $E[e] \leftarrow 1$ and return `INVISIBLE` [S1].
        *   If `allInvisible` is False, cache $(k, \lambda)$ in $M$ and return `INVISIBLE` (or the decoded label if visible, but here it is invisible, so return `INVISIBLE` after caching) [S1]. Note: The source states "return INVISIBLE" after setting $E[e]=1$. If not all invisible, it caches in $M$. The return value for the invisible case where not all are invisible is implicitly `INVISIBLE` for the current voxel, but the block is not marked in $E$.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_label_of_voxel(v, l, E, M, brick_invisible_map, decode_label, label_visible, h1, h2):
    # 1. Brick Skipping
    brick_idx = v // brick_size
    if brick_invisible_map[brick_idx]:
        return INVISIBLE

    # 2. Empty Block Check
    e = get_empty_block_index(v)
    if E[e] == 1:
        return INVISIBLE

    # 3. Hash Lookup
    k = hash_key(v, l)
    p1 = h1(v, l)
    k1, lambda1 = M[p1]
    if k1 == k:
        return lambda1

    p2 = h2(k, p1)
    k2, lambda2 = M[p2]
    if k2 == (k | 0x80000000): # Check second position flag
        return lambda2

    # 4. Decode and Verify
    lambda_decoded = decode_label(v, l)
    
    if not label_visible(lambda_decoded):
        # Check if all voxels in the empty block are invisible
        all_invisible = True
        for v_prime in get_voxels_in_empty_block(e):
            # Decode at finest LOD
            lambda_prime = decode_label(v_prime, 0)
            if label_visible(lambda_prime):
                all_invisible = False
                break
        
        if all_invisible:
            E[e] = 1
            return INVISIBLE
        else:
            # Cache the invisible label in M for future lookups
            M[p1] = (k, lambda_decoded)
            return INVISIBLE
    else:
        # Visible label, cache it
        M[p1] = (k, lambda_decoded)
        return lambda_decoded
```

**Edge Cases and Pitfalls**
*   **Partially Visible Blocks:** If an empty block contains both visible and invisible voxels, $E[e]$ remains 0. All voxels in such a block must be decoded and cached in $M$ individually. The bit vector $E$ only marks *fully* invisible blocks [S1].
*   **Transfer Function Changes:** $E$ must be reset to all zeros whenever the visibility transfer function changes, as the invisibility status of voxels may change [S1].
*   **Atomicity:** Insertions into $E$ and $M$ should be atomic if accessed in parallel [S1].

## Operating characteristics
*   **Memory:** $E$ uses 1 bit per 8 voxels for invisible regions. $M$ uses 64 bits per voxel (32 for $\lambda$, 32 for $k$) for cached voxels. $E$ provides significant memory savings for large invisible regions [S1].
*   **Performance:** Reduces hash table lookups for invisible voxels by checking $E$ first. However, partially visible blocks require full decoding and caching, potentially increasing CPU/GPU workload for those regions [S1].
*   **Scaling:** Effective for volumes with large contiguous invisible regions. Less effective for volumes with scattered invisible voxels or high frequency of transfer function changes [S1].

## Background
This method is part of the CSGV-R (Compressed Segmentation Volume Random Access) framework for interactive volume rendering. It builds on the CSGV brick skipping mechanism and the Adapted Cuckoo Hashing for Voxel Labels method. The approach was introduced to address the insufficiency of brick-level skipping alone, providing a finer granularity for empty space optimization [S1].

## Sources
[S1] Random Access Segmentation Volume Compression for Interactive Volume Rendering, https://doi.org/10.1111/cgf.70116, fetched 2026-10-28
