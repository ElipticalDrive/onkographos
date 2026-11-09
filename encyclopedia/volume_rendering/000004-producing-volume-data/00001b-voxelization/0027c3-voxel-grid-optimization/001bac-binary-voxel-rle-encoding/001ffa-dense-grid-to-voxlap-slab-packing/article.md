# Dense-grid to Voxlap-slab packing
## Overview
Converts a dense voxel mask and color grid of size VSID × VSID × MAXZDIM into the Voxlap slab-RLE column format [S1, unofficial]. It seeds an all-solid VXL and carves air voxels using a column function to preserve buried-voxel-aware encoding [S1, unofficial].
## How it works
**Inputs:**
- `grid`: A slice of `u8` bytes sized VSID × VSID × MAXZDIM in (y, x, z) order, where a non-zero byte marks a voxel as solid [S1, unofficial].
- `color`: A slice of `u32` bytes sized VSID × VSID × MAXZDIM, storing the BGRA colour for each voxel [S1, unofficial].
- `vsid`: The voxel side dimension (width/depth) [S1, unofficial].

**Outputs:**
- A `VXL` struct containing the packed voxel data, column offsets, and metadata [S1, unofficial].

**Data Structures:**
- `VXL`: Contains `vsid`, `ipo` (origin), `ist` (x-axis), `ihe` (z-axis), `ifo` (y-axis), `data` (boxed slice of bytes), `column_offset` (boxed slice of `u32`), `mip_base_offsets`, `vbit`, and `vbiti` [S1, unofficial].
- `column_offset`: A vector of `u32` with capacity `n_cols + 1`, where `n_cols = vsid²`, storing the byte offset into `data` for the start of each column [S1, unofficial].
- `data`: A vector of bytes storing the slab-RLE encoded columns [S1, unofficial].
- `Vspan`: Struct with fields `x` (`u32`), `y` (`u32`), `z0` (`u8`), `z1` (`u8`), where `z1` is inclusive [S1, unofficial].

**Parameters:**
- `MAXZDIM`: The maximum height of the voxel grid, defined as 256 (one byte per z value) [S1, unofficial].

