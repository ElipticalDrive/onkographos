# NanoVolumeHDDA

Also called: Span-merging hierarchical DDA, Zero-allocation callback-based HDDA

## Overview
NanoVolumeHDDA is an iterator that traverses a `NanoGrid` along a ray to yield merged active spans $(t_0, t_1)$ without heap allocations, using a callback interface [S1, unofficial]. It merges adjacent active leaves and tiles to eliminate seam artifacts at 8-voxel tile edges [S1, unofficial]. It builds on [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md) by operating specifically on the `NanoGrid` structure and optimizing for zero-allocation iteration [S1, unofficial].

## How it works
**Inputs:** A `NanoGrid` (containing the sparse voxel data), a `Ray` (origin and direction) [S1, unofficial].
**Outputs:** `TimeSpan` structs representing merged active regions along the ray in front-to-back order, yielded via an iterator protocol that internally invokes a callback [S1, unofficial].

**Data Structures:**
*   `TimeSpan`: A struct with fields `t0::Float64` and `t1::Float64` [S1, unofficial].
*   `HDDAState{T}`: A mutable struct maintaining traversal state [S1, unofficial]:
    *   `roots::Vector{Tuple{Float64, Int}}`: List of root hits, storing `(tmin, offset)` [S1, unofficial].
    *   `root_idx::Int`: Current index in the roots list [S1, unofficial].
    *   `i2_ndda::Union{NodeDDA, Nothing}`: The DDA state for the I2 (tile) level [S1, unofficial].
    *   `i2_off::Int`: Offset in the buffer for the current I2 node [S1, unofficial].
    *   `i2_t_entry::Float64`: Entry time for the current I2 cell [S1, unofficial].
    *   `i1_ndda::Union{NodeDDA, Nothing}`: The DDA state for the I1 (leaf) level [S1, unofficial].
    *   `i1_off::Int`: Offset in the buffer for the current I1 node [S1, unofficial].
    *   `i1_t_entry::Float64`: Entry time for the current I1 cell [S1, unofficial].
    *   `span_t0::Float64`: Start time of the current accumulated span; `-1.0` indicates no open span [S1, unofficial].
*   `NodeDDA`: A simple state struct for hierarchical DDA stepping, containing fields `t`, `t_max`, `step`, `cidx`, `origin`, `size` [G].

**Parameters:**
*   `_MAX_ROOTS`: Maximum number of root nodes to process, set to 8 [S1, unofficial].
*   `span_t0`: `-1.0` indicates no open span [S1, unofficial].

**Algorithm Steps:**

1.  **Collect Root Hits:**
    Iterate `i` from 0 to `root_count - 1`.
    *   Calculate entry pointer `ep = root_pos + i * entry_sz` [S1, unofficial].
    *   Load `is_child` from buffer at `ep + 12`. If `is_child != 0x01`, continue to next `i` [S1, unofficial].
    *   Load `i2_off` from buffer at `ep + 13` [S1, unofficial].
    *   Load `origin` from buffer at `i2_off` [S1, unofficial].
    *   Construct an AABB with `origin` as the center and size 4096.0 (extending to `origin + (4096, 4096, 4096)`) [S1, unofficial].
    *   Intersect `ray` with this AABB. If the intersection is valid, let `hit[1]` be the entry time `tmin`. Store `(tmin, i2_off)` in the `roots` list [S1, unofficial].
    *   Break if the number of roots exceeds `_MAX_ROOTS` (8) [S1, unofficial].

2.  **Sort Roots:**
    Perform an insertion sort on the `roots` list based on the entry time `tmin` (first element of the tuple) to ensure front-to-back processing order [S1, unofficial].

3.  **Initialize State:**
    *   If `roots` is empty, return `nothing` (no intersection) [S1, unofficial].
    *   Initialize `HDDAState` with the sorted `roots`, `root_idx = 0`, `i2_ndda = nothing`, `i2_off = 0`, `i2_t_entry = 0.0`, `i1_ndda = nothing`, `i1_off = 0`, `i1_t_entry = 0.0`, and `span_t0 = -1.0` [S1, unofficial].

