# Polyhedron Moment Integration and Voxelization

Also called: Polyhedron Moment Integration, R3D Polyhedron Moment Integration and Voxelization, R3D moment-based volume integration, simplicial decomposition integration

## Overview
Polyhedron Moment Integration computes the polynomial density moments of a convex polyhedron within a voxel grid by recursively splitting the polyhedron along its longest axis until it occupies single voxels, then integrating the polynomial density field over the resulting polyhedron volume for each voxel [S1, unofficial]. It differs from [Stochastic Ray-Casting Voxelization](1067-stochastic-ray-casting-voxelization.md) and [Ray-Cast Style Voxel Filling](1502-ray-cast-style-voxel-filling.md) by providing exact analytic integration of density moments rather than stochastic or rasterization-based approximations, and from [Mesh Winding Number Voxelization](2328-mesh-winding-number-voxelization.md) by supporting arbitrary polynomial density orders (not just binary occupancy) [S1, unofficial].

## How it works
**Inputs:**
- `poly`: A convex polyhedron structure (`r3d_poly`) containing vertices, neighbor indices, and orientation flags [S1, unofficial].
- `node`: A bounding box defined by minimum indices `imin`, `jmin`, `kmin` and dimensions `ioff`, `joff`, `koff` representing the grid region to process [S1, unofficial].
- `polyorder`: Integer specifying the polynomial order: 0 for constant (1 moment), 1 for linear (4 moments), 2 for quadratic (10 moments) [S1, unofficial].
- `dest_grid`: Output array for moments, sized according to grid dimensions and moment count [S1, unofficial].
- `gridsz`: Total grid dimensions `(gridsz.i, gridsz.j, gridsz.k)` [S1, unofficial].

**Outputs:**
- `dest_grid`: Array filled with integration results. The order of moments is `1`, `x`, `y`, `z`, `x^2`, `y^2`, `z^2`, `x*y`, `y*z`, `z*x` for order 2 [S1, unofficial].

**Data Structures:**
- `r3d_poly`: Contains `r3d_vertex verts[R3D_MAX_VERTS]` (vertex buffer, max 64), `r3d_int nverts` (number of vertices), and `r3d_int funclip` (index of any unclipped vertex) [S1, unofficial].
- `r3d_vertex`: Contains `r3d_rvec3 pos` (position), `unsigned char pnbrs[3]` (neighbor indices), and `r3d_orientation orient` (orientation w.r.t clip planes) [S1, unofficial].
- `r3d_orientation`: Contains `fflags` (bitmask of face intersections) and `fdist` (distance to faces) [S1, unofficial].
- `Stack Entry`: Contains `r3d_poly poly` and node bounding box fields (`imin`, `jmin`, `kmin`, `ioff`, `joff`, `koff`) [S1, unofficial]. Max stack size is `ceil(log2(gridsz.i)) + ceil(log2(gridsz.j)) + ceil(log2(gridsz.k)) + 1` [S1, unofficial].
- `R3D_NUM_MOMENTS(order)`: Macro calculating array size for moments: `((order + 1) * (order + 2) * (order + 3) / 6)` [S1, unofficial].

**Steps:**
1. **Initialize:** Check for invalid inputs (null pointers, non-positive grid sizes). If invalid, return immediately [S1, unofficial].
2. **Push Initial State:** Copy input poly and node bounding box to `stack[0]`. Set `nstack = 1` [S1, unofficial].
3. **Process Stack:** While `nstack > 0`:
    a. **Pop State:** Decrement `nstack` and retrieve current state from `stack[nstack]` [S1, unofficial].
    b. **Find Split Axis:** Iterate `i=0..2` over dimensions `ioff`, `joff`, `koff`. Track max dimension (`dmax`) and its axis index (`spax`) [S1, unofficial].
    c. **Base Case (Single Voxel):** If `dmax == 1`:
        i. Call `r3d_reduce` on the poly to get moments [S1, unofficial].
        ii. Adjust moments for box offset if `polyorder >= 1` or `2` [S1, unofficial].
        iii. Compute grid index using `gind(ii, jj, kk, mm) = nmom * ((ii - ibox[0].i) * gridsz.j * gridsz.k + (jj - ibox[0].j) * gridsz.k + (kk - ibox[0].k)) + mm` [S1, unofficial].
        iv. Add moments to `dest_grid` at the voxel position `ibox[0]` [S1, unofficial].
    d. **Recursive Step (Split):** If `dmax > 1`:
        i. Split the poly along axis `spax` at the midpoint of the current node dimension [S1, unofficial].
        ii. Store children in `stack[nstack]` and `stack[nstack+1]` [S1, unofficial].
        iii. Update Bounding Boxes: Copy node to child 1. Update child 0 dimensions by subtracting `(dmax - dmax/2)` [S1, unofficial].
        iv. Increment `nstack` to push child 1 [S1, unofficial].