**Steps:**
1. Validate input sizes: Assert `grid.len()` and `color.len()` equal `vsid² × MAXZDIM`, and panic if they do not [S1, unofficial].
2. Seed VXL data structures: Initialize `data` and `column_offset` vectors with capacity `n_cols * 8` and `n_cols + 1` respectively, where `n_cols = vsid²` [S1, unofficial].
3. Iterate columns: Iterate `y` from 0 to `vsid` and `x` from 0 to `vsid` [S1, unofficial]. Push the current `data` length to `column_offset` [S1, unofficial]. Write the 4-byte header `[nextptr=0, z1=0, z1c=0, z0=0]` to `data` [S1, unofficial]. Calculate the base index in `grid`/`color` [S1, unofficial]. If `grid[base]` is 0, set the base color to 0; otherwise set it to `color[base]` [S1, unofficial]. Write the base color as 4 little-endian bytes to `data` [S1, unofficial].
4. Finalize column offsets: Push the final `data` length to `column_offset` [S1, unofficial].
5. Construct VXL struct: Set `vsid`, `ipo=[0.0,0.0,0.0]`, `ist=[1.0,0.0,0.0]`, `ihe=[0.0,0.0,1.0]`, `ifo=[0.0,1.0,0.0]` [S1, unofficial]. Set `data` to `data.into_boxed_slice()` and `column_offset` to `column_offset.into_boxed_slice()` [S1, unofficial]. Set `mip_base_offsets` to `Box::new([0, n_cols + 1])`, `vbit` to `Box::new([])`, and `vbiti` to 0 [S1, unofficial].
6. Reserve edit capacity: Call `vxl.reserve_edit_capacity(n_cols * 512)` [S1, unofficial].
7. Build sorted air-span list: Initialize an empty `air_spans` vector [S1, unofficial]. Iterate `y` from 0 to `vsid` and `x` from 0 to `vsid` [S1, unofficial]. Calculate the base index and initialize `z=0` [S1, unofficial]. While `z < MAXZDIM`: if `grid[base+z] == 0`, set `zs=z` [S1, unofficial]. Inner while loop: while `z < MAXZDIM` and `grid[base+z] == 0`, increment `z` [S1, unofficial]. Push `Vspan { x: x as u32, y: y as u32, z0: zs as u8, z1: (z - 1) as u8 }` to `air_spans` [S1, unofficial]. Else increment `z` [S1, unofficial].
8. Carve air spans: If `air_spans` is not empty, call `set_spans_with_colfunc(&mut vxl, &air_spans, SpanOp::Carve, closure)` [S1, unofficial]. The closure takes `x`, `y`, `z`, calculates `idx = (y * vsid + x) * MAXZDIM + z`, and returns `roxlap_formats::VoxColor(color[idx])` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pack_dense_grid_to_vxl(grid: list, color: list, vsid: int) -> dict:
    MAXZDIM = 256
    n_cols = vsid * vsid

    # Step 1: Validate input sizes
    assert len(grid) == vsid * vsid * MAXZDIM
    assert len(color) == vsid * vsid * MAXZDIM

    # Step 2: Seed VXL data structures
    data = []
    column_offset = []

    # Step 3: Iterate columns
    for y in range(vsid):
        for x in range(vsid):
            column_offset.append(len(data))
            # Write header: nextptr=0, z1=0, z1c=0, z0=0 (4 bytes)
            data.extend([0, 0, 0, 0])

            base_idx = (y * vsid + x) * MAXZDIM
            if grid[base_idx] != 0:
                col_val = color[base_idx]
            else:
                col_val = 0
            # Write base color as 4 little-endian bytes
            data.extend(col_val.to_bytes(4, byteorder='little'))

    # Step 4: Finalize column offsets
    column_offset.append(len(data))

    # Step 5: Construct VXL struct
    vxl = {
        'vsid': vsid,
        'ipo': [0.0, 0.0, 0.0],
        'ist': [1.0, 0.0, 0.0],
        'ihe': [0.0, 0.0, 1.0],
        'ifo': [0.0, 1.0, 0.0],
        'data': bytes(data),
        'column_offset': column_offset,
        'mip_base_offsets': [0, n_cols + 1],
        'vbit': [],
        'vbiti': 0
    }

    # Step 6: Reserve edit capacity
    # vxl.reserve_edit_capacity(n_cols * 512) # Implementation detail not exposed in Python API

    # Step 7: Build sorted air-span list
    air_spans = []
    for y in range(vsid):
        for x in range(vsid):
            base_idx = (y * vsid + x) * MAXZDIM
            z = 0
            while z < MAXZDIM:
                if grid[base_idx + z] == 0:
                    zs = z
                    while z < MAXZDIM and grid[base_idx + z] == 0:
                        z += 1
                    air_spans.append({'x': x, 'y': y, 'z0': zs, 'z1': z - 1})
                else:
                    z += 1

    # Step 8: Carve air spans
    if air_spans:
        def closure(x, y, z):
            idx = (y * vsid + x) * MAXZDIM + z
            return color[idx]
        # set_spans_with_colfunc is not defined in the sources; it applies the column function to carve air spans [S1, unofficial]
        set_spans_with_colfunc(vxl, air_spans, 'Carve', closure)

    return vxl
```

**Pitfalls:**
- Grid size mismatch: The function panics if `grid.len()` or `color.len()` does not equal `vsid² × MAXZDIM` [S1, unofficial].
- Slab pool overflow: The function panics if a column's solid voxels cannot fit in the slab pool during carving [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Efficiently handles dense grids with large solid regions by seeding all-solid columns and only carving air [S1, unofficial]. Preserves color information for exposed voxels [S1, unofficial].
- **Limitations:** Requires `MAXZDIM` to be 256 [S1, unofficial]. Panics if grid size mismatch or slab pool overflow occurs during carving [S1, unofficial].
- **When to Use:** When converting dense voxel data to a format suitable for acoustic or rendering simulations that require buried-voxel awareness [S1, unofficial].
- **When Not to Use:** Not given in the sources.

## Background
Originated in the Roxlap project for acoustic simulation and cavity estimation [S1, unofficial].

## Sources
[S1] GitHub - NCrashed/roxlap: Rust portable reimplementation of Ken ..., https://github.com/NCrashed/roxlap, fetched 2026-10-29 (unofficial)
