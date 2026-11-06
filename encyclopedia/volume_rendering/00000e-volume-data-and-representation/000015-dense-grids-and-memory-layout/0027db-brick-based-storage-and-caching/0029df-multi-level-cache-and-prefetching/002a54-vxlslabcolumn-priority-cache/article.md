# VxlSlabColumn Priority Cache


## Overview
The VxlSlabColumn Priority Cache is a memory management mechanism for voxel volume data that loads and recompiles slab columns based on priority levels to ensure spatial neighbors are available for surface operations [S1, unofficial]. It differs from sibling caches like the Rolling 3-Row Slab Cache by explicitly managing dirty/clean states via a priority enum (`Edit`, `Neighbor`, `Support`) and triggering recompilation only when necessary, rather than relying solely on spatial sliding windows [S1, unofficial].

## How it works
**Inputs:** A voxel grid with slab data, a cache window defined by `current_y` and `base_slot`, and a set of dirty columns requiring recompilation.
**Outputs:** A compiled column structure (`VxlSlabColumn`) with optimized slab headers and color runs, and an updated cache state.

**Data Structures:**
*   `VxlSlabColumn`:
    *   `len: usize`: Number of slabs in the column.
    *   `priority: VxlSlabColumnPriority`: Current priority level.
    *   `slabs: [VxlSlab; MAX_VERTICAL_STACK]`: Array of slabs.
*   `VxlSlab`:
    *   `start: u8`: Start Z-coordinate (inclusive).
    *   `end: u8`: End Z-coordinate (exclusive).
*   `VxlSlabColumnPriority` (enum, sorted by priority):
    *   `Support = 0`: Read-only, clean.
    *   `Neighbor`: Dirty, requires neighbor loading.
    *   `Edit`: Dirty, highest priority, requires recompilation.

**Algorithm Steps:**

1.  **Neighbor Caching:** When a center column is loaded or edited, iterate through all four horizontal directions (`VxlDirection::ARRAY`). For each direction, calculate the offset coordinates and call `cache_column` for the neighbor. The neighbor's priority is set to one level lower than the center column (e.g., if center is `Edit`, neighbor becomes `Neighbor`) [S1, unofficial].

2.  **Flush Logic:** Iterate through the cache's X-axis range from `flush_x_start` to `flush_x_end`. For each index `i`:
    *   Check if `columns[i]` exists.
    *   Check if `columns[i].priority > Support`.
    *   If not, continue to the next index.
    *   If yes, call `cache_neighbors` to ensure all spatial neighbors are loaded into the cache.
    *   Gather references to the neighbors.
    *   Call `self.compiled.recompile_column` to process the column [S1, unofficial].

3.  **Recompilation Process:** The recompiler processes the raw slabs into a compiled format:
    *   It iterates through slabs, handling states like `Unexposed`, `BottomColorRun`, and `FinishSpan` [S1, unofficial].
    *   It merges overlapping slabs and shifts elements to maintain a sorted, non-overlapping list [S1, unofficial].
    *   It writes headers (`VxlCompiledSpanHeader`) containing `len`, `top_color_start`, `top_color_end`, and `air_start` [S1, unofficial].
    *   It writes color data, fetching old colors from `ColorRunVec` or generating new ones via a `ColorFunc` [S1, unofficial].
    *   It handles edge cases where the column reaches the bottom of the map (`MAXZDIM`) [S1, unofficial].

4.  **Post-Flush Update:** After recompiling a column, set its `priority` back to `Support` to mark it as clean/read-only [S1, unofficial].