**Formulas:**
- **Grid Index:** `gind(ii, jj, kk, mm) = nmom * ((ii - ibox[0].i) * gridsz.j * gridsz.k + (jj - ibox[0].j) * gridsz.k + (kk - ibox[0].k)) + mm` [S1, unofficial].
- **Moment Order:** Row-major ordering of moments. Sequence: `1`, `x`, `y`, `z`, `x^2`, `x*y`, `x*z`, `y^2`, `y*z`, `z^2`, `x^3`, `x^2*y`... [S1, unofficial].
- **Simplicial Decomposition Moments:** For each triangle with vertices `v0, v1, v2`:
    - `locvol = ONE_SIXTH * (-(v2.x*v1.y*v0.z) + v1.x*2.y*v0.z + v2.x*v0.y*v1.z - v0.x*v2.y*v1.z - v1.x*v0.y*v2.z + v0.x*v1.y*v2.z)` [S1, unofficial].
    - `moments[0] += locvol` [S1, unofficial].
    - If `polyorder >= 1`:
        - `moments[1] += locvol * 0.25 * (v0.x + v1.x + v2.x)` [S1, unofficial].
        - `moments[2] += locvol * 0.25 * (v0.y + v1.y + v2.y)` [S1, unofficial].
        - `moments[3] += locvol * 0.25 * (v0.z + v1.z + v2.z)` [S1, unofficial].
    - If `polyorder >= 2`:
        - `moments[4] += locvol * 0.1 * (v0.x*v0.x + v1.x*v1.x + v2.x*v2.x + v1.x*v2.x + v0.x*(v1.x + v2.x))` [S1, unofficial].
        - `moments[5] += locvol * 0.1 * (v0.y*v0.y + v1.y*v1.y + v2.y*v2.y + v1.y*v2.y + v0.y*(v1.y + v2.y))` [S1, unofficial].
        - `moments[6] += locvol * 0.1 * (v0.z*v0.z + v1.z*v1.z + v2.z*v2.z + v1.z*v2.z + v0.z*(v1.z + v2.z))` [S1, unofficial].
        - `moments[7] += locvol * 0.05 * (v2.x*v0.y + v2.x*v1.y + 2*v2.x*v2.y + v0.x*(2*v0.y + v1.y + v2.y) + v1.x*(v0.y + 2*v1.y + v2.y))` [S1, unofficial].
        - `moments[8] += locvol * 0.05 * (v2.y*v0.z + v2.y*v1.z + 2*v2.y*v2.z + v0.y*(2*v0.z + v1.z + v2.z) + v1.y*(v0.z + 2*v1.z + v2.z))` [S1, unofficial].
        - `moments[9] += locvol * 0.05 * (v2.x*v0.z + v2.x*v1.z + 2*v2.x*v2.z + v0.x*(2*v0.z + v1.z + v2.z) + v1.x*(v0.z + 2*v1.z + v2.z))` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def polyhedron_moment_integration(poly, node, polyorder, dest_grid, gridsz):
    # Initialize
    if poly is None or gridsz.i <= 0 or gridsz.j <= 0 or gridsz.k <= 0:
        return

    # Stack initialization
    stack = []
    stack.append({'poly': poly, 'node': node})
    nstack = 1
    
    nmom = ((polyorder + 1) * (polyorder + 2) * (polyorder + 3) // 6)
    
    while nstack > 0:
        nstack -= 1
        current = stack[nstack]
        poly = current['poly']
        node = current['node']
        
        # Find split axis
        dmax = 0
        spax = 0
        dims = [node.ioff, node.joff, node.koff]
        for i in range(3):
            if dims[i] > dmax:
                dmax = dims[i]
                spax = i
        
        if dmax == 1:
            # Base case: single voxel
            moments = r3d_reduce(poly, polyorder)
            
            # Adjust moments for box offset if polyorder >= 1 or 2 [G]
            if polyorder >= 1:
                xmin = (node.imin + 0.5) * 1.0
                ymin = (node.jmin + 0.5) * 1.0
                zmin = (node.kmin + 0.5) * 1.0
                moments[1] += xmin * moments[0]
                moments[2] += ymin * moments[0]
                moments[3] += zmin * moments[0]
            if polyorder >= 2:
                xmin = (node.imin + 0.5) * 1.0
                ymin = (node.jmin + 0.5) * 1.0
                zmin = (node.kmin + 0.5) * 1.0
                moments[4] += 2.0 * xmin * moments[1] + xmin * xmin * moments[0]
                moments[5] += 2.0 * ymin * moments[2] + ymin * ymin * moments[0]
                moments[6] += 2.0 * zmin * moments[3] + zmin * zmin * moments[0]
                moments[7] += xmin * moments[2] + ymin * moments[1] + xmin * ymin * moments[0]
                moments[8] += ymin * moments[3] + zmin * moments[2] + ymin * zmin * moments[0]
                moments[9] += xmin * moments[3] + zmin * moments[1] + xmin * zmin * moments[0]
            
            # Add to dest_grid
            ii = node.imin
            jj = node.jmin
            kk = node.kmin
            for mm in range(nmom):
                gind = nmom * ((ii) * gridsz.j * gridsz.k + (jj) * gridsz.k + (kk)) + mm
                dest_grid[gind] += moments[mm]
        else:
            # Recursive step: split
            split_coord = dmax // 2 # Integer division ensures voxel alignment [S1, unofficial]
            
            # Child 0 (lower half)
            child0_poly = split_poly_along_axis(poly, spax, split_coord) # Sources do not give the exact splitting algorithm [G]
            child0_node = {'imin': node.imin, 'jmin': node.jmin, 'kmin': node.kmin,
                           'ioff': node.ioff, 'joff': node.joff, 'koff': node.koff}
            child0_node['dims'][spax] -= (dmax - split_coord)
            
            # Child 1 (upper half)
            child1_poly = split_poly_along_axis(poly, spax, split_coord)
            child1_node = {'imin': node.imin, 'jmin': node.jmin, 'kmin': node.kmin,
                           'ioff': node.ioff, 'joff': node.joff, 'koff': node.koff}
            child1_node['dims'][spax] = split_coord
            child1_node['offsets'][spax] += split_coord
            
            stack.append(child1_poly, child1_node)
            stack.append(child0_poly, child0_node)
            nstack += 2

def r3d_reduce(poly, polyorder):
    moments = [0.0] * ((polyorder + 1) * (polyorder + 2) * (polyorder + 3) // 6)
    verts = poly.verts
    nverts = poly.nverts
    
    # Find first unclipped vertex
    vcur = -1
    for v in range(nverts):
        if not (verts[v].orient.fflags & 0xFF): # CLIP_MASK assumed as 0xFF
            vcur = v
            break
    
    if vcur == -1:
        return moments
    
    # Stack for edge traversal
    nvstack = 0
    vstack = []
    vstack.append(vcur)
    vstack.append(0)
    
    emarks = [[0] * 3 for _ in range(64)] # R3D_MAX_VERTS = 64
    
    while nvstack > 0:
        nvstack -= 1
        pnext = vstack.pop()
        vcur = vstack.pop()
        
        if emarks[vcur][pnext]:
            continue
        
        emarks[vcur][pnext] = 1
        vstart = vcur
        v0 = verts[vstart].pos
        vnext = verts[vcur].pnbrs[pnext]
        
        vstack.append(vcur)
        vstack.append((pnext + 1) % 3)
        
        # Move to second edge
        np = 0
        while np < 3 and verts[vnext].pnbrs[np] != vcur:
            np += 1
        vcur = vnext
        pnext = (np + 1) % 3
        emarks[vcur][pnext] = 1
        vnext = verts[vcur].pnbrs[pnext]
        
        vstack.append(vcur)
        vstack.append((pnext + 1) % 3)
        
        # Triangle fan
        while vnext != vstart:
            v2 = verts[vcur].pos
            v1 = verts[vnext].pos
            
            locvol = (1.0/6.0) * (-(v2.x*v1.y*v0.z) + v1.x*2.y*v0.z + v2.x*v0.y*v1.z - v0.x*v2.y*v1.z - v1.x*v0.y*v2.z + v0.x*v1.y*v2.z)
            
            moments[0] += locvol
            if polyorder >= 1:
                moments[1] += locvol * 0.25 * (v0.x + v1.x + v2.x)
                moments[2] += locvol * 0.25 * (v0.y + v1.y + v2.y)
                moments[3] += locvol * 0.25 * (v0.z + v1.z + v2.z)
            if polyorder >= 2:
                moments[4] += locvol * 0.1 * (v0.x*v0.x + v1.x*v1.x + v2.x*v2.x + v1.x*v2.x + v0.x*(v1.x + v2.x))
                moments[5] += locvol * 0.1 * (v0.y*v0.y + v1.y*v1.y + v2.y*v2.y + v1.y*v2.y + v0.y*(v1.y + v2.y))
                moments[6] += locvol * 0.1 * (v0.z*v0.z + v1.z*v1.z + v2.z*v2.z + v1.z*v2.z + v0.z*(v1.z + v2.z))
                moments[7] += locvol * 0.05 * (v2.x*v0.y + v2.x*v1.y + 2*v2.x*v2.y + v0.x*(2*v0.y + v1.y + v2.y) + v1.x*(v0.y + 2*v1.y + v2.y))
                moments[8] += locvol * 0.05 * (v2.y*v0.z + v2.y*v1.z + 2*v2.y*v2.z + v0.y*(2*v0.z + v1.z + v2.z) + v1.y*(v0.z + 2*v1.z + v2.z))
                moments[9] += locvol * 0.05 * (v2.x*v0.z + v2.x*v1.z + 2*v2.x*v2.z + v0.x*(2*v0.z + v1.z + v2.z) + v1.x*(v0.z + 2*v1.z + v2.z))
            
            # Move to next edge
            np = 0
            while np < 3 and verts[vnext].pnbrs[np] != vcur:
                np += 1
            vcur = vnext
            pnext = (np + 1) % 3
            emarks[vcur][pnext] = 1
            vnext = verts[vcur].pnbrs[pnext]
            
            vstack.append(vcur)
            vstack.append((pnext + 1) % 3)
            
    return moments
```

**Edge Cases and Pitfalls:**
- Integer division in split: The split coordinate uses integer division `dmax/2`. This may introduce slight inaccuracies in split position compared to floating point midpoint, but ensures integer voxel alignment [S1, unofficial].
- Stack Overflow Prevention: The stack size is calculated based on `log2` of grid dimensions, ensuring it never overflows for grids up to ~512^3 [S1, unofficial].
- All vertices clipped: If all vertices have been clipped, `r3d_reduce` returns immediately with zero moments [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ where $N$ is the number of voxels intersected by the polyhedron, due to recursive splitting [S1, unofficial].
- **Memory Complexity:** $O(\log (\text{grid dimensions}))$ for the stack, plus $O(\text{grid size} \times \text{moments per voxel})$ for the output grid [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales well with grid resolution due to logarithmic stack depth [S1, unofficial].
- **Strengths:** Exact analytic integration of polynomial density moments; supports arbitrary polynomial orders; efficient for convex polyhedra [S1, unofficial].
- **Limitations:** Limited to convex polyhedra; integer division may introduce slight inaccuracies; stack size limits grid dimensions [S1, unofficial].
- **When to use:** When exact polynomial density moments are required for convex polyhedra [S1, unofficial].
- **When not to use:** For non-convex polyhedra or when approximate binary occupancy is sufficient [S1, unofficial].

## Background
Not given in the sources.

## Sources
[S1] devonmpowell/r3d, https://github.com/devonmpowell/r3d, fetched 2026-10-27 (unofficial)
