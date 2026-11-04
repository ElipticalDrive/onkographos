# Adaptive Octree Marching Tetrahedra


## Overview
An isosurface extraction method that uses an adaptive octree to subdivide a 3D volume only where the scalar field contains the target isovalue, applying the Marching Tetrahedra algorithm at leaf nodes to generate the mesh [S1, unofficial]. It differs from its parent [Marching Tetrahedrons](3720-marching-tetrahedrons.md) by introducing spatial adaptivity via octree subdivision, reducing computation in empty regions, and from [Naive Isosurface Extraction](4328-naive-isosurface-extraction.md) by avoiding uniform grid traversal [S1, unofficial].

## How it works
**Inputs:**
- A 3D scalar field stored in a 1D array `teapotColour` of size $X \times Y \times Z$ (source assumes fixed $256^3$) [S1, unofficial].
- Target isovalue `iso` [S1, unofficial].
- Volume bounds defined by `topLeftFront` and `bottomRightBack` points [S1, unofficial].

**Outputs:**
- A list of vertices (`newvert`) representing the isosurface mesh [S1, unofficial].

**Data Structures:**
- `Point`: Struct with fields `int x`, `int y`, `int z`. Default constructor initializes to $(-1, -1, -1)$ [S1, unofficial].
- `Octree` Node:
  - `Point* topLeftFront`: Minimum corner of the node's bounding box [S1, unofficial].
  - `Point* bottomRightBack`: Maximum corner of the node's bounding box [S1, unofficial].
  - `Point* point`: If `NULL`, the node is internal; if $(-1, -1, -1)$, the node is empty [S1, unofficial].
  - `vector<Octree*> children`: Array of 8 child pointers [S1, unofficial].

**Algorithm Steps:**

1. **Build Octree (`make` function):**
   - Calculate midpoints: `midx = (topLeftFront->x + bottomRightBack->x) / 2`, `midy = (topLeftFront->y + bottomRightBack->y) / 2`, `midz = (topLeftFront->z + bottomRightBack->z) / 2` [S1, unofficial].
   - Iterate through all voxels in the node's bounds to find `min_iso` and `max_iso` from `teapotColour` using index `z * 256 * 256 + y * 256 + x` [S1, unofficial].
   - If node dimensions are greater than 1 (checked by `(midx - topLeftFront->x) > 1 || (midy - topLeftFront->y) > 1 || (bottomRightBack->z - midz) > 1`), create 8 child octants [S1, unofficial].
   - Child indices (0-7) are defined by their bounding boxes relative to midpoints:
     - Child 0: `(x1,y1,z1)` to `(midx,midy,midz)`
     - Child 1: `(midx,y1,z1)` to `(x2,midy,midz)`
     - Child 2: `(x1,midy,z1)` to `(midx,y2,midz)`
     - Child 3: `(midx,midy,z1)` to `(x2,y2,midz)`
     - Child 4: `(x1,y1,midz)` to `(midx,midy,z2)`
     - Child 5: `(midx,y1,midz)` to `(x2,midy,z2)`
     - Child 6: `(x1,midy,midz)` to `(midx,y2,z2)`
     - Child 7: `(midx,midy,midz)` to `(x2,y2,z2)` [S1, unofficial].
   - Recursively call `make()` on children if their dimensions > 1 [S1, unofficial].

2. **Leaf Processing (`explore` function):**
   - When a leaf node is reached (dimensions equal 1, i.e., `(-topLeftFront->x + bottomRightBack->x) == 1 && ...`), identify the 8 vertices of the cube [S1, unofficial].
   - Indices into `teapotColour` (assuming $256^3$ grid):
     - `a` (BottomLeftFront): `topLeftFront->z*256*256 + topLeftFront->y*256 + topLeftFront->x`
     - `b` (BottomRightFront): `a + 1`
     - `c` (TopLeftFront): `a + 257`
     - `d` (BottomLeftBack): `a + 256`
     - `e` (TopLeftBack): `256*256 + a`
     - `f` (TopRightFront): `e + 1`
     - `g` (TopRightBack): `e + 257`
     - `h` (TopLeftBack? Note: Source labels `h` as TopLeftBack in comment but uses it in specific tetrahedra): `e + 256` [S1, unofficial].
   - Decompose the cube into 5 tetrahedra. For each tetrahedron, pass the scalar values of its 4 vertices to the `tetra` function [S1, unofficial].
     - Tet 1: Vertices `h, g, c, f`
     - Tet 2: Vertices `h, c, d, b`
     - Tet 3: Vertices `d, a, e, b`
     - Tet 4: Vertices `h, d, e, b`
     - Tet 5: Vertices `h, c, f, b` [S1, unofficial].

