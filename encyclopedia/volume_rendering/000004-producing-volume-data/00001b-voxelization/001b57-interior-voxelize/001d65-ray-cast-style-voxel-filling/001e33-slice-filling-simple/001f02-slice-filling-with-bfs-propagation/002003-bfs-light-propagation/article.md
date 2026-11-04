# BFS Light Propagation

Also called: Flood-fill lighting, Light propagation BFS

## Overview
BFS Light Propagation is a voxel-based lighting algorithm that propagates light levels from emitters across a padded voxel chunk using a Breadth-First Search (BFS) [S1, unofficial]. It solves the problem of efficiently illuminating non-opaque blocks in a voxel grid while attenuating light intensity with distance [S1, unofficial]. It differs from its parent, [Slice Filling with BFS Propagation](1909-slice-filling-with-bfs-propagation.md), by operating on the full 3D padded chunk rather than just the slice plane, and by explicitly modeling light attenuation per step [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A padded voxel chunk where each block has an `is_opaque()` property and a `propagated_light` value. Light sources are initialized with `propagated_light > 0` [S1, unofficial].
- Output: Updated `propagated_light` values for all blocks in the chunk.

**Data Structures:**
- `Chunk`: A 3D grid of blocks flattened to a 1D array for indexing [S1, unofficial].
- `Block`: Contains `is_opaque()` boolean and `propagated_light` integer or float [S1, unofficial].
- `Queue`: Stores tuples `(index, level)` for BFS traversal [S1, unofficial].

**Steps:**
1. Initialize an empty queue [S1, unofficial].
2. Iterate over all blocks in the padded chunk. If a block has `propagated_light > 0`, push `(block_index, block.propagated_light)` into the queue [S1, unofficial].
3. Define 6D neighbor strides for the padded chunk: `[1, -1, PADDED, -PADDED, PADDED*PADDED, -PADDED*PADDED]`, where `PADDED` is the width of the padded chunk in blocks [S1, unofficial].
4. While the queue is not empty:
   a. Pop `(idx, level)` from the queue [S1, unofficial].
   b. If `level <= 1`, skip processing this neighbor (do not propagate further) [S1, unofficial].
   c. Compute `new_level = level - 1` [S1, unofficial].
   d. For each stride `s` in the 6D strides:
      i. Compute neighbor index `ni = idx + s` [S1, unofficial].
      ii. If `ni` is out of bounds of the padded chunk, skip [S1, unofficial].
      iii. If the block at `ni` is not opaque (`!is_opaque(ni)`), update its `propagated_light` to `max(current_propagated_light, new_level)` and push `(ni, new_level)` into the queue [S1, unofficial].

**Parameters:**
- Light attenuation: 1 level per step (`new_level = level - 1`) [S1, unofficial].
- Propagation condition: Light only propagates through non-opaque blocks (`is_opaque()` returns false) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
from collections import deque

def bfs_light_propagation(chunk, padded_size):
    queue = deque()
    strides = [1, -1, padded_size, -padded_size, padded_size * padded_size, -padded_size * padded_size]
    
    # Seed queue with light sources
    for idx in range(len(chunk)):
        if chunk[idx].propagated_light > 0:
            queue.append((idx, chunk[idx].propagated_light))
            
    while queue:
        idx, level = queue.popleft()
        
        if level <= 1:
            continue
            
        new_level = level - 1
        
        for s in strides:
            ni = idx + s
            if 0 <= ni < len(chunk):
                if not chunk[ni].is_opaque():
                    if new_level > chunk[ni].propagated_light:
                        chunk[ni].propagated_light = new_level
                        queue.append((ni, new_level))
```

## Subtopics

### Voxel Light Propagation via Component-wise Max
This variant replaces simple attenuation with a component-wise maximum operation across neighbors, using a per-block falloff factor of 0.8 [S2, unofficial]. It addresses the slow convergence and exponential decay of initial Jacobi diffusion approaches, which averaged six neighbors, required $O(r^2)$ frames to converge, and limited light reach to 3-4 blocks [S2, unofficial]. By using component-wise max, reach increased to 15 blocks, and fill time reduced from 179 frames to 8 frames at an 8-block distance [S2, unofficial]. It also fades the tint from 0.02 to 0.10 luma at the end of the light volume to prevent visible ring artifacts [S2, unofficial].

## Operating characteristics
- Performance: In the component-wise max variant, fill time reduced from 179 frames to 8 frames for an 8-block reach [S2, unofficial].
- Strengths: Efficient propagation through non-opaque blocks, no leakage through opaque walls [S2, unofficial].
- Limitations: Simple attenuation (1 per step) may not accurately model real-world light falloff [S1, unofficial].

## Background
The algorithm is implemented in voxel-based rendering contexts, as demonstrated in test scenes containing voxel blocks [S1, unofficial]. The component-wise max variant was developed to address performance issues in light propagation algorithms, specifically in the Hearthlight project [S2, unofficial].

## Sources
[S1] finnbear/voxmesh, https://github.com/finnbear/voxmesh, fetched 2026-10-26 (unofficial)
[S2] Hearthlight: sun/moon shadows, voxel local lights, hand fixed, https://github.com/MeYashverma/Hearthlight/pull/1, fetched 2026-10-26 (unofficial)
