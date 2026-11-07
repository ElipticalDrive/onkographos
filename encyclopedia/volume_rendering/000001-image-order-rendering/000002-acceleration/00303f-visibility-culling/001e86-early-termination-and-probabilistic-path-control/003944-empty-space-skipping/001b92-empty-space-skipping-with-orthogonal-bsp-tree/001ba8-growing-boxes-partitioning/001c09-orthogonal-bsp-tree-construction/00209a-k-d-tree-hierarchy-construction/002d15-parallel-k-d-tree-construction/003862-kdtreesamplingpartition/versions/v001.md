# KDTreeSamplingPartition

Also called: DIY2 block sampling, Distributed sample distribution, KDTree Sampling Partitioning

## Overview
KDTreeSamplingPartition is a domain decomposition algorithm for distributing volume rendering samples across multiple compute blocks in a distributed-memory system [S1, unofficial]. It builds a k-d tree structure over spatial data points by recursively splitting blocks along coordinate dimensions using a sampling-based approach to determine split positions [S1, unofficial]. It sets itself apart from static partitioning methods by dynamically determining split planes via random sampling and managing neighbor links via a reduce proxy to handle wrap-around or boundary conditions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of spatial points distributed across processing blocks, the spatial dimension `dim_`, the number of samples to draw `samples_`, and a `Partners` object managing communication rounds, dimensions, and wrap-around settings.
*   **Output:** A partitioned set of points where each block holds points within its assigned k-d tree leaf region, and updated neighbor links for subsequent communication or rendering steps.

**Data Structures:**
*   `Block`: Contains a member `points_` which is a `std::vector<Point>` [S1, unofficial].
*   `Point`: A structure containing coordinates for the spatial dimension `dim_` [S1, unofficial].
*   `Partners`: Manages the state of the partitioning process.
    *   `rounds()`: Total number of splitting rounds.
    *   `dim(round)`: The splitting dimension for a given round.
    *   `swap_round(round)`: Boolean indicating if a swap occurs in this round.
    *   `sub_round(round)`: Sub-round index within a main round.
    *   `wrap`: Boolean enabling domain wrap-around.
    *   `domain`: The global domain bounds.
*   `SRP` (Splitting Reduce Proxy): Manages communication links.
    *   `in_link()`: List of incoming links from neighbors.
    *   `out_link()`: List of outgoing links to neighbors.
    *   `gid()`: Global ID of the current block.
    *   `round()`: Current round index.

**Numbered Steps:**
1.  **Initialization:** Assign an equal number of samples to each processing block [S1, unofficial]. Update inter-block links to handle wrap-around or boundary conditions across the domain [S1, unofficial].
2.  **Iterative Splitting:** For each round `r` from 0 to `partners.rounds() - 1`:
    a. Determine the current splitting dimension `dim`:
       $$ \text{dim} = \begin{cases} \text{partners.dim}(r) & \text{if } r < \text{partners.rounds}() \\ \text{partners.dim}(r - 1) & \text{otherwise} \end{cases} $$
       [S1, unofficial]
    b. If `r == partners.rounds() - 1` (final round), call `update_links` to finalize links [S1, unofficial].
    c. If `partners.swap_round(r)` is true (and not a link round):
       i. Receive samples from neighbor 0: `receive_samples(b, srp, samples)` [S1, unofficial].
       ii. Enqueue exchange: `enqueue_exchange(b, srp, dim, samples)` [S1, unofficial].
    d. If `partners.sub_round(r) == 0`:
       i. If `r > 0`, update links for the previous dimension `prev_dim`:
          $$ \text{prev\_dim} = (\text{dim} - 1 < 0) ? \text{dim\_} + \text{dim} - 1 : \text{dim} - 1 $$
          [S1, unofficial]
       ii. Compute local samples: `compute_local_samples(b, srp, dim)` [S1, unofficial].
    e. Else (final sample aggregation):
       i. Add local samples: `add_samples(b, srp, samples)` [S1, unofficial].
       ii. If `samples.size() != 1`, find the median using `std::nth_element` on the `samples` vector, swap the median to index 0, resize the vector to 1, and forward via `forward_samples(b, srp, samples)` [S1, unofficial].

**Helper Functions:**
*   `compute_local_samples(b, srp, dim)`:
    1.  Let `n = min(points_size, samples_)` [S1, unofficial].
    2.  Draw `n` random samples from the block's points along dimension `dim`.
    3.  For each sample, pick a random index `idx = rand() % points_size` and extract the coordinate `coord = (b->*points_)[idx][dim]` [S1, unofficial].
    4.  Enqueue the vector of coordinates to `srp.out_link().target(0)` [S1, unofficial].
*   `add_samples(b, srp, samples)`:
    1.  Iterate over `srp.in_link().size()` neighbors.
    2.  For each neighbor `nbr_gid`, dequeue samples and append them to the local `samples` vector [S1, unofficial].
*   `enqueue_exchange(b, srp, dim, samples)`:
    1.  If `srp.out_link().size() == 0`, return [S1, unofficial].
    2.  Pick `split = samples[0]` [S1, unofficial].
    3.  Partition local points into `out_points[0]` (where `x[dim] < split`) and `out_points[1]` (where `x[dim] >= split`) [S1, unofficial].
    4.  Swap the appropriate subset back into `(b->*points_)` based on which output link target matches `srp.gid()` [S1, unofficial].
    5.  Enqueue other subsets to respective targets [S1, unofficial].
    6.  Update `link->core().max[dim]` if `pos == 0`, else `link->core().min[dim]` [S1, unofficial].