3. **Marching Tetrahedra Logic (`tetra` function):**
   - Determine the case index based on which vertices are inside the isosurface (`val > isolevel`) [S1, unofficial].
   - For each case, compute intersection points on the edges using linear interpolation:
     $$ \text{interpolate}(v1, v2, \text{isolevel}, col1, col2) = v1 + \frac{\text{isolevel} - col1}{col2 - col1} (v2 - v1) $$
   - Store resulting `Vector3f` in `newvert` array [S1, unofficial].
   - Example Case `0x0C` (vertices `d, b, a, c` inside? Source code implies specific mapping):
     - Intersect edge `d-b`: `interpolate(Vertices[d], Vertices[b], isolevel, col[3], col[1])`
     - Intersect edge `a-d`: `interpolate(Vertices[a], Vertices[d], isolevel, col[0], col[3])`
     - Intersect edge `a-c`: `interpolate(Vertices[a], Vertices[c], isolevel, col[0], col[2])` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def interpolate(v1, v2, isolevel, col1, col2):
    """Linear interpolation of vertex v1 and v2 based on scalar values col1 and col2."""
    if col2 == col1:
        return v1  # Avoid division by zero
    t = (isolevel - col1) / (col2 - col1)
    return v1 + t * (v2 - v1)

def get_index(x, y, z, width=256):
    """Get 1D index from 3D coordinates."""
    return z * width * width + y * width + x

def make_octree(top_left, bottom_right, teapot_color):
    """Recursively build the adaptive octree."""
    mid_x = (top_left.x + bottom_right.x) // 2
    mid_y = (top_left.y + bottom_right.y) // 2
    mid_z = (top_left.z + bottom_right.z) // 2
    
    # Check if subdivision is needed
    if (mid_x - top_left.x > 1 or mid_y - top_left.y > 1 or bottom_right.z - mid_z > 1):
        # Define 8 children bounds
        children_bounds = [
            (top_left, (mid_x, mid_y, mid_z)),
            ((mid_x, top_left.y, top_left.z), (bottom_right.x, mid_y, mid_z)),
            ((top_left.x, mid_y, top_left.z), (mid_x, bottom_right.y, mid_z)),
            ((mid_x, mid_y, top_left.z), (bottom_right.x, bottom_right.y, mid_z)),
            ((top_left.x, top_left.y, mid_z), (mid_x, mid_y, bottom_right.z)),
            ((mid_x, top_left.y, mid_z), (bottom_right.x, mid_y, bottom_right.z)),
            ((top_left.x, mid_y, mid_z), (mid_x, bottom_right.y, bottom_right.z)),
            ((mid_x, mid_y, mid_z), bottom_right)
        ]
        
        for bounds in children_bounds:
            child = OctreeNode(bounds[0], bounds[1])
            make_octree(bounds[0], bounds[1], teapot_color)
            # Add child to parent
    else:
        # Leaf node
        explore_leaf(top_left, bottom_right, teapot_color)

