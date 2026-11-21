# Tile-Local Bin Construction

Also called: Gaussian-tile binning with atomic count/scatter, Tile-bin-major pair stream construction

## Overview
Tile-Local Bin Construction assigns Gaussian-tile intersection entries to coarse depth bins within each tile and constructs a flattened tile-bin-major pair stream using direct atomic operations for count and scatter [S1]. It solves the problem of organizing splats for cache-efficient compositing by grouping them into depth-ordered buckets, serving as the foundational data structure for subsequent rasterization passes [S1]. The method uses direct atomics for the count/scatter phase as its default configuration, with alternative aggregation variants producing identical bin-offset arrays and Gaussian-ID streams [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of baseline entries $E$. Each entry $e \in E$ contains:
    *   $t(e)$: Tile identifier.
    *   $g(e)$: Gaussian flat identifier.
    *   $s(e)$: Original baseline sort key (encoding baseline depth order).
    *   $z(e)$: Depth value.
    *   Clipped depth range $[z_{min}, z_{max}]$ [S1].
    *   Number of bins $K$ [S1].
    *   `num_tiles`: Total number of tiles to process [G].
*   **Output:**
    *   `binOffsets[t, 0...K]`: Array defining the concatenated interval for all bins in tile $t$. The segment for bin $b$ is $[binOffsets[t,b], binOffsets[t,b+1])$ [S1].
    *   $P$: Tile-bin-major pair stream storing tuples $(s(e), g(e))$ [S1].
    *   Optional sidecars: `flatIds` and Packed-GW geometry attributes [S1].

**Data Structures**
*   `count[t, b]`: Integer array initialized to zero, tracking the number of entries assigned to bin $b$ in tile $t$ [S1].
*   `cursor[t, b]`: Write pointer initialized from `binOffsets`, tracking the next available write position within bin $b$ of tile $t$ [S1].
*   $P$: Array of pairs. The stream is ordered such that all entries for tile $t$, bin $0$ precede entries for tile $t$, bin $1$, etc. [S1].

**Parameters**
*   $K = 64$: Number of depth bins (default) [S1].
*   $\epsilon = 10^{-4}$: Epsilon for log-space stability [S1].
*   $z'_{min} = \max(z_{min}, \epsilon)$ [S1].
*   $z'_{max} = \max(z_{max}, z'_{min} + 10^{-6})$ [S1].

**Algorithm**
1.  Initialize `count[t, b] = 0` for all tiles $t$ and bins $b \in \{0, \dots, K-1\}$ [S1].
2.  For all entries $e \in E$ in parallel:
    a. Compute the bin index $b(e)$ using the log-depth bin mapping [S1].
    b. Atomically increment `count[t(e), b(e)]` [S1].
3.  Perform an exclusive scan on the `count` array to compute `binOffsets[t, b]` [S1].
4.  Initialize `cursor[t, b]` to `binOffsets[t, b]` for all $t, b$ [S1].
5.  For all entries $e \in E$ in parallel:
    a. Retrieve bin $b = b(e)$ [S1].
    b. Retrieve write position $p = \text{atomic increment of cursor}[t(e), b]$ [S1].
    c. Write $P[p] = (s(e), g(e))$ [S1].
    d. If flat-id sidecar is enabled, write `flatIds[p] = g(e)` [S1].
    e. If Packed-GW geometry sidecar is enabled, materialize selected geometry attributes at index $p$ [S1].