*   `dequeue_exchange(b, srp, dim)`:
    1.  Iterate over `srp.in_link().size()` neighbors.
    2.  For each neighbor `nbr_gid` (skip if `nbr_gid == srp.gid()`), dequeue `in_points` [S1, unofficial].
    3.  Validate each point's coordinate at `dim` is within `[link->core().min[dim], link->core().max[dim]]`. Throw `std::runtime_error` if not [S1, unofficial].
    4.  Push valid points into `(b->*points_)` [S1, unofficial].
*   `update_links(b, srp, dim, prev_dim, swap_rounds, wrap, domain)`:
    1.  Build a `link_map` from existing link targets to indices [S1, unofficial].
    2.  Dequeue split values and directions from incoming links, reversing direction vectors (`dir[j] = -dir[j]`) [S1, unofficial].
    3.  Map dequeued data to `splits[k]` [S1, unofficial].
    4.  Construct `new_link` with `dim_` neighbors [S1, unofficial].
    5.  Determine `lower = !(gid & (1 << (rounds - 1 - round)))` [S1, unofficial].
    6.  For each existing link entry:
        a. If `dir[dim] != 0`:
           i. If split condition `(dir[dim] < 0 && lower) || (dir[dim] > 0 && !lower)` holds, compute neighbor GID via `divide_gid`, update bounds via `update_neighbor_bounds`, add wrap if enabled, and add to `new_link` [S1, unofficial].
        b. If `dir[dim] == 0` (non-aligned side):
           i. Iterate `j=0,1`, compute neighbor GID, update bounds, check intersection via `intersects`, and add if intersecting [S1, unofficial].
    7.  Add dual block neighbor, update its bounds, and set direction (right if lower, left if not) [S1, unofficial].
    8.  Swap `link` with `new_link` [S1, unofficial].
*   `divide_gid(gid, lower, rounds, round)`:
    1.  If `lower`, clear bit `(rounds - 1 - round)` in `gid` (`gid &= ~(1 << (rounds - 1 - round))`); else set it (`gid |= (1 << (rounds - 1 - round))`) [S1, unofficial].
    2.  Return `gid` [S1, unofficial].
*   `split_to_neighbors(link, bounds)`:
    1.  Find split value via `find_split(link->core(), link->bounds())` [S1, unofficial].
    2.  Enqueue split and direction to each neighbor in `link` [S1, unofficial].
*   `find_split(changed, original)`:
    1.  Iterate `i` from 0 to `dim_-1` [S1, unofficial].
    2.  If `changed.min[i] != original.min[i]`, return `changed.min[i]` [S1, unofficial].
    3.  If `changed.max[i] != original.max[i]`, return `changed.max[i]` [S1, unofficial].
    4.  Assert false if no difference found [S1, unofficial].
*   `intersects(x, y, wrap, domain, dim)`:
    1.  If `wrap` is true, return true if `x.min[dim] == domain.min[dim] && y.max[dim] == domain.max[dim]` OR `y.min[dim] == domain.min[dim] && x.max[dim] == domain.max[dim]` [S1, unofficial].
    2.  Otherwise, return `x.min[dim] <= y.max[dim] && y.min[dim] <= x.max[dim]` [S1, unofficial].
*   `update_neighbor_bounds(bounds, lower, split, dim)`:
    1.  If `lower`, set `bounds.max[dim] = split`; else set `bounds.min[dim] = split` [S1, unofficial].
*   `forward_samples(b, srp, samples)`:
    1.  Enqueue `samples` to all targets in `srp.out_link()` [S1, unofficial].
*   `receive_samples(b, srp, samples)`:
    1.  Dequeue samples from `srp.in_link().target(0).gid` into `samples` [S1, unofficial].

**Parameters:**
*   `samples_`: Number of samples to draw. Typical value: Not given in the sources.
*   `dim_`: Spatial dimension. Typical value: 3 for volume rendering [S1, unofficial].

**Edge Cases and Pitfalls:**
*   The current implementation does not perform a perfect subsample; it should randomly select samples so the total equals `samples * nblocks` [S1, unofficial].
*   Random sampling relies on the legacy C `rand()` function, which must be seeded externally before use [S1, unofficial].
*   Validation of point coordinates during `dequeue_exchange` throws a `std::runtime_error` if a point is outside the expected bounds [S1, unofficial].

## Operating characteristics
Time and memory complexity: Not given in the sources.
Measured performance: Not given in the sources.
Scaling: Not given in the sources.
Strengths and limitations: The method dynamically adapts to data distribution via sampling, but relies on legacy random number generation and may not achieve perfect subsampling [S1, unofficial].
When to use it and when not: Use in distributed-memory volume rendering systems requiring dynamic domain decomposition [S1, unofficial].

## Background
Origin: The algorithm is part of the Rover project, developed at Lawrence Livermore National Laboratory [S1, unofficial].
History and development: It is used within the Rover framework for energy engine rendering [S1, unofficial].
Where it is used: Distributed volume rendering [S1, unofficial].

## Sources
[S1] llnl/rover, https://github.com/llnl/rover, fetched 2026-10-26 (unofficial)
