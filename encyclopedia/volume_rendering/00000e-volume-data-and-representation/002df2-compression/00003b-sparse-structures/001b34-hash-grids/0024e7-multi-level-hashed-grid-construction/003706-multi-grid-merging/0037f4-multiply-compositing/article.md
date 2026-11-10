# Multiply Compositing
## Overview
Multiply Compositing merges two sparse hash grids by returning a new grid whose active voxels are the union of the inputs, with overlapping values multiplied together [S1, unofficial]. It differs from [Replace Compositing](7905-multi-grid-merging.md) by preserving both contributions in overlapping regions rather than letting one overwrite the other [S1, unofficial].
## How it works
**Inputs:** Two sparse hash grids, `a` and `b`, containing active voxels with associated values [S1, unofficial].
**Outputs:** A new sparse hash grid, `data`, containing the merged result [S1, unofficial].
**Data Structures:** The output `data` is a hash map (dictionary) mapping voxel coordinates `c` to their scalar values `v` [S1, unofficial].
**Steps:**
1. Initialize an empty hash map `data`.
2. Copy all active voxels from grid `a` to `data`. For each active voxel in `a` with coordinate `c` and value `v`, set `data[c] = v` [S1, unofficial].
3. Iterate through all active voxels in grid `b`. For each active voxel in `b` with coordinate `c` and value `v`: if `c` exists in `data`, update `data[c] = data[c] * v`; else, set `data[c] = v` [S1, unofficial].
**Formulas:** Not given in the sources.
**Parameters:** Not given in the sources.
**Edge Cases and Pitfalls:** Not given in the sources.
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multiply_compositing(grid_a, grid_b):
    data = {}
    for c, v in grid_a.active_voxels():
        data[c] = v
    for c, v in grid_b.active_voxels():
        if c in data:
            data[c] = data[c] * v
        else:
            data[c] = v
    return data
```
## Subtopics

### Replace Compositing
Returns a new grid starting with `a`'s active voxels, then overwriting with `b`'s active voxels, so `b` stamps on top of `a` [S1, unofficial].
**Steps:**
1. Copy all active voxels from grid `a` to `data` [S1, unofficial].
2. Iterate active voxels from grid `b`. Set `data[c] = v` (overwriting if present) [S1, unofficial].
3. Build grid using `build_grid(data, a.tree.background)` [S1, unofficial]. (Note: exact parameters not given in sources [S1, unofficial].)

## Operating characteristics
Not given in the sources.
## Background
Not given in the sources.
## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
