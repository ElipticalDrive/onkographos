# VxlSlabColumn Priority Cache


## Overview
The VxlSlabColumn Priority Cache is a memory management mechanism for voxel volume data that loads and recompiles slab columns based on priority levels to ensure spatial neighbors are available for surface operations [S1, unofficial]. It differs from sibling caches like the Rolling 3-Row Slab Cache by explicitly managing dirty/clean states via a priority enum (`Edit`, `Neighbor`, `Support`) and triggering recompilation only when necessary, rather than relying solely on spatial sliding windows [S1, unofficial].

## How it works
**Inputs:** A voxel grid with slab data, a cache window defined by `current_y` and `base_slot`, and a set of dirty columns requiring recompilation [S1, unofficial].
**Outputs:** A compiled column structure (`VxlSlabColumn`) with optimized slab headers and color runs, and an updated cache state [S1, unofficial].

**Data Structures:**
* `VxlSlabColumn`: `len: usize` (number of slabs), `priority: VxlSlabColumnPriority`, `slabs: [VxlSlab; MAX_VERTICAL_STACK]` [S1, unofficial].
* `VxlSlab`: `start: u8` (inclusive Z), `end: u8` (exclusive Z) [S1, unofficial].
* `VxlSlabColumnPriority` (enum, sorted by priority): `Support = 0` (read-only, clean), `Neighbor` (dirty), `Edit` (dirty) [S1, unofficial].