def explore_leaf(top_left, bottom_right, teapot_color):
    """Process a leaf node (1x1x1 cube) using Marching Tetrahedra."""
    # Calculate indices for the 8 vertices
    # Note: Source assumes fixed width 256
    a_idx = get_index(top_left.x, top_left.y, top_left.z)
    b_idx = a_idx + 1
    c_idx = a_idx + 257
    d_idx = a_idx + 256
    e_idx = 256*256 + a_idx
    f_idx = e_idx + 1
    g_idx = e_idx + 257
    h_idx = e_idx + 256
    
    # Get scalar values
    col = [
        teapot_color[h_idx], teapot_color[g_idx], teapot_color[c_idx], teapot_color[f_idx],
        teapot_color[d_idx], teapot_color[b_idx], teapot_color[a_idx], teapot_color[e_idx]
    ]
    
    # Define vertices coordinates
    # Assuming integer coordinates for simplicity based on source
    verts = [
        Point(top_left.x, top_left.y, top_left.z), # a
        Point(top_left.x+1, top_left.y, top_left.z), # b
        Point(top_left.x, top_left.y+1, top_left.z), # c
        Point(top_left.x, top_left.y, top_left.z+1), # d
        Point(top_left.x, top_left.y+1, top_left.z+1), # e
        Point(top_left.x+1, top_left.y+1, top_left.z), # f
        Point(top_left.x+1, top_left.y+1, top_left.z+1), # g
        Point(top_left.x, top_left.y, top_left.z+1) # h (Note: Source h is TopLeftBack, same as d? Source code ambiguity)
    ]
    
    # Decompose cube into 5 tetrahedra
    # Tet 1: h, g, c, f
    tetra([col[h_idx], col[g_idx], col[c_idx], col[f_idx]], [verts[h_idx], verts[g_idx], verts[c_idx], verts[f_idx]])
    # Tet 2: h, c, d, b
    tetra([col[h_idx], col[c_idx], col[d_idx], col[b_idx]], [verts[h_idx], verts[c_idx], verts[d_idx], verts[b_idx]])
    # Tet 3: d, a, e, b
    tetra([col[d_idx], col[a_idx], col[e_idx], col[b_idx]], [verts[d_idx], verts[a_idx], verts[e_idx], verts[b_idx]])
    # Tet 4: h, d, e, b
    tetra([col[h_idx], col[d_idx], col[e_idx], col[b_idx]], [verts[h_idx], verts[d_idx], verts[e_idx], verts[b_idx]])
    # Tet 5: h, c, f, b
    tetra([col[h_idx], col[c_idx], col[f_idx], col[b_idx]], [verts[h_idx], verts[c_idx], verts[f_idx], verts[b_idx]])

def tetra(scalars, vertices):
    """Apply Marching Tetrahedra to a single tetrahedron."""
    # Determine case index
    case = 0
    for i in range(4):
        if scalars[i] > isolevel:
            case |= (1 << i)
            
    # Handle cases
    if case == 0x0C: # Example case from source
        # Intersect edges d-b, a-d, a-c
        # Note: Source mapping is ambiguous, using generic edge intersection logic
        p1 = interpolate(vertices[3], vertices[1], isolevel, scalars[3], scalars[1])
        p2 = interpolate(vertices[0], vertices[3], isolevel, scalars[0], scalars[3])
        p3 = interpolate(vertices[0], vertices[2], isolevel, scalars[0], scalars[2])
        newverts.append(p1)
        newverts.append(p2)
        newverts.append(p3)
```

**Parameters:**
- `iso`: Target isovalue [S1, unofficial].
- Grid dimensions: `x_axis`, `y_axis`, `z_axis` (source uses fixed 256) [S1, unofficial].

**Edge Cases and Pitfalls:**
- Grid indexing assumes a fixed width of 256 for index calculation (`z * 256 * 256 + y * 256 + x`), which fails for volumes with different Y-axis dimensions [S1, unofficial].
- Vertex labeling ambiguity: Variable names `a` through `h` in `explore` do not directly correspond to the `Vertices[a]`...`Vertices[h]` used in the `tetra` switch cases in the provided snippet [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Origin: Unofficial implementation by yashwanth850 [S1, unofficial].

## Sources
[S1] yashwanth850/Domain-search-isosurface-extraction, https://github.com/yashwanth850/Domain-search-isosurface-extraction, fetched 2026-10-27 (unofficial)
