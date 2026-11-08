# Row-difference projection for AMR cells


## Overview
Row-difference projection is an exact rendering method for rotated Adaptive Mesh Refinement (AMR) cells that projects them along pixel rows using row-difference accumulation, replacing a separate per-pixel exact path [S1]. It handles cells truncated by depth ranges and unit cubes used for interpolation tables by treating them as large cells within the row-difference framework [S1]. It sets itself apart from its siblings by eliminating the need for general clipped-polytope volume calculations and separate exact paths, relying instead on the divergence theorem for efficient volume integration within quadrants [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A rotated AMR cell (or unit cube), a z-range $[z_{min}, z_{max}]$ defining the projection depth limits, and the image grid geometry [G].
*   **Output:** Accumulated row-difference values for the affected pixels, which are later summed to produce the final image contribution [G].

**Data Structures:**
*   **Second-difference buffer:** A floating-point buffer (float64) storing the second differences of the projected values per pixel. This buffer must remain in double precision to maintain accuracy [S1].
*   **Quadrant limits:** Clamped coordinates defining the extent of the cell within the image grid [G].

**Steps:**
1.  **Clamp Quadrant Limits:** Before processing, clamp the quadrant limits to the cube's geometric extent. This avoids numerical cancellation errors when the projection is called with effectively infinite z-limits [S1].
2.  **Determine Projection Path:** If the rotated cell covers a size threshold and lies entirely within the z-range, use the precomputed interpolation table [S1]. Otherwise, project the cell exactly using the row-difference method [S1]. This replaces the previous logic where cells truncated by depth ranges or unit cubes used a separate per-pixel exact path [S1].
3.  **Row Projection with Z-Range:** The row projection accepts the z-range $[z_{min}, z_{max}]$ [S1]. Identify the cross-sections of the cube at $z_{min}$ and $z_{max}$. These cross-sections introduce up to 12 additional lines along which the column's gradient changes [S1]. Limit the integration chords to the specified z-range [S1].
4.  **Volume Calculation via Divergence Theorem:** Calculate the volume of the cube within a specific quadrant and the z-range using the divergence theorem [S1]. Apply the theorem about a point located on three of the four clipping planes [S1]. This approach ensures that only the clipped cube faces and one cross-section contribute to the integral, replacing the general clipped-polytope calculation which is now removed [S1].
5.  **In-Place Accumulation:** Perform cumulative sums, clamping, and addition on the second-difference buffer in place [S1]. This reduces memory usage from 28 bytes per pixel (per thread) to 12 bytes per pixel, consisting of the float32 result and the float64 buffer [S1].

**Formulas:**
*   Volume contribution per quadrant: $V = \oint_{\partial \Omega} \mathbf{F} \cdot d\mathbf{A}$, evaluated about a reference point on three of the four clipping planes, where $\partial \Omega$ comprises only the clipped cube faces and the single cross-section at the z-boundary [S1].

**Parameters:**
*   `row_threshold`: This parameter was removed. Rotated cells now use a table if they cover their size and lie within the z range; otherwise, they are projected exactly [S1].

**Edge Cases and Pitfalls:**
*   **Numerical Cancellation:** Quadrant limits must be clamped to the cube's extent before use to avoid numerical cancellation when called with effectively infinite limits [S1].
*   **Memory Usage:** The final cumulative sums over the second-difference buffer previously created three image-sized temporaries, peaking at 28 bytes per pixel per thread. This is reduced to 12 bytes per pixel by performing sums, clamping, and addition in place, keeping the float64 buffer in double precision [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def project_rotated_cell_amr(cell, z_min, z_max, width, height):
    # Initialize second-difference buffer (float64) and result buffer (float32)
    second_diff = [[0.0] * width for _ in range(height)]
    result = [[0.0] * width for _ in range(height)]

    # Step 1: Clamp quadrant limits to cube extent
    q_min_x = max(cell.q_min_x, cell.geom.min_x)
    q_max_x = min(cell.q_max_x, cell.geom.max_x)
    q_min_y = max(cell.q_min_y, cell.geom.min_y)
    q_max_y = min(cell.q_max_y, cell.geom.max_y)

    # Step 2: Determine projection path
    if cell.covers_threshold() and z_min <= cell.z_min and cell.z_max <= z_max:
        # Use interpolation table for efficiency [G]
        # Sources do not specify table lookup implementation [G]
        table_vals = [[0.0] * (q_max_x - q_min_x) for _ in range(q_max_y - q_min_y)]
        for y in range(height):
            for x in range(width):
                if q_min_x <= x < q_max_x and q_min_y <= y < q_max_y:
                    second_diff[y][x] += table_vals[y - q_min_y][x - q_min_x]
    else:
        # Step 3: Row projection with z-range
        # Cross-sections at z_min and z_max add up to 12 lines where gradient changes
        # Sources do not specify cross-section line computation [G]
        gradient_lines = []
        # Limit chords to z-range
        # Sources do not specify chord limiting implementation [G]
        chords = cell.chords

        # Step 4: Volume calculation via divergence theorem
        for quadrant in cell.quadrants:
            # Apply divergence theorem about a point on three clipping planes
            # Sources do not specify divergence theorem volume calculation [G]
            vol = 0.0
            # Accumulate volume contribution into buffer based on pixel coverage
            for y in range(height):
                for x in range(width):
                    # Sources do not specify pixel coverage check [G]
                    if True:
                        second_diff[y][x] += vol

    # Step 5: In-place cumulative sums, clamping, and addition
    # Reduces memory from 28 to 12 bytes/pixel
    for y in range(height):
        for x in range(width):
            if y > 0:
                second_diff[y][x] += second_diff[y-1][x]
            # Clamp and add to result
            val = second_diff[y][x]
            if val < 0.0:
                val = 0.0
            if val > 1.0:
                val = 1.0
            result[y][x] += float(val)

    return result, second_diff
```

## Operating characteristics
*   **Time Complexity:** Depth-limited projections are 2-20x faster than the previous method [S1]. Other projections are unchanged within timing noise [S1].
*   **Memory Complexity:** Peak memory usage is reduced to 12 bytes per pixel (per thread) for the second-difference buffer and final values, down from 28 bytes per pixel [S1].
*   **Measured Performance:** Tested on synthetic lattices and Ramses test data [S1].
*   **Strengths:** Eliminates the need for general clipped-polytope calculations and separate per-pixel exact paths, simplifying the codebase and improving performance for depth-limited projections [S1].
*   **Limitations:** Requires careful handling of numerical precision, necessitating double precision for the second-difference buffer [S1].
*   **When to Use:** Use for rendering rotated AMR cells, especially when depth-limited projections are required or when cells are truncated by depth ranges [S1].

## Background
This method was introduced in the context of the `pynbody` library, specifically within the `pynbody.imaging` package, to handle AMR rendering more efficiently [S1]. It was developed to replace a separate per-pixel exact path for cells truncated by depth ranges and unit cubes used for interpolation tables [S1]. The change was part of a broader effort to move rendering from `pynbody.sph` to a new `pynbody.imaging` package, reflecting the handling of both AMR and SPH simulations [S1].

## Sources
[S1] Opt-in exact AMR rendering, new pynbody.imaging package, and SPH perspective fixes, https://github.com/pynbody/pynbody/pull/1048, fetched 2026-10-26
