# Dual Contouring 2D

Also called: DualContour2, dc2

## Overview
Dual Contouring 2D is a surface extraction algorithm that generates a 2D polygonal mesh (vertices and edges) from a scalar field defined on a uniform grid [S1, unofficial]. It solves the problem of extracting manifold 2D contours with sharp feature preservation by placing a single vertex inside each grid cell that intersects the isosurface, determined by minimizing a Quadratic Error Function (QEF) derived from intersection points and surface normals [S1, unofficial]. It differs from its sibling [Dual Contouring (2D Grid Implementation)](3764-dual-contouring-2d-grid-implementation.md) by utilizing a specific dispatch mechanism for predefined function types and a sequential cell-processing loop with explicit edge generation, rather than a generic iterative grid approach [S1, unofficial]. It builds on [Dual Contouring](778-dual-contouring.md) by restricting the domain to 2D and optimizing the QEF construction for planar cells [G].

## How it works
**Inputs and Outputs:**
The algorithm takes a scalar field function type, domain bounds, grid resolution, and output buffers [S1, unofficial].
- `ft`: Enum specifying the scalar field function (`FT_UNIT_SPHERE` or `FT_HEART`) [S1, unofficial].
- `xs, xt`: Starting and ending x coordinates of the domain [S1, unofficial].
- `ys, yt`: Starting and ending y coordinates of the domain [S1, unofficial].
- `n`: Number of grid intervals in the y-direction [S1, unofficial].
- `p`: Output buffer for mesh vertices, stored as an array of `float2` (x, y). Capacity `pcap` [S1, unofficial].
- `pcap`: Capacity of the vertex output buffer `p` [S1, unofficial].
- `pcnt`: Output variable storing the actual count of generated vertices [S1, unofficial].
- `edges`: Output buffer for mesh edges, stored as an array of `int2` (vertex index pairs). Capacity `ecap` [S1, unofficial].
- `ecap`: Capacity of the edge buffer `edges` [S1, unofficial].
- `ecnt`: Output variable storing the actual count of generated edges [S1, unofficial].

**Data Structures:**
The implementation uses temporary arrays allocated per cell processing or reused across the loop [S1, unofficial]:
- `S`: Sampled scalar values [S1, unofficial].
- `I`: Intersection indices/coordinates [S1, unofficial].
- `N`: Surface normals at intersection points [S1, unofficial].
- `E`: Edge indices [S1, unofficial].
- `ENS`: Edge normal signs or weights [S1, unofficial].
- `H`: Horizontal edge vertex indices (mapping cell edge to vertex index) [S1, unofficial].
- `V`: Vertical edge vertex indices (not explicitly detailed in the snippet but implied by vertical edge generation) [S1, unofficial].
- `QEF`: Quadratic Error Function matrix and vector for the current cell [S1, unofficial].