**Log-Depth Bin Mapping Formula**
The bin index $b$ is computed as follows [S1]:
$$ u = \text{clamp}\left(\frac{\ln(\max(z, \epsilon)) - \ln(z'_{\min})}{\ln(z'_{\max}) - \ln(z'_{\min})}, 0, 1 - 10^{-6}\right) $$
$$ b = \lfloor K \cdot u \rfloor $$
$$ b = \text{clamp}(b, 0, K-1) $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def tile_local_bin_construction(entries, z_min, z_max, K, num_tiles, enable_flat_ids=False, enable_packed_gw=False):
    epsilon = 1e-4
    z_prime_min = max(z_min, epsilon)
    z_prime_max = max(z_max, z_prime_min + 1e-6)

    def log_depth_bin(z):
        z_safe = max(z, epsilon)
        u = (math.log(z_safe) - math.log(z_prime_min)) / (math.log(z_prime_max) - math.log(z_prime_min))
        u = max(0.0, min(u, 1.0 - 1e-6))
        b = int(math.floor(K * u))
        return max(0, min(b, K - 1))

    # 1. Initialize counts
    count = [[0] * K for _ in range(num_tiles)]

    # 2. Parallel Count Pass
    for e in entries:
        b = log_depth_bin(e.z)
        count[e.t][b] += 1

    # 3. Exclusive Scan for Offsets
    binOffsets = [[0] * (K + 1) for _ in range(num_tiles)]
    for t in range(num_tiles):
        current_offset = 0
        for b in range(K):
            binOffsets[t][b] = current_offset
            current_offset += count[t][b]
        binOffsets[t][K] = current_offset

    # 4. Initialize Cursors
    cursor = [row[:K] for row in binOffsets]

    # 5. Parallel Scatter Pass
    total_entries = sum(binOffsets[t][K] for t in range(num_tiles))
    P = [None] * total_entries
    flatIds = [None] * total_entries if enable_flat_ids else None
    gw_attrs = [None] * total_entries if enable_packed_gw else None

    for e in entries:
        b = log_depth_bin(e.z)
        p = cursor[e.t][b]
        cursor[e.t][b] += 1
        P[p] = (e.s, e.g)
        if enable_flat_ids:
            flatIds[p] = e.g
        if enable_packed_gw:
            gw_attrs[p] = materialize_geometry(e) # [G] Standard geometry materialization

    return binOffsets, P, flatIds, gw_attrs
```

**Edge Cases and Pitfalls**
*   **Depth Range Estimation:** The $z_{min}$ and $z_{max}$ are derived from percentile selection of sampled candidates using exact order statistics, not histogram approximation [S1]. Incorrect range estimation leads to bin overflow or underflow [G].
*   **Log-Space Stability:** The use of $\epsilon$ and clamping prevents $\ln(0)$ and ensures numerical stability for depths near zero [S1].
*   **Atomic Contention:** Direct atomics are used for count and scatter [S1].
*   **Order Within Bin:** The atomic scatter operation does not guarantee any specific order within a bin [G].

## Subtopics

### [Selective Exact Repair](10281-selective-exact-repair.md)
A post-binning sorting step that restores exact baseline depth order for specific segments of the tile-bin-major pair stream to ensure correct rasterization, while leaving other segments in their tile-local bin order. It balances correctness and speed by repairing only segments likely to cause artifacts or bottlenecks.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Performance:** On an RTX 4090, the default direct atomic implementation takes approximately 2.86811 ms [S1]. This is faster than warp-private histograms (3.32800 ms), block-local hash aggregation (2.92911 ms), and two-stage aggregation (2.85863 ms), though warp-key grouping is slightly faster at 2.85796 ms [S1].
*   **Memory:** Requires storage for `count`, `binOffsets`, `cursor`, and the output pair stream $P$. The size of $P$ is equal to the number of entries $|E|$ [S1].
*   **Strengths:** Simple implementation, good performance on modern GPUs with efficient atomic units, robust log-depth binning handles wide depth ranges well [G].
*   **Limitations:** Atomic contention can become a bottleneck in highly clustered scenes [G].

## Background
Tile-Local Bin Construction is part of the TileGS rasterization pipeline, introduced in the paper "TileGS: Tile-Local Depth Binning for Gaussian Splatting Rasterization" [S1]. It builds on the concept of tile binning for splatting, which groups depth-sorted splats into tile buckets to improve cache locality during compositing [Parent]. The method uses direct atomic operations for count/scatter as the default construction strategy, chosen for its performance characteristics on contemporary hardware [S1].

## Sources
[S1] TileGS: Tile-Local Depth Binning for Gaussian Splatting Rasterization, https://doi.org/10.2312/pg.20261013, fetched 2026-10-29