5.  **Cache Advancement:**
    *   Call `flush()` to process any remaining dirty rows.
    *   Call `reset_stale_row()` to clear data from the oldest row.
    *   Increment `current_y`.
    *   Update `base_slot` using modulo 3 arithmetic to wrap around the 3-row cache buffer [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def flush_cache(columns, flush_x_start, flush_x_end, current_y, base_slot, MAXZDIM):
    # Step 2: Flush Logic
    for i in range(flush_x_start, flush_x_end):
        if i not in columns:
            continue
        
        col = columns[i]
        # Check if priority is greater than Support (0)
        if col.priority <= 0:
            continue
            
        # Ensure neighbors are loaded
        cache_neighbors(col, columns)
        
        # Recompile the column
        recompile_column(col, columns, MAXZDIM)
        
        # Step 4: Post-flush update
        col.priority = 0 # Support

    # Step 5: Cache Advancement
    flush_remaining_dirty_rows(columns)
    reset_stale_row(columns, base_slot)
    current_y += 1
    base_slot = (base_slot + 1) % 3

def cache_neighbors(center_col, columns):
    # Step 1: Neighbor Caching
    for direction in VxlDirection.ARRAY:
        offset_coords = calculate_offset(center_col.coords, direction)
        neighbor_col = columns.get(offset_coords)
        
        if neighbor_col is None:
            # Load neighbor with decreased priority
            neighbor_col = load_column(offset_coords)
            # Priority decreases by one level
            # Edit(2) -> Neighbor(1), Neighbor(1) -> Support(0)
            neighbor_col.priority = max(0, center_col.priority - 1)
            columns[offset_coords] = neighbor_col
        else:
            # Update priority if it's higher than current
            # Note: Logic implies ensuring neighbors are 'available', 
            # priority management ensures they are loaded if needed.
            pass

def recompile_column(col, columns, MAXZDIM):
    new_column = VxlSlabColumn()
    cursor = 0
    header_cursor = 0
    state = CompileState.Unexposed
    unfinished_header = VxlCompiledSpanHeader()
    
    slab_iter = iter(col.slabs)
    try:
        slab = next(slab_iter)
    except StopIteration:
        slab = None

    neighbor_scanners = [None] * 4 # Placeholder for neighbor scanners
    
    while state != CompileState.Exit:
        if state == CompileState.Unexposed:
            # Skip past unexposed voxels
            z = compile_next_possible_surface_z(slab.end, neighbor_scanners, z)
            
            for neighbor in neighbor_scanners:
                if neighbor:
                    neighbor.advance_to(z)
            
            if z >= MAXZDIM - 1:
                state = CompileState.Exit
                continue
                
            state = CompileState.BottomColorRun
            
        elif state == CompileState.BottomColorRun:
            while compile_needs_color(slab, neighbor_scanners, z):
                compile_write_color(new_column, cursor, header_cursor, z, columns)
                z += 1
            # Transition logic handled internally or via state machine
            state = CompileState.Unexposed # Simplified flow
            
        elif state == CompileState.FinishSpan:
            if slab.end >= MAXZDIM:
                state = CompileState.Exit
                continue
                
            try:
                last_slab = slab
                slab = next(slab_iter)
            except StopIteration:
                state = CompileState.Exit
                continue
                
            compile_write_header(new_column, header_cursor, cursor, unfinished_header)
            unfinished_header = VxlCompiledSpanHeader(
                len=0,
                top_color_start=z,
                top_color_end=z-1,
                air_start=z
            )
            state = CompileState.Unexposed

    col.slabs = new_column.slabs
    col.len = new_column.len

def compile_next_possible_surface_z(slab_end, neighbor_scanners, z):
    next_boundary = slab_end - 1
    for neighbor in neighbor_scanners:
        if neighbor:
            boundary = neighbor.next_boundary_from(z)
            if boundary < next_boundary:
                next_boundary = boundary
    return next_boundary
```

## Operating characteristics
*   **Time Complexity:** Recompile is proportional to the number of slabs and color runs in the column. Neighbor loading is O(1) per neighbor. Flush is O(N) where N is the number of columns in the flush range.
*   **Memory Complexity:** O(1) per column in the cache window (3 rows x M columns).
*   **Strengths:** Efficiently handles dirty data by only recompiling when necessary. Ensures spatial consistency for surface operations.
*   **Limitations:** Edge cases require careful handling of neighbor loading (e.g., editing at cache boundaries) [S1, unofficial].
*   **When to use:** When voxel data is frequently edited and surface operations depend on accurate neighbor information.

## Background
The VxlSlabColumn Priority Cache is part of the Rust Voxelap library implementation, specifically designed for efficient voxel volume rendering and editing [S1, unofficial]. It addresses the need for fast access to voxel data while maintaining the integrity of spatial relationships during dynamic updates.

## Sources
[S1] MuffinTastic/rust-voxlap-vxl, https://github.com/MuffinTastic/rust-voxlap-vxl, fetched 2026-10-28 (unofficial)