4.  **Iterate (Callback Loop):**
    The iteration proceeds by calling an internal advance function `_hdda_advance` which manages two phases [S1, unofficial].

    **Phase 1: I1 (Leaf) DDA Loop**
    *   While `s.i1_ndda` is not `nothing` and `node_dda_inside(s.i1_ndda)` is true:
        *   Get `ndda = s.i1_ndda` and `cidx = node_dda_child_index(ndda)` [S1, unofficial].
        *   Check `has_child` by testing if the child mask bit `cidx` is set at `s.i1_off + _I1_CMASK_OFF` in the buffer [S1, unofficial].
        *   Check `has_tile`: If `!has_child`, check if the voxel mask bit `cidx` is set at `s.i1_off + _I1_VMASK_OFF` [S1, unofficial].
        *   If `has_child` or `has_tile` is true AND `s.span_t0 < 0.0`:
            *   Set `s.span_t0 = s.i1_t_entry` [S1, unofficial].
        *   If `!(has_child or has_tile)` is true AND `s.span_t0 >= 0.0`:
            *   Call the callback `f(s.span_t0, s.i1_t_entry)` [S1, unofficial].
            *   Reset `s.span_t0 = -1.0` [S1, unofficial].
        *   Update `s.i1_t_entry = node_dda_cell_time(ndda)` [S1, unofficial].
        *   Step `ndda.state` [S1, unofficial].

    **Phase 2: I2 (Tile) DDA Loop**
    *   After Phase 1 (or if Phase 1 was not entered), loop while `s.i2_ndda` is not `nothing` and `node_dda_inside(s.i2_ndda)` is true:
        *   Get `ndda = s.i2_ndda` and `cidx = node_dda_child_index(ndda)` [S1, unofficial].
        *   Check `has_child` by testing if the child mask bit `cidx` is set at `s.i2_off + _I2_CMASK_OFF` in the buffer [S1, unofficial].
        *   If `has_child` is true:
            *   Calculate `tidx` using `_buf_count_on_before` to determine the child index count [S1, unofficial].
            *   Load `i1_off` for the child node [S1, unofficial].
            *   Construct an I1 AABB with size 128.0 centered at the child's origin [S1, unofficial].
            *   Intersect `ray` with this I1 AABB. If it hits:
                *   Initialize `s.i1_ndda` with `node_dda_init(ray, tmin, i1_origin, Int32(16), Int32(8))` [S1, unofficial].
                *   Set `s.i1_t_entry = tmin` [S1, unofficial].
                *   Set `s.i1_off = i1_off` [S1, unofficial].
                *   `continue` to the I2 loop (which will re-enter Phase 1 in the next advance call) [S1, unofficial].
            *   If I1 AABB misses:
                *   If `s.span_t0 >= 0.0`, call `f(s.span_t0, s.i2_t_entry)` and reset `s.span_t0 = -1.0` [S1, unofficial].
        *   If `has_child` is false:
            *   Check `has_tile` by testing if the voxel mask bit `cidx` is set at `s.i2_off + _I2_VMASK_OFF` [S1, unofficial].
            *   If `has_tile` is true AND `s.span_t0 < 0.0`:
                *   Set `s.span_t0 = s.i2_t_entry` [S1, unofficial].
            *   If `!has_tile` is true AND `s.span_t0 >= 0.0`:
                *   Call `f(s.span_t0, s.i2_t_entry)` and reset `s.span_t0 = -1.0` [S1, unofficial].
        *   Update `s.i2_t_entry = node_dda_cell_time(ndda)` [S1, unofficial].
        *   Step `ndda.state` [S1, unofficial].

    **Finalization:**
    *   After the I2 loop exhausts, if `s.span_t0 >= 0.0`, call `f(s.span_t0, s.i2_t_entry)` and reset `s.span_t0 = -1.0` [S1, unofficial].
    *   Return the next span or `nothing` if traversal is complete [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nano_volume_hdda_iterate(grid, ray, callback):
    """
    Iterates over a NanoGrid along a ray, yielding merged active spans via callback.
    """
    buf = grid.buffer
    root_count = grid.root_count
    root_pos = grid.root_buffer_ptr
    entry_sz = 16  # Based on offsets ep+12, ep+13
    
    roots = []
    for i in range(root_count):
        ep = root_pos + i * entry_sz
        is_child = buf[ep + 12]
        if is_child != 0x01:
            continue
        i2_off = int.from_bytes(buf[ep+13:ep+17], 'little')
        origin = read_vec3(buf, i2_off)
        aabb_min = origin - vec3(2048.0)
        aabb_max = origin + vec3(2048.0)
        hit = intersect_bbox(ray, aabb_min, aabb_max)
        if hit is not None:
            roots.append((hit.entry_time, i2_off))
        if len(roots) >= 8:
            break
            
    # Insertion sort
    for i in range(1, len(roots)):
        j = i
        while j > 0 and roots[j][0] < roots[j-1][0]:
            roots[j], roots[j-1] = roots[j-1], roots[j]
            j -= 1
            
    if not roots:
        return

    state = {
        'roots': roots, 'root_idx': 0,
        'i2_ndda': None, 'i2_off': 0, 'i2_t_entry': 0.0,
        'i1_ndda': None, 'i1_off': 0, 'i1_t_entry': 0.0,
        'span_t0': -1.0
    }
    
    while True:
        span, state = _hdda_advance(buf, ray, state)
        if span is None:
            break
        callback(span[0], span[1])

def _hdda_advance(buf, ray, s):
    # Phase 1: I1 DDA
    while s['i1_ndda'] is not None and node_dda_inside(s['i1_ndda']):
        ndda = s['i1_ndda']
        cidx = ndda['cidx']
        has_child = (buf[s['i1_off'] + _I1_CMASK_OFF] >> cidx) & 1
        has_tile = (not has_child) and (buf[s['i1_off'] + _I1_VMASK_OFF] >> cidx) & 1
        
        if (has_child or has_tile) and s['span_t0'] < 0.0:
            s['span_t0'] = s['i1_t_entry']
            
        if not (has_child or has_tile) and s['span_t0'] >= 0.0:
            span = (s['span_t0'], s['i1_t_entry'])
            s['span_t0'] = -1.0
            s['i1_t_entry'] = ndda['t']
            step_ndda(ndda)
            return span, s
            
        s['i1_t_entry'] = ndda['t']
        step_ndda(ndda)
        
    s['i1_ndda'] = None
    
    # Phase 2: I2 DDA
    while s['i2_ndda'] is not None and node_dda_inside(s['i2_ndda']):
        ndda = s['i2_ndda']
        cidx = ndda['cidx']
        has_child = (buf[s['i2_off'] + _I2_CMASK_OFF] >> cidx) & 1
        
        if has_child:
            tidx = sum(1 for k in range(cidx) if (buf[s['i2_off'] + _I2_CMASK_OFF] >> k) & 1)
            i1_off = s['i2_off'] + tidx * 4  # child_stride assumed 4 bytes per child offset
            i1_origin = read_vec3(buf, i1_off)
            aabb_min = i1_origin - vec3(64.0)
            aabb_max = i1_origin + vec3(64.0)
            hit = intersect_bbox(ray, aabb_min, aabb_max)
            if hit is not None:
                tmin = hit.entry_time
                s['i1_ndda'] = node_dda_init(ray, tmin, i1_origin, 16, 8)
                s['i1_t_entry'] = tmin
                s['i1_off'] = i1_off
                return None, s  # Continue to I2 loop, re-enters Phase 1 next advance
            else:
                if s['span_t0'] >= 0.0:
                    span = (s['span_t0'], s['i2_t_entry'])
                    s['span_t0'] = -1.0
                    return span, s
        else:
            has_tile = (buf[s['i2_off'] + _I2_VMASK_OFF] >> cidx) & 1
            if has_tile and s['span_t0'] < 0.0:
                s['span_t0'] = s['i2_t_entry']
            elif not has_tile and s['span_t0'] >= 0.0:
                span = (s['span_t0'], s['i2_t_entry'])
                s['span_t0'] = -1.0
                return span, s
                
        s['i2_t_entry'] = ndda['t']
        step_ndda(ndda)
        
    if s['span_t0'] >= 0.0:
        span = (s['span_t0'], s['i2_t_entry'])
        s['span_t0'] = -1.0
        return span, s
        
    return None, s

# Constants and helpers
_I1_CMASK_OFF = 0
_I1_VMASK_OFF = 4
_I2_CMASK_OFF = 0
_I2_VMASK_OFF = 4
# Note: Offsets are relative to node buffer pointers. Exact values depend on NanoGrid layout [G].

def read_vec3(buf, addr):
    return vec3(int.from_bytes(buf[addr:addr+4], 'little'),
                int.from_bytes(buf[addr+4:addr+8], 'little'),
                int.from_bytes(buf[addr+8:addr+12], 'little'))

def intersect_bbox(ray, min, max):
    # Standard slab intersection
    tmin = (min.x - ray.origin.x) / ray.direction.x
    tmax = (max.x - ray.origin.x) / ray.direction.x
    if tmin > tmax: tmin, tmax = tmax, tmin
    tymin = (min.y - ray.origin.y) / ray.direction.y
    tymax = (max.y - ray.origin.y) / ray.direction.y
    if tymin > tymax: tymin, tymax = tymax, tymin
    if tmin > tymax or tymin > tmax: return None
    if tymin > tmin: tmin = tymin
    if tymax < tmax: tmax = tymax
    tzmin = (min.z - ray.origin.z) / ray.direction.z
    tzmax = (max.z - ray.origin.z) / ray.direction.z
    if tzmin > tzmax: tzmin, tzmax = tzmax, tzmin
    if tmin > tzmax or tzmin > tmax: return None
    if tzmin > tmin: tmin = tzmin
    return type('Hit', (), {'entry_time': tmin})()

def node_dda_init(ray, t_entry, origin, cell_size, tile_size):
    # Simplified DDA state initialization
    step = vec3(cell_size / abs(ray.direction.x), cell_size / abs(ray.direction.y), cell_size / abs(ray.direction.z))
    t_step = vec3(step.x if ray.direction.x > 0 else -step.x,
                  step.y if ray.direction.y > 0 else -step.y,
                  step.z if ray.direction.z > 0 else -step.z)
    return {'t': t_entry, 't_max': t_entry + tile_size, 'step': t_step, 'cidx': 0, 'origin': origin, 'size': cell_size}

def node_dda_inside(ndda):
    return ndda['t'] < ndda['t_max']

def node_dda_child_index(ndda):
    return ndda['cidx']

def node_dda_cell_time(ndda):
    return ndda['t']

def step_ndda(ndda):
    ndda['t'] += ndda['step'].x if ndda['cidx'] == 0 else (ndda['step'].y if ndda['cidx'] == 1 else ndda['step'].z)
    ndda['cidx'] = (ndda['cidx'] + 1) % 3
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** $O(1)$ auxiliary memory per ray traversal instance, as it avoids heap allocations for span lists [S1, unofficial].
*   **Strengths:** Zero-allocation iteration makes it suitable for GPU kernels or high-performance CPU loops where memory fragmentation or GC pressure is a concern [S1, unofficial]. Merging spans reduces the number of shader invocations [S1, unofficial].
*   **Limitations:** Bounded root count (8) may miss intersections in highly fragmented scenes if not handled by the caller [S1, unofficial]. Specific to `NanoGrid` structure [S1, unofficial].
*   **When to use:** When rendering sparse voxel grids (VDB) with `NanoGrid` and performance/zero-allocation is critical [S1, unofficial].

## Background
NanoVolumeHDDA is part of the Lyr.jl rendering library, developed by Tobias Osborne [S1, unofficial]. It is designed to efficiently traverse sparse voxel data structures (specifically `NanoGrid`) for volume rendering, leveraging the hierarchical nature of the data to skip empty space and merge active regions [S1, unofficial].

## Sources
[S1] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
