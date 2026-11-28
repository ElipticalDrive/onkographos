# Grid Background Change
## Overview
Grid Background Change updates the background value of inactive voxels in a volume grid while preserving the values of active voxels [S1, unofficial]. It returns a new grid where inactive voxels holding the old background value are updated to a new background value, while active voxels remain unchanged [S1, unofficial].
## How it works
**Inputs:**
- `grid`: The source volume grid containing active and inactive voxels [S1, unofficial].
- `new_bg`: The new background value (scalar type `T`) to assign to inactive voxels [S1, unofficial].

**Outputs:**
- A new grid instance with the same structure (tree, voxel size, name) but with inactive voxels set to `new_bg` [S1, unofficial].

**Data Structures:**
- The grid uses a tree structure where each leaf node contains a `value_mask` indicating active voxels and a `values` array storing scalar data [S1, unofficial].
- `Dict{Coord, T}`: A dictionary mapping 3D coordinates to scalar values for active voxels [S1, unofficial].

**Steps:**
1. Iterate all leaves in the tree [S1, unofficial].
2. For each leaf, iterate offsets in `on_indices(leaf.value_mask)` [S1, unofficial].
3. Decode coordinate `c` from `origin` and offset bits [S1, unofficial].
4. Store `leaf.values[offset + 1]` in a `Dict{Coord, T}` at key `c` [S1, unofficial].
5. Build the new grid using `build_grid(data, new_bg; name=grid.name, grid_class=grid.grid_class, voxel_size=vs)` where `vs` is the voxel size [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def grid_background_change(grid, new_bg):
    active_data = {}
    for leaf in grid.tree.leaves():
        for offset in on_indices(leaf.value_mask):
            c = decode_coordinate(leaf.origin, offset)
            active_data[c] = leaf.values[offset + 1]
    return build_grid(active_data, new_bg, grid.name, grid.grid_class, grid.voxel_size)

def on_indices(mask):
    """Yields bit offsets where mask has a 1."""
    i = 0
    while mask > 0:
        if mask & 1:
            yield i
        mask >>= 1
        i += 1

def decode_coordinate(origin, offset):
    """Maps leaf-local offset to global coordinate. Not given in the sources."""
    return origin + offset

def build_grid(data, new_bg, name, grid_class, voxel_size):
    """Constructs a new grid, initializing inactive voxels to new_bg and inserting active data. Not given in the sources."""
    grid = Grid(name=name, grid_class=grid_class, voxel_size=voxel_size)
    for c, v in data.items():
        grid.set(c, v)
    return grid

class Grid:
    def __init__(self, name, grid_class, voxel_size):
        self.name = name
        self.grid_class = grid_class
        self.voxel_size = voxel_size
        self.data = {}
    def set(self, c, v):
        self.data[c] = v
```

**Parameters:**
- `new_bg`: The new background scalar value. Typical values depend on the data type. Not given in the sources. [S1, unofficial]

**Edge Cases:**
- **Empty Active Set:** If no voxels are active, the new grid will consist entirely of `new_bg` [S1, unofficial].
- **Precision & Boundaries:** Coordinate decoding and boundary handling must correctly map leaf-local offsets to global coordinates [G].

## Subtopics

### [Grid Deactivation](3214-grid-deactivation.md)
Returns a new grid where any active voxel whose value equals a specific `value` is deactivated (removed from the active set). Choose it when isolating foreground objects by removing background noise is required.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to Use:** Not given in the sources.
- **When Not to Use:** Not given in the sources.

## Background
The method is defined in the Lyr.jl library [S1, unofficial].

## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
