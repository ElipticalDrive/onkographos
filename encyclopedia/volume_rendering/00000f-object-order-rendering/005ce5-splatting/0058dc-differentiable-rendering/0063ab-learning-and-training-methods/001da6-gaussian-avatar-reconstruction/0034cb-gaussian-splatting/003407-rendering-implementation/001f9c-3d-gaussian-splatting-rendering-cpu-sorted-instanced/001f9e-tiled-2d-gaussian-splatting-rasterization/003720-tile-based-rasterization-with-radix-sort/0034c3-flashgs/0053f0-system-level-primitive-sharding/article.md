# System Level Primitive Sharding

Also called: BlitzGS sharding, Distributed rendering, Index parity sharding, Primitive Sharding

## Overview
System Level Primitive Sharding is a distributed rendering and training strategy for 3D Gaussian Splatting that partitions a global Gaussian model across $M$ GPUs to handle city-scale scenes with hundreds of millions of primitives [S1, unofficial]. It ensures that both storage and per-step compute scale linearly with the GPU count by assigning Gaussians via index parity and distributing rendered tiles across the GPU cluster [S1, unofficial]. Unlike spatial block partitioning, which incurs boundary waste, duplicate camera assignments, and merge steps, primitive sharding assigns every camera exactly once and eliminates the need for a final merge [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A global set of 3D Gaussians $G$ with associated parameters (position, covariance, opacity, spherical harmonics coefficients), optimizer states, camera parameters, and the number of GPUs $M$ [S1, unofficial].
*   **Output:** Rendered images for assigned tiles, updated Gaussian parameters, and distributed optimizer states [S1, unofficial].

**Data Structures:**
*   **Shard $G^{(m)}$:** A subset of the global Gaussian set $G$ assigned to GPU rank $m$ ($0 \le m < M$), containing approximately $|G|/M$ Gaussians [S1, unofficial].
*   **Optimizer State:** Per-Gaussian optimization buffers (e.g., momentum, variance) stored locally on the owning GPU [S1, unofficial].
*   **Tile Assignment:** A mapping of image tiles to GPU ranks, determined by a cost-aware partitioning algorithm to balance rasterization load [S1, unofficial].

**Steps:**
1.  **Initial Sharding:** Partition the global Gaussian set $G$ into $M$ shards $G^{(m)}$ based on index parity ($i \bmod M$) [S1, unofficial].
2.  **Tile Partitioning:** Divide the rendered image into tiles and assign each tile to a specific GPU rank using cost-aware partitioning to even out rasterization load [S1, unofficial].
3.  **Projection:** Each GPU projects only the Gaussians in its local shard $G^{(m)}$ into camera space [S1, unofficial].
4.  **All-to-All Communication:** Perform a single all-to-all communication step to route every projected Gaussian to the GPUs owning the tiles it covers [S1, unofficial].
5.  **Compositing:** Each GPU composites its assigned tiles using the received Gaussians [S1, unofficial].
6.  **Rebalancing:** Apply index-parity redistribution to rebalance shard sizes if per-GPU skew exceeds a threshold [S1, unofficial].

**Formulas:**
*   **Shard Size:** $|G^{(m)}| \approx \frac{|G|}{M}$ [S1, unofficial].
*   **Memory Scaling:** Per-rank peak memory decreases as $M$ increases. For the Rubble scene, memory drops from 17.2 GB at $M=1$ to 2.9 GB at $M=8$ [S1, unofficial].

**Parameters:**
*   $M$: The number of GPUs across which the model is sharded [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def system_level_primitive_sharding(G, cameras, M, rebalance_threshold):
    # Step 1: Initial Sharding by Index Parity
    shards = [[] for _ in range(M)]
    optimizer_states = [[] for _ in range(M)]
    for i, gaussian in enumerate(G):
        m = i % M
        shards[m].append(gaussian)
        optimizer_states[m].append(gaussian.optimizer_state)

    # Step 2: Tile Partitioning
    # Cost-aware tile partitioning to balance rasterization load [S1, unofficial]
    # The exact algorithm is not specified in the sources [G]
    tile_to_gpu = {t: t % M for t in range(len(cameras[0].tiles))}

    # Step 3 & 4: Projection and All-to-All Communication
    local_gaussians = [list(shards[m]) for m in range(M)]
    for m in range(M):
        projected = [project_3d_to_2d(g, cameras[m]) for g in shards[m]]
        for g in projected:
            covered_tiles = get_tile_indices(g)
            for t in covered_tiles:
                target = tile_to_gpu[t]
                if target != m:
                    send_to_rank(target, g)
                else:
                    local_gaussians[m].append(g)
    for m in range(M):
        local_gaussians[m] += receive_from_rank(m)

    # Step 5: Compositing
    for m in range(M):
        assigned_tiles = [t for t, gpu in tile_to_gpu.items() if gpu == m]
        render_tiles(assigned_tiles, local_gaussians[m], cameras[m])

    # Step 6: Rebalancing
    if compute_skew(shards) > rebalance_threshold:
        shards = redistribute_by_index_parity(shards)
        optimizer_states = redistribute_optimizer_states(optimizer_states)

    return shards, optimizer_states

def project_3d_to_2d(gaussian, camera):
    return gaussian.project(camera) # [G] Standard projection

def get_tile_indices(gaussian):
    return [t for t in range(num_tiles) if gaussian.overlaps_tile(t)] # [G] Standard coverage

def send_to_rank(target_rank, gaussian):
    pass # [G] Standard MPI all-to-all

def receive_from_rank(source_rank):
    return [] # [G] Standard MPI all-to-all

def render_tiles(tiles, gaussians, camera):
    pass # [G] Standard splatting rasterization

def compute_skew(shards):
    sizes = [len(s) for s in shards]
    return (max(sizes) - min(sizes)) / (sum(sizes) / len(sizes)) # [G] Standard skew

def redistribute_by_index_parity(shards):
    return [ [] for _ in range(M) ] # [G] Reassign by index modulo M

def redistribute_optimizer_states(optimizer_states):
    return [ [] for _ in range(M) ] # [G] Move states with Gaussians
```

**Edge Cases and Pitfalls:**
*   **Skew:** Non-uniform Gaussian distribution may cause imbalanced shards; index-parity redistribution mitigates this [S1, unofficial].
*   **Communication Overhead:** The all-to-all step can become a bottleneck if the number of Gaussians or tiles is very large, though sources indicate manageable scaling [S1, unofficial].
*   **Boundary Issues:** Unlike spatial partitioning, there are no spatial boundaries to handle, eliminating duplicate camera assignments and merge steps [S1, unofficial].
*   **Fixed Overhead:** Per-rank data loading, optimizer, and bookkeeping remain constant at 15–20 ms and dominate shrinking iterations [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Per-iteration cost scales near-linearly with GPU count for compute-heavy stages like the backward pass. On the Residence scene with A800 80 GB GPUs, total per-iteration cost drops from 186.9 ms (1 GPU) to 68.6 ms (8 GPUs) [S1, unofficial].
*   **Memory Complexity:** Per-rank peak memory scales inversely with $M$. On the Rubble scene, it drops from 17.2 GB (1 GPU) to 2.9 GB (8 GPUs) [S1, unofficial]. The binding limit is the aggregate capacity across all GPUs [S1, unofficial].
*   **Strengths:** Eliminates boundary waste and merge steps inherent in spatial block partitioning. Assigns every camera exactly once, preventing duplicate processing. Scales storage and compute linearly with GPU count [S1, unofficial].
*   **Limitations:** Requires efficient all-to-all communication. Residual overhead from data loading and optimizer bookkeeping becomes significant as iteration time shrinks [S1, unofficial].
*   **When to Use:** Suitable for city-scale scenes with hundreds of millions of primitives that exceed single GPU memory [S1, unofficial].

## Background
System Level Primitive Sharding, also known as BlitzGS sharding, was introduced in "BlitzGS: City-Scale Gaussian Splatting at Lightning Speed" [S1, unofficial]. It addresses spatial block partitioning limitations, including cross-block visibility waste, imbalanced shards, and the need for a final merge step [S1, unofficial]. The method is inspired by prior work on distributed rendering [S1, unofficial]. It is used in the context of 3D Gaussian Splatting for large-scale scene reconstruction and rendering [S1, unofficial].

## Sources
[S1] BlitzGS: City-Scale Gaussian Splatting at Lightning Speed, https://doi.org/10.48550/arXiv.2605.13794, fetched 2026-10-29 (unofficial)