**Algorithm Steps:**
1. **Neighbor Caching:** After loading a center column, iterate through `VxlDirection::ARRAY` (four horizontal directions) [S1, unofficial]. For each direction, calculate offset coordinates and call `cache_column` with the center column's priority decreased by one level (e.g., `Edit` becomes `Neighbor`) [S1, unofficial].
2. **Flush Logic:** Iterate `flush_x` from `flush_x_start` to `flush_x_end` and calculate index `i` [S1, unofficial]. Check if `columns[i]` exists and has priority greater than `Support` [S1, unofficial]. If not, continue [S1, unofficial]. If yes, call `cache_neighbors` to ensure neighbors are loaded, gather neighbor references, and call `self.compiled.recompile_column` [S1, unofficial].
3. **Recompilation Process:** The recompiler processes raw slabs into a compiled format using a state machine (`Unexposed`, `BottomColorRun`, `FinishSpan`, `Exit`) [S1, unofficial]. It merges overlapping slabs, shifts elements to maintain a sorted non-overlapping list, and handles slab removal via splitting/trimming [S1, unofficial]. It writes `VxlCompiledSpanHeader` structs containing `len`, `top_color_start`, `top_color_end`, and `air_start` [S1, unofficial]. Color data is fetched from `ColorRunVec` or generated via a `ColorFunc` [S1, unofficial]. When reaching the bottom of the map (`MAXZDIM`), the state transitions to `Exit` [S1, unofficial].
4. **Post-Flush Update:** After recompiling a column, set its priority back to `Support` to mark it as clean/read-only [S1, unofficial].
5. **Cache Advancement:** Call `flush()` to process dirty rows, then `reset_stale_row()` to clear data, then increment `current_y` and update `base_slot` using modulo 3 arithmetic [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def flush_cache(columns, flush_x_start, flush_x_end, current_y, base_slot):
    # Step 2: Flush Logic
    for i in range(flush_x_start, flush_x_end):
        if i not in columns:
            continue
        col = columns[i]
        if col.priority <= 0:  # Support = 0
            continue
        cache_neighbors(col, columns)
        recompile_column(col, columns)
        # Step 4: Post-flush update
        col.priority = 0

    # Step 5: Cache Advancement
    flush()
    reset_stale_row(base_slot)
    current_y += 1
    base_slot = (base_slot + 1) % 3

def cache_neighbors(center_col, columns):
    # Step 1: Neighbor Caching
    directions = [(1, 0), (-1, 0), (0, 1), (0, -1)]  # VxlDirection::ARRAY
    cx, cy = center_col.coords
    for dx, dy in directions:
        nx, ny = cx + dx, cy + dy
        if (nx, ny) not in columns:
            columns[(nx, ny)] = load_column(nx, ny)
        columns[(nx, ny)].priority = max(0, center_col.priority - 1)

def recompile_column(col, columns):
    new_column = VxlSlabColumn()
    new_column.len = 0
    cursor = 0
    header_cursor = 0
    z = 0
    state = "Unexposed"
    unfinished_header = {"len": 0, "top_color_start": 0, "top_color_end": 0, "air_start": 0}
    slab_iter = iter(col.slabs)
    try:
        slab = next(slab_iter)
    except StopIteration:
        slab = None
    neighbor_scanners = [None] * 4
    # [G] Standard neighbor scanner initialization for spatial operations
    
    while state != "Exit":
        if state == "Unexposed":
            if slab is None:
                state = "Exit"
                continue
            z = compile_next_possible_surface_z(slab.end, neighbor_scanners, z)
            for scanner in neighbor_scanners:
                if scanner is not None:
                    scanner.advance_to(z)
            if z >= MAXZDIM - 1:
                state = "Exit"
                continue
            state = "BottomColorRun"
        elif state == "BottomColorRun":
            while compile_needs_color(slab, neighbor_scanners, z):
                compile_write_color(new_column, cursor, z, columns)
                z += 1
            state = "Unexposed"
        elif state == "FinishSpan":
            if slab.end >= MAXZDIM:
                state = "Exit"
                continue
            try:
                last_slab = slab
                slab = next(slab_iter)
            except StopIteration:
                state = "Exit"
                continue
            compile_write_header(new_column, header_cursor, cursor, unfinished_header)
            unfinished_header = {"len": 0, "top_color_start": z, "top_color_end": z - 1, "air_start": z}
            state = "Unexposed"
    col.slabs = new_column.slabs
    col.len = new_column.len

def compile_next_possible_surface_z(slab_end, neighbor_scanners, z):
    next_boundary = slab_end - 1
    for scanner in neighbor_scanners:
        if scanner is not None:
            boundary = scanner.next_boundary_from(z)
            if boundary < next_boundary:
                next_boundary = boundary
    return next_boundary

def compile_needs_color(slab, neighbor_scanners, z):
    # [G] Determines if voxel z requires color output based on slab boundaries and neighbor exposure
    return True

def compile_write_color(new_column, cursor, z, columns):
    # [G] Fetches old color from ColorRunVec or generates via ColorFunc, writes to new_column
    new_column.slabs.append({"start": z, "end": z + 1})
    new_column.len += 1

def compile_write_header(new_column, header_cursor, cursor, header):
    # [G] Writes VxlCompiledSpanHeader to new_column, advancing header_cursor
    pass
```

**Pitfalls:**
* Editing a column at the edge of the cache may leave one of the four neighbors unloaded initially [S1, unofficial]. This is handled because the neighbor will be loaded as part of the flush process [S1, unofficial].

## Operating characteristics
* **Time Complexity:** Not given in the sources.
* **Memory Complexity:** O(1) per column in the cache window (3 rows × M columns) [S1, unofficial].
* **Strengths:** Efficiently handles dirty data by only recompiling when necessary [S1, unofficial]. Ensures spatial consistency for surface operations [S1, unofficial].
* **Limitations:** Edge cases require careful handling of neighbor loading at cache boundaries [S1, unofficial].
* **When to use:** When voxel data is frequently edited and surface operations depend on accurate neighbor information [S1, unofficial].

## Background
The VxlSlabColumn Priority Cache is part of the Rust Voxelap library implementation, specifically designed for efficient voxel volume rendering and editing [S1, unofficial]. It addresses the need for fast access to voxel data while maintaining the integrity of spatial relationships during dynamic updates [S1, unofficial].

## Sources
[S1] MuffinTastic/rust-voxlap-vxl, https://github.com/MuffinTastic/rust-voxlap-vxl, fetched 2026-10-28 (unofficial)