**Numbered Steps:**
1. **Dispatch:** The entry function `DualContour2` checks `ft`. If `FT_UNIT_SPHERE`, it calls `GenericDualContour2<FT_UNIT_SPHERE>`. If `FT_HEART`, it calls `GenericDualContour2<FT_HEART>`. Otherwise, it returns `false` [S1, unofficial].
2. **Initialization:** Inside `GenericDualContour2`, allocate temporary arrays `S, I, N, E, ENS, H, V, QEF`. Initialize `*pcnt = 0` and `*ecnt = 0` [S1, unofficial].
3. **Row Loop:** Iterate through grid rows `l` from `0` to `n - 2`. Let `prevy` be the y-coordinate of the previous row, and `y` be the current row's y-coordinate [S1, unofficial].
4. **Sample Scalar Field:** For the current row `y`, sample the scalar field values `s` across the x-domain `[xs, xt]` with `nsamples` points [S1, unofficial].
5. **Collect Intersection Edges:** For the strip between `prevy` and `y`, collect intersection points on the horizontal edges. This involves calling `CollectIntersectionEdges2` with the x-range and y-bounds, producing intersection data `ens` and edge indices `en` [S1, unofficial].
6. **Solve Intersections:** For the current row's horizontal edges, solve for the exact intersection points `ix, iy` using `SolveIntersection2`. This uses the scalar values and edge data [S1, unofficial].
7. **Sample Gradients:** Compute the surface normals `nx, ny` at the intersection points `ix, iy` using `SampleGradient2`. This provides the gradient information needed for the QEF [S1, unofficial].
8. **Construct QEF:** For each cell in the current row strip, construct the QEF. The QEF is built using intersection data from the current row (`CURR`) and the next row (`NEXTR`). The function `ConstructQEF2` takes intersection points, normals, and edge signs to form the linear system $A^T A x = A^T b$. The number of equations `nQEF` is returned [S1, unofficial].
9. **Check Capacity:** If `*pcnt + nQEF > pcap`, return `false` (buffer overflow) [S1, unofficial].
10. **Solve QEF:** Solve the QEF system to find the optimal vertex position. Store the result in `p` starting at index `2 * (*pcnt)`. The function `SolveQEF2` performs this minimization [S1, unofficial].
11. **Update Vertex Count:** Increment `*pcnt` by `nQEF` [S1, unofficial].
12. **Generate Horizontal Edges:** Iterate through cells `c` in the row. If both the current horizontal edge index `h1[c]` and the previous horizontal edge index `h1[c-1]` are valid (not `-1`), create an edge between the corresponding vertices. The vertex indices are offset by `base1` (the starting index of the current row's vertices). Check edge buffer capacity before adding [S1, unofficial].
13. **Generate Vertical Edges:** Iterate through cells. If `l != 0` (not the first row) and both the current horizontal edge index `h1[c]` and the current vertical edge index `h0[c]` are valid, create an edge between the corresponding vertices. The vertical vertex indices come from the previous row's solution. Check edge buffer capacity [S1, unofficial].
14. **Cleanup:** Free temporary arrays. Return `true` on success [S1, unofficial].

**Formulas:**
The QEF for a cell is constructed from linear constraints derived from surface normals $n_i$ and intersection points $p_i$:
$$ (n_i \cdot (x - p_i))^2 $$
Summing over all intersection points $i$ in the cell:
$$ \min_x \sum_i (n_i \cdot x - n_i \cdot p_i)^2 $$
This is solved as a least-squares problem:
$$ \min_x \| A x - b \|^2 $$
where $A$ is a matrix with rows $n_i^T$ and $b$ is a vector with entries $n_i \cdot p_i$ [G].

**Parameters:**
- `ft`: `FT_UNIT_SPHERE` (0), `FT_HEART` (1) [S1, unofficial].
- `xs, xt, ys, yt`: Domain bounds [S1, unofficial].
- `n`: Grid resolution [S1, unofficial].
- `pcap, ecap`: Buffer capacities [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Buffer Overflow:** If the number of generated vertices or edges exceeds `pcap` or `ecap`, the function returns `false`. Implementers must ensure sufficient capacity or handle the error [S1, unofficial].
- **Degenerate Cells:** If a cell does not intersect the isosurface, no vertex is generated, and `h` or `v` indices may be `-1`. Edge generation must check for `-1` [S1, unofficial].
- **Precision:** Floating-point precision in `SampleGradient2` and `SolveIntersection2` affects the accuracy of the QEF solution [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def DualContour2(ft, xs, xt, ys, yt, n, p, pcap, pcnt, edges, ecap, ecnt):
    if ft == 0:  # FT_UNIT_SPHERE
        return GenericDualContour2(0, xs, xt, ys, yt, n, p, pcap, pcnt, edges, ecap, ecnt)
    elif ft == 1:  # FT_HEART
        return GenericDualContour2(1, xs, xt, ys, yt, n, p, pcap, pcnt, edges, ecap, ecnt)
    else:
        return False

def GenericDualContour2(ft, xs, xt, ys, yt, n, p, pcap, pcnt, edges, ecap, ecnt):
    pcnt[0] = 0
    ecnt[0] = 0
    last = 0
    ystep = (yt - ys) / n  # [G]
    xstep = (xt - xs) / n  # [G]
    ncells = n             # [G]
    nsamples = n + 1       # [G]
    prevy = ys + ystep
    y = prevy + ystep

    # Allocate temporary arrays
    S = [None] * (n + 2)
    I = [None] * (n + 2)
    N = [None] * (n + 2)
    ENS = [None] * (n + 2)
    H = [None] * (n + 2)
    en = [None] * (n + 2)

    for l in range(n - 1):
        # Sample scalar field for current row
        Sample2(ft, xs, xt + xstep, y, y, S[l + 1], nsamples)

        # Collect intersection edges for the strip between prevy and y
        CollectIntersectionEdges2(xs, xt + xstep, prevy, prevy, xs, xt + xstep, y, y,
                                  S[l], S[l + 1], nsamples, xstep, ystep,
                                  ENS[l + 1], en[l + 1])

        # Solve for intersection points
        SolveIntersection2(xstep, ystep, I[l + 1], en[l + 1])

        # Sample gradients at intersection points
        SampleGradient2(I[l + 1], en[l + 1], N[l + 1])

        # Construct QEF for cells in the strip
        nQEF = ConstructQEF2(I[l], I[l + 1], N[l], N[l + 1], ENS[l], ENS[l + 1],
                             ncells, QEF, H[l + 1], nQEF)

        # Check vertex buffer capacity
        if pcnt[0] + nQEF > pcap:
            free_temp_arrays()
            return False

        # Solve QEF and store vertices
        SolveQEF2(QEF, p[2 * pcnt[0]:2 * (pcnt[0] + nQEF)], nQEF)
        pcnt[0] += nQEF

        # Generate horizontal edges
        base1 = pcnt[0] - nQEF
        h1 = H[l + 1]
        for c in range(1, ncells - 1):
            if h1[c] != -1 and h1[c - 1] != -1:
                if ecnt[0] + 1 > ecap:
                    free_temp_arrays()
                    return False
                edges[ecnt[0] * 2] = base1 + h1[c]
                edges[ecnt[0] * 2 + 1] = base1 + h1[c - 1]
                ecnt[0] += 1

        # Generate vertical edges
        base0 = base1 - last
        h0 = H[l]
        for c in range(ncells):
            if l != 0 and h1[c] != -1 and h0[c] != -1:
                if ecnt[0] + 1 > ecap:
                    free_temp_arrays()
                    return False
                edges[ecnt[0] * 2] = base1 + h1[c]
                edges[ecnt[0] * 2 + 1] = base0 + h0[c]
                ecnt[0] += 1

        last = nQEF
        prevy = y
        y += ystep

    free_temp_arrays()
    return True
```
*Note: Helper functions (`Sample2`, `CollectIntersectionEdges2`, `SolveIntersection2`, `SampleGradient2`, `ConstructQEF2`, `SolveQEF2`, `free_temp_arrays`) and parameters (`xstep`, `ystep`, `ncells`, `nsamples`, `S`, `I`, `N`, `E`, `ENS`, `H`, `V`, `QEF`, `en`, `xlow`, `ylow`, `xhigh`, `yhigh`) are defined in the source implementation [S1, unofficial]. `pcnt` and `ecnt` are passed as single-element lists to simulate C pointer semantics in Python.*

## Subtopics

### [Dual Contouring 2D (CUDA)](3671-dual-contouring-2d-cuda.md)
A GPU-accelerated implementation using CUDA parallelism, tiled thread blocks, atomic operations, and parallel prefix sum for vertex and edge generation. Choose this for high-performance extraction on NVIDIA GPUs.

### [Dual Contouring (2D Grid Implementation)](3764-dual-contouring-2d-grid-implementation.md)
A concrete, iterative implementation on a simple uniform grid structure. Choose this for a straightforward CPU-based implementation that builds on the core Dual Contouring 2D concepts.

## Operating characteristics
- **Time Complexity:** $O(N \cdot M)$ where $N$ is the number of grid cells in x and $M$ is the number of grid cells in y. Each cell is processed once, and QEF solving is constant time per cell (small linear system) [G].
- **Memory Complexity:** $O(N \cdot M)$ for the output buffers. Temporary arrays are $O(N)$ or $O(1)$ depending on implementation [G].
- **Measured Performance:** Not given in the sources.
- **Strengths:** Produces manifold meshes with sharp feature preservation. More accurate than Marching Squares for curved surfaces [S1, unofficial].
- **Limitations:** Requires buffer capacity checks. Sequential implementation may be slow for large grids [S1, unofficial].
- **When to use:** When manifold 2D meshes with sharp features are needed from a scalar field [S1, unofficial].
- **When not to use:** When GPU acceleration is required (use CUDA variant) or when a simpler, less accurate method suffices (use Marching Squares) [S1, unofficial].

## Background
Origin: The method is described in the unofficial source [S1, unofficial] from the `cudc` repository. It is a 2D specialization of the Dual Contouring algorithm, which was originally developed for 3D surface extraction. The 2D variant adapts the QEF minimization and dual graph construction to planar grids [G].

## Sources
[S1] thebusytypist/cudc, https://github.com/thebusytypist/cudc, fetched 2026-10-26 (unofficial)
