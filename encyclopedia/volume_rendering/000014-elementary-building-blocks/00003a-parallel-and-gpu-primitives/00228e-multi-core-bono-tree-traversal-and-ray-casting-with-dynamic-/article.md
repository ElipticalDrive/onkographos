# Multi-core BONO Tree Traversal and Ray Casting with Dynamic Tile Grouping

Also called: Phase I/Phase II Multi-core Scheme, Z-ordered Dynamic Tile Allocation

## Overview
This method is a multi-core parallel rendering technique for isosurface visualization that decouples data structure traversal from ray casting to optimize cache performance and load balancing [S1, unofficial]. It solves the problem of uneven computational load and poor spatial locality in parallel volume rendering by dividing the screen into contiguous regions for initial block list generation (Phase I) and then dynamically assigning Z-ordered, weight-balanced groups of image tiles to threads for ray casting (Phase II/III) [S1, unofficial]. It differs from standard static tiling approaches by using a space-filling curve (Z-order) to group tiles based on their estimated complexity (weight), ensuring that threads with heavier workloads are processed in spatially coherent batches, thereby reducing synchronization overhead and improving cache hit rates [S1, unofficial].

## How it works
The method operates in three distinct phases: Phase I (Block List Generation), Phase II (Tile Grouping), and Phase III (Dynamic Ray Casting) [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   $p$: Number of available processor cores/threads.
*   $k$: Maximum size of the block list for a pixel group (memory limit).
*   $T$: The BONO tree data structure representing the volume.
*   $R$: The set of rays corresponding to the screen pixels.
*   $S$: Screen resolution (e.g., $1024 \times 1024$).
*   $TileSize$: Dimensions of small image tiles (e.g., $8 \times 8$ or $16 \times 16$).

**Outputs:**
*   Rendered image.
*   List $LI$: A sequence of groups, where each group is a pair of indices $(first, last)$ defining a contiguous range of Z-ordered tiles [S1, unofficial].

### Data Structures
1.  **Block List ($L_i$):** For each pixel (or small group of pixels), a list of 3D data blocks from the BONO tree that the ray intersects [S1, unofficial].
    *   Constraint: $|L_i| \le k$.
2.  **Tile Grid:** The screen is partitioned into $N$ tiles of size $TileSize \times TileSize$ [S1, unofficial].
3.  **Z-Order Map:** An array mapping tile index $i$ to its Z-order position [G].
4.  **Weight Array ($W_{tile}$):** An integer array where $W_{tile}[j]$ is the number of non-empty block lists within tile $j$ [S1, unofficial].
5.  **Group List ($LI$):** A list of tuples $((first\_tile\_idx, last\_tile\_idx), group\_id)$ [S1, unofficial].

### Numbered Steps

#### Phase I: Block List Generation
1.  Divide the screen into $p$ almost equal contiguous horizontal (or vertical) regions, one per thread [S1, unofficial].
2.  For each thread $t$ responsible for region $R_t$:
    a.  Initialize empty block lists for all pixels in $R_t$ [G].
    b.  Traverse the BONO tree $T$ [S1, unofficial].
    c.  **Traversal Rule:** For a node $N$, traverse its children in front-to-back order **only if**:
        i.  The projection of $N$'s minimum bounding box intersects $R_t$ [S1, unofficial].
        ii. At least one pixel in $R_t$ has a block list size $< k$ [S1, unofficial].
    d.  If the node is a leaf (data block), add the block to the lists of all intersecting pixels in $R_t$, provided the list size $< k$ [S1, unofficial].
    e.  **Note:** Due to the limit $k$, only a fraction (no more than 10%) of total BONO tree nodes are accessed [S1, unofficial].

#### Phase II: Dynamic Tile Grouping
1.  **Partitioning:** Partition the entire screen into small tiles of size $TileSize \times TileSize$ [S1, unofficial].
2.  **Z-Ordering:** Order these tiles using a Z-order (Morton code) space-filling curve. This ensures spatial locality [S1, unofficial].
3.  **Weighting:**
    a.  For each tile $j$, compute weight $w_j = \sum_{p \in Tile_j} (1 \text{ if } L_p \neq \emptyset \text{ else } 0)$ [S1, unofficial].
    b.  Compute total weight $W = \sum_{all \ tiles} w_j$ [S1, unofficial].
4.  **Grouping Algorithm:**
    a.  Initialize an empty list $LI$.
    b.  Let $remaining\_weight = W$.
    c.  Let $current\_tile\_index = 0$ (in Z-order) [G].
    d.  While $remaining\_weight > 0$:
        i.  Target weight for this iteration: $Target = remaining\_weight / 2$ [S1, unofficial].
        ii. Divide $Target$ into $p$ equal sub-targets: $SubTarget = Target / p$ [S1, unofficial].
        iii. For each group $g$ from $0$ to $p-1$:
            1.  Start with $start\_tile = current\_tile\_index$.
            2.  Accumulate weights from Z-ordered tiles starting at $start\_tile$ until the sum reaches or exceeds $SubTarget$ [S1, unofficial].
            3.  Let $end\_tile$ be the index of the last tile included.
            4.  Append $(start\_tile, end\_tile)$ to $LI$ [S1, unofficial].
            5.  Update $current\_tile\_index = end\_tile + 1$ [G].
        iv. Update $remaining\_weight = remaining\_weight / 2$ [S1, unofficial].
    e.  **Termination:** The process repeats logarithmically relative to screen size. The first iterations create large groups (coarse-grained), and subsequent iterations create smaller groups (fine-grained) [S1, unofficial].

#### Phase III: Dynamic Ray Casting
1.  Initialize a shared queue/pointer to the list $LI$ [G].
2.  For each thread $t$:
    a.  **Loop:**
        i.  Atomically fetch the next available group $(first, last)$ from $LI$. If none remain, terminate [G].
        ii. For each tile index $j$ from $first$ to $last$:
            1.  For each pixel $p$ in tile $j$:
                1.  Retrieve block list $L_p$ [S1, unofficial].
                2.  If $L_p$ is empty, skip [G].
                3.  Shoot ray through blocks in $L_p$ (front-to-back) [S1, unofficial].
                4.  **Shading:**
                    *   If ray intersects isosurface:
                        *   Calculate intersection position by solving the trilinear interpolation equation [S1, unofficial].
                        *   Compute normal using forward difference gradient [S1, unofficial].
                        *   Apply diffuse shade model [S1, unofficial].
                    *   Else if ray exits volume without intersection:
                        *   Mark pixel as background [G].
        iii. If any pixels in the processed tiles have block lists of size $k$ but no intersection was found (unfinished pixels), handle them in a separate pass (Step iv of the source description) [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import threading

def multi_core_bono_rendering(p, k, bono_tree, screen_width, screen_height, tile_size):
    # Phase I: Block List Generation
    region_height = math.ceil(screen_height / p)
    block_lists = {}

    for t in range(p):
        y_start = t * region_height
        y_end = min((t + 1) * region_height, screen_height)
        local_lists = {(x, y): [] for y in range(y_start, y_end) for x in range(screen_width)}
        traverse_bono(bono_tree, local_lists, y_start, y_end, screen_width, k)
        block_lists.update(local_lists)

    # Phase II: Dynamic Tile Grouping
    tiles = [(x, y) for y in range(0, screen_height, tile_size) for x in range(0, screen_width, tile_size)]
    z_ordered_tiles = sorted(tiles, key=lambda t: z_order(t[0], t[1], screen_width))
    
    weights = []
    for tile in z_ordered_tiles:
        tx, ty = tile
        w = sum(1 for dy in range(tile_size) for dx in range(tile_size) 
                if (tx+dx, ty+dy) in block_lists and len(block_lists[(tx+dx, ty+dy)]) > 0)
        weights.append(w)
    
    total_weight = sum(weights)
    LI = []
    current_idx = 0
    remaining_weight = total_weight
    
    while remaining_weight > 0:
        target_weight = remaining_weight / 2
        sub_target = target_weight / p
        for g in range(p):
            if current_idx >= len(z_ordered_tiles): break
            start_idx = current_idx
            current_tile_weight = 0
            while current_idx < len(z_ordered_tiles) and current_tile_weight < sub_target:
                current_tile_weight += weights[current_idx]
                current_idx += 1
            end_idx = current_idx - 1
            if start_idx <= end_idx: LI.append((start_idx, end_idx))
        remaining_weight /= 2

    # Phase III: Dynamic Ray Casting
    li_queue = list(LI)
    li_ptr = 0
    
    def phase_iii_thread():
        nonlocal li_ptr
        while True:
            group = None
            if li_ptr < len(li_queue):
                group = li_queue[li_ptr]
                li_ptr += 1
            if group is None: break
            
            start_idx, end_idx = group
            for idx in range(start_idx, end_idx + 1):
                tile = z_ordered_tiles[idx]
                tx, ty = tile
                for dy in range(tile_size):
                    for dx in range(tile_size):
                        px, py = tx + dx, ty + dy
                        if (px, py) not in block_lists: continue
                        ray_list = block_lists[(px, py)]
                        if not ray_list: continue
                        color = cast_ray(ray_list, px, py)
                        write_pixel(px, py, color)
            # Handle unfinished pixels (size k, no intersection) [S1, unofficial]
            handle_unfinished_pixels(tx, ty, tile_size)

    threads = []
    for _ in range(p):
        t = threading.Thread(target=phase_iii_thread)
        t.start()
        threads.append(t)
    for t in threads: t.join()

def traverse_bono(node, lists, y_start, y_end, screen_width, k):
    if node.is_leaf:
        for y in range(y_start, y_end):
            for x in range(screen_width):
                if len(lists[(x, y)]) < k: lists[(x, y)].append(node.block)
        return
    if not node.bbox_intersects_region(y_start, y_end): return
    if not any(len(lists[(x, y)]) < k for y in range(y_start, y_end) for x in range(screen_width)): return
    for child in node.children_front_to_back:
        traverse_bono(child, lists, y_start, y_end, screen_width, k)

def z_order(x, y, width):
    z = 0
    bits = int(math.log2(width)) + 1
    for i in range(bits):
        z |= ((x & (1 << i)) << i) | ((y & (1 << i)) << (i - 1))
    return z

def ray_intersects_block(px, py, block): return True # [G]
def solve_trilinear_intersection(px, py, block): return (0.5, 0.5, 0.5) # [G]
def compute_forward_difference_gradient(pos, block): return (0.0, 1.0, 0.0) # [G]
def apply_diffuse_shading(normal): return (0.8, 0.8, 0.8) # [G]
def write_pixel(px, py, color): pass # [G]
def handle_unfinished_pixels(tx, ty, tile_size): pass # [S1, unofficial]
```

### Parameters
*   $p$: Number of cores/threads. Typical values tested: 1, 2, 4, 8 [S1, unofficial].
*   $k$: Upper bound on block list size. Limits memory usage and traversal depth. Exact value not specified, but it is critical for limiting BONO node access to <10% [S1, unofficial].
*   $TileSize$: Small image tile dimensions. Typical values: $8 \times 8$ or $16 \times 16$ [S1, unofficial].

### Edge Cases and Pitfalls
*   **Unfinished Pixels:** Pixels whose block lists reach size $k$ but contain no intersecting voxels must be handled separately. These pixels will not be processed in the main ray casting loop if the logic assumes intersection or empty lists. A post-processing step or specific flag is required [S1, unofficial].
*   **Boundary Conditions:** The screen division in Phase I must handle non-divisible screen heights (e.g., $1024 / 8 = 128$, but if $p=3$, regions must be "almost equal") [S1, unofficial].

## Operating characteristics
*   **Time Complexity:**
    *   Phase I: Proportional to the number of pixels in the region and the number of BONO nodes accessed. Node access is limited to <10% of total due to $k$ [S1, unofficial].
    *   Phase II: Grouping overhead is small, approximately 2-3 milliseconds [S1, unofficial]. The grouping process requires at most a logarithmic number of iterations relative to screen size [S1, unofficial].
    *   Phase III: Proportional to the total number of pixels and the average length of block lists.
*   **Memory Complexity:**
    *   Dominated by the storage of block lists. Each pixel stores up to $k$ block references.
    *   BONO tree overhead varies by dataset (e.g., 10.53% to 25.05% of data size for tested datasets) [S1, unofficial].
*   **Measured Performance:**
    *   Hardware: Clovertown platform (2 Quad-Core Intel Xeon 5320, 1.86 GHz), 8GB RAM, 8MB L2 cache [S1, unofficial].
    *   Resolution: $1024 \times 1024$ [S1, unofficial].
    *   View Angles: Zenith $\phi \in \{15^\circ, 45^\circ, 75^\circ\}$, Azimuth $\theta \in \{22.5^\circ, 45^\circ\}$ [S1, unofficial].
    *   Scalability: Tested on 1, 2, 4, 8 cores [S1, unofficial].
*   **Strengths:**
    *   Better load balancing than static tiling due to weight-based grouping.
    *   High cache performance due to Z-order spatial locality.
    *   Minimal synchronization overhead due to coarse-to-fine grain task allocation [S1, unofficial].
*   **Limitations:**
    *   Requires preprocessing (Phase II) which adds latency.
    *   Complexity of implementation compared to simple static tiling.

## Background
The method was developed for interactive high-resolution isosurface ray tracing on multi-core processors [S1, unofficial]. The work addresses the limitations of standard octree ray casting with static tile allocation, which suffers from load imbalance and poor cache utilization [S1, unofficial]. The BONO tree is used as the underlying data structure for efficient spatial indexing [S1, unofficial].

## Sources
[S1] Interactive High Resolution Isosurface Ray Tracing on Multi-Core Processors, https://www.umiacs.umd.edu/~joseph/isosurface-paper.pdf, fetched 2026-10-26 (unofficial)
