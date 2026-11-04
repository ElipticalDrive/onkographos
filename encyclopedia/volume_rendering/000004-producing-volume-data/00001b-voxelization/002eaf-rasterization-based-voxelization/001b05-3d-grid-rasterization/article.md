# 3D Grid Rasterization

Also called: Grid-Snapped Voxelization, Surface Voxelize, Mesh Voxelizer, CPU Voxelization, CPUVoxelizer, CPU Intersect Scanline Voxelization, Voxelization from Triangle Mesh

## Overview
3D Grid Rasterization converts a triangular mesh into a discrete 3D occupancy grid (voxel array) by mapping mesh geometry to a fixed-resolution integer lattice [S1, S2, unofficial][S3][S6, unofficial]. It solves the problem of representing continuous surface geometry as a volume suitable for ray marching or other volume rendering techniques [G]. Builds on [Voxelization](12-voxelization.md).

## How it works
The method typically follows one of two primary implementations: a **Triangle-AABB Sampling** approach (common in C++ libraries like `voxelizer`) or a **Slice-and-Rasterize** approach (common in Python libraries like `mesh_vox`) [S2, S6, unofficial]. Both produce a 3D grid where each cell indicates occupancy and optionally color [S2, S6, unofficial].

### Inputs and Outputs
**Inputs:**
- A triangular mesh defined by vertices $V$ and indices $I$ [S2, unofficial][S3][S6, unofficial].
- Target grid dimensions: width $W$, height $H$, depth $D$ (or a single resolution $R$ for the largest dimension) [S2, unofficial][S3, S5][S6, unofficial].
- Optional: Vertex colors (RGB or RGBA) [S2, S4, unofficial].
- Optional: STL file path (for direct reading) [S3][S6, unofficial].

**Outputs:**
- A 3D array `voxels` of shape $(D, W, H)$ or $(Z, X, Y)$ [S3][S6, unofficial].
- Data type: Boolean (occupancy) or 32-bit integer (packed ABGR color) [S2, S6, unofficial].

### Data Structures

**1. AABB (Axis-Aligned Bounding Box)**
Used to cull triangles.
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class AABB:
    min: Vector3  # Minimum coordinates (x, y, z)
    max: Vector3  # Maximum coordinates (x, y, z)
```

**2. Triangle**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Triangle:
    p1: Vector3
    p2: Vector3
    p3: Vector3
    color: Optional[Vector4]  # RGBA, if present
```

**3. Voxel Grid (C++ Style)**
Flat array of unsigned integers (ABGR packed) [S2, unofficial].
Layout: `index = x + y * W + z * (W * H)`.
Size: $W \times H \times D$.

**4. Voxel Grid (Python Style)**
Boolean array [S6, unofficial].
Shape: `(Z, X, Y)`.
Addressed as `voxels[z][x][y]`.

**5. STL Binary Record (for input parsing)**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
stl_record_dtype = [
    ('normals', np.float32, (3,)),
    ('Vertex1', np.float32, (3,)),
    ('Vertex2', np.float32, (3,)),
    ('Vertex3', np.float32, (3,)),
    ('atttr', np.int16, (1,))
]
```

### Algorithm: Slice-and-Rasterize (Python Style)
This method slices the mesh at each height $z$, computes 2D intersection lines, and rasterizes them into a 2D grid [S6, unofficial].

**Steps:**

1.  **Read Mesh:** Load STL file (binary or ASCII) [S6, unofficial].
2.  **Scale and Shift:**
    - Find min/max coordinates of all vertices [S6, unofficial].
    - Compute `xyscale = (resolution - 1) / max(width, depth)` [S6, unofficial].
    - `shift = -min`.
    - `scale = [xyscale, xyscale, xyscale]`.
    - Transform vertices: $v' = (v + shift) \times scale$ [S6, unofficial].
    - Filter degenerate triangles (fewer than 3 unique points after transformation) [S6, unofficial].
3.  **Initialize Voxel Array:** Boolean array `voxels` of shape `(bounding_box[2], bounding_box[0], bounding_box[1])` [S6, unofficial].
4.  **Iterate Height $z$:**
    a. **Find Intersecting Triangles:** Filter triangles that cross the plane $z$ [S6, unofficial].
       - A triangle crosses if:
         - It has vertices both above ($z_v > z$) and below ($z_v < z$) [S6, unofficial].
         - OR it has 2 or 3 vertices exactly on the plane ($z_v == z$) [S6, unofficial].
       - Exclude triangles lying entirely on the plane [S6, unofficial].
    b. **Compute Intersection Lines:**
       - For each valid triangle, compute the intersection line segment with the plane $z$ [S6, unofficial].
       - Cases:
         - 2 vertices on plane: Return those 2 vertices [S6, unofficial].
         - 1 vertex on plane: Compute intersection point on the edge between the above/below vertex and the on-plane vertex [S6, unofficial]. Return (intersection, on-plane vertex).
         - 0 vertices on plane: Compute intersection points on the two edges crossing the plane [S6, unofficial]. Return those 2 points.
       - Intersection point calculation:
         - Ensure $p_1.z < p_2.z$.
         - If $p_1.z == p_2.z$, distance = 0.
         - Else, $t = \frac{z - p_1.z}{p_2.z - p_1.z}$ [S6, unofficial].
         - $point = p_1 + t \times (p_2 - p_1)$ [S6, unofficial].
    c. **Rasterize Lines to 2D Grid:**
       - Initialize `prepixel` grid of size `(bounding_box[0], bounding_box[1])` to False [S6, unofficial].
       - For each $x$ column:
         - Find relevant lines crossing this $x$. A line is relevant if it spans across $x$ (vertices on both sides) or one vertex is on $x$ and another is above [S6, unofficial].
         - Compute target $y$ coordinates for each line at this $x$:
           - If vertical ($x_0 == x_1$), return -1 (skip) [S6, unofficial].
           - Else, $ratio = \frac{x - x_0}{x_1 - x_0}$ [S6, unofficial].
           - $y_{new} = y_0 + ratio \times (y_1 - y_0)$ [S6, unofficial].
         - For each $y$ row:
           - If `isBlack` is True, set `prepixel[x][y] = True` [S6, unofficial].
           - If $y$ matches a target $y$ coordinate AND the point lies on the line segment (checked by integer comparison and range checks), toggle `isBlack` and set `prepixel[x][y] = True` [S6, unofficial].
       - Assign `prepixel` to `voxels[z]` [S6, unofficial].
5.  **Padding (Optional):** If padding is enabled, expand dimensions by 2 in each axis and shift data by 1 [S6, unofficial].

**Pseudocode (Slice-and-Rasterize):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def voxelize_slice_and_rasterize(mesh_triangles, resolution):
    # 1. Scale and Shift
    all_points = [pt for tri in mesh_triangles for pt in tri]
    mins = np.min(all_points, axis=0)
    maxs = np.max(all_points, axis=0)
    max_dim = max(maxs[0] - mins[0], maxs[1] - mins[1])
    if max_dim == 0: max_dim = 1e-9
    xyscale = (resolution - 1) / max_dim
    shift = -mins
    scale = np.array([xyscale, xyscale, xyscale])
    
    scaled_triangles = []
    for tri in mesh_triangles:
        new_tri = [(np.array(pt) + shift) * scale for pt in tri]
        if len(set(map(tuple, new_tri))) == 3: # Filter degenerate
            scaled_triangles.append(new_tri)
            
    z_max = int(np.ceil((maxs[2] - mins[2]) * xyscale))
    x_max, y_max = resolution, resolution
    voxels = np.zeros((z_max, x_max, y_max), dtype=bool)
    
    # 2. Iterate Height
    for z in range(z_max):
        # Find crossing triangles
        crossing_tris = []
        for tri in scaled_triangles:
            zs = [pt[2] for pt in tri]
            has_above = any(z_v > z for z_v in zs)
            has_below = any(z_v < z for z_v in zs)
            on_plane = sum(1 for z_v in zs if abs(z_v - z) < 1e-9)
            if (has_above and has_below) or on_plane >= 2:
                crossing_tris.append(tri)
                
        # Compute intersection lines
        lines = []
        for tri in crossing_tris:
            p1, p2, p3 = tri
            pts = [p1, p2, p3]
            above = [p for p in pts if p[2] > z]
            below = [p for p in pts if p[2] < z]
            same = [p for p in pts if abs(p[2] - z) < 1e-9]
            
            if len(same) == 2:
                lines.append((same[0], same[1]))
            elif len(same) == 1:
                if above and below:
                    side = where_line_crosses_z(above[0], below[0], z)
                    lines.append((side, same[0]))
            else:
                for i in range(3):
                    pa, pb = pts[i], pts[(i+1)%3]
                    if (pa[2] > z) != (pb[2] > z):
                        inter = where_line_crosses_z(pa, pb, z)
                        lines.append(inter)
                        
        # Rasterize to 2D grid
        prepixel = np.zeros((x_max, y_max), dtype=bool)
        for x in range(x_max):
            is_black = False
            for y in range(y_max):
                if is_black:
                    prepixel[x, y] = True
                for line in lines:
                    if on_line_at_x(line, x, y):
                        is_black = not is_black
                        prepixel[x, y] = True
            if is_black:
                raise ValueError(f"Unbalanced edge crossings at z={z}, x={x}")
        voxels[z] = prepixel
    return voxels

def where_line_crosses_z(p1, p2, z):
    if p1[2] > p2[2]: p1, p2 = p2, p1
    dist = 0 if abs(p2[2] - p1[2]) < 1e-9 else (z - p1[2]) / (p2[2] - p1[2])
    return (p1[0] - dist * (p1[0] - p2[0]), p1[1] - dist * (p1[1] - p2[1]))

def on_line_at_x(line, x, y):
    p1, p2 = line
    if p1[0] == p2[0]: return False # Vertical lines skip
    ratio = (x - p1[0]) / (p2[0] - p1[0])
    newy = p1[1] + ratio * (p2[1] - p1[1])
    return int(newy) == y and min(p1[0],p2[0]) <= x <= max(p1[0],p2[0]) and min(p1[1],p2[1]) <= y <= max(p1[1],p2[1])
```

### Algorithm: Triangle-AABB Sampling (C++ Style)
This method iterates over triangles, computes their bounding box in voxel space, and tests each voxel in that box for intersection [S2, unofficial][S5].

**Steps:**

1.  **Initialize Grid:** Allocate a flat array `data` of size $W \times H \times D$, initialized to 0 [S2, S4, unofficial].
2.  **Compute Mesh AABB:** Iterate over all triangles to compute the global bounding box `meshaabb` of the mesh [S2, S4, unofficial].
3.  **Compute Resolution per Axis:**
    $$
    res_x = \frac{meshaabb.max.x - meshaabb.min.x}{W}
    $$
    $$
    res_y = \frac{meshaabb.max.y - meshaabb.min.y}{H}
    $$
    $$
    res_z = \frac{meshaabb.max.z - meshaabb.min.z}{D}
    $$
4.  **Voxelize Points (Optional Pre-step):** Some implementations first convert the mesh to a point cloud using `vx_voxelize_pc` with the computed resolutions, then map points to the grid [S2, unofficial].
5.  **Iterate Triangles:** For each triangle $T$ with vertices $v_1, v_2, v_3$:
    a. Compute $T$'s AABB in world space.
    b. Map the AABB min/max to voxel grid indices [S2, unofficial].
       - Compute offset: $ox = \text{tri\_min}.x + \text{fabs}(aabb.min.x)$ [S2, unofficial].
       - Index: $ix = \frac{ox}{ax} \times (W - 1)$ where $ax = aabb.max.x - aabb.min.x$ [S2, unofficial].
       - Clamp indices to $[0, W-1]$ [S5].
    c. Iterate over voxel indices $(x, y, z)$ within the mapped AABB range [S2, unofficial][S5].
    d. **Intersection Test:** For each voxel, construct its AABB (centered at grid point, size $res_x, res_y, res_z$) [S2, unofficial]. Test for intersection with triangle $T$ using `vx__triangle_box_overlap` [S4, unofficial].
    e. **Color Interpolation (if colors present):**
       If intersecting, compute the color at the voxel center $C_{voxel}$ using barycentric interpolation [S4, unofficial].
       - Create sub-triangles $t_1(v_1, v_2, C_{voxel})$, $t_2(v_2, v_3, C_{voxel})$, $t_3(v_3, v_1, C_{voxel})$ [S4, unofficial].
       - Compute areas $A_1, A_2, A_3$.
       - Total Area $A = A_1 + A_2 + A_3$.
       - Weighted Color $C = \frac{C_1 A_2 + C_2 A_3 + C_3 A_1}{A}$ [S4, unofficial].
    f. **Insertion:**
       - Compute flat index $idx = x + y \cdot W + z \cdot (W \cdot H)$ [S2, unofficial].
       - If `data[idx]` is 0, set `data[idx] = Color` [S2, unofficial].
       - If `data[idx]` is non-zero, blend: `data[idx] = mix(data[idx], Color)` (average of existing and new color) [S2, unofficial].

**Pseudocode (Triangle-AABB Sampling):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def voxelize_aabb_sampling(vertices, indices, width, height, depth, colors=None):
    grid = np.zeros((width, height, depth), dtype=np.uint32)
    
    # Compute mesh AABB
    min_v = np.min(vertices, axis=0)
    max_v = np.max(vertices, axis=0)
    ax, ay, az = max_v - min_v
    
    # Handle zero dimensions
    res_x = ax[0] / width if ax[0] > 0 else 1e-9
    res_y = ax[1] / height if ax[1] > 0 else 1e-9
    res_z = ax[2] / depth if ax[2] > 0 else 1e-9
    
    for i in range(0, len(indices), 3):
        i1, i2, i3 = indices[i], indices[i+1], indices[i+2]
        p1, p2, p3 = vertices[i1], vertices[i2], vertices[i3]
        
        tri_min = np.minimum(np.minimum(p1, p2), p3)
        tri_max = np.maximum(np.maximum(p1, p2), p3)
        
        # Map to voxel indices (S2 formula)
        ox = tri_min[0] + abs(min_v[0])
        ix_min = int((ox / ax[0]) * (width - 1)) if ax[0] > 0 else 0
        ix_max = int(((tri_max[0] + abs(min_v[0])) / ax[0]) * (width - 1)) if ax[0] > 0 else 0
        # Clamp
        ix_min = max(0, min(ix_min, width - 1))
        ix_max = max(0, min(ix_max, width - 1))
        # Similar for y, z...
        
        for z in range(z_min, z_max + 1):
            for y in range(y_min, y_max + 1):
                for x in range(x_min, x_max + 1):
                    # Intersection test placeholder [G]
                    intersects = True 
                    if intersects:
                        # Color interpolation (S4)
                        if colors is not None:
                            c1, c2, c3 = colors[i1], colors[i2], colors[i3]
                            center = np.array([x, y, z]) * np.array([res_x, res_y, res_z]) + min_v
                            a1 = tri_area(p1, p2, center)
                            a2 = tri_area(p2, p3, center)
                            a3 = tri_area(p3, p1, center)
                            total = a1 + a2 + a3
                            if total > 0:
                                r = (c1[0]*a2 + c2[0]*a3 + c3[0]*a1) / total
                                g = (c1[1]*a2 + c2[1]*a3 + c3[1]*a1) / total
                                b = (c1[2]*a2 + c2[2]*a3 + c3[2]*a1) / total
                            else: r,g,b = 0,0,0
                        else: r,g,b = 1,1,1
                        
                        color_int = ((int(255)&0xff)<<24) | ((int(b*255)&0xff)<<16) | ((int(g*255)&0xff)<<8) | ((int(r*255)&0xff)<<0)
                        idx = x + y * width + z * (width * height)
                        if grid[idx] == 0: grid[idx] = color_int
                        else:
                            # Blend
                            old = grid[idx]
                            new_r = ((old>>0)&0xff + int(r*255))//2
                            new_g = ((old>>8)&0xff + int(g*255))//2
                            new_b = ((old>>16)&0xff + int(b*255))//2
                            grid[idx] = (255<<24) | (new_b<<16) | (new_g<<8) | new_r
    return grid
```

### Parameters
- **Resolution:** Number of voxels for the largest dimension. Typical value: 100 [S6, unofficial].
- **Width, Height, Depth:** Explicit grid dimensions [S2, S4, unofficial].
- **Padding:** Boolean flag (default True). Adds a 1-voxel border of empty space by expanding dimensions by 2 in each axis [S6, unofficial].

### Edge Cases and Pitfalls
- **Degenerate Triangles:** Triangles with collinear or duplicate vertices are filtered out during scaling [S6, unofficial].
- **Division by Zero:** Grid dimensions of zero are handled by ternary checks (e.g., `ax == 0.0 ? 0 : ...`) [S2, unofficial].
- **Vertical Lines:** In slice rasterization, vertical lines ($x_0 == x_1$) return $Y=-1$, which skips them in scanline fill [S6, unofficial].
- **Unbalanced Edges:** If `isBlack` is still True after processing all $y$ rows, an error is printed, indicating unbalanced edge crossings [S6, unofficial].
- **Precision:** Floating-point comparisons for "on plane" or "on line" should use a small epsilon to avoid missing intersections due to precision errors [G].

## Subtopics

### [A-Buffer Solid Voxelization](1189-a-buffer-solid-voxelization.md)
A GPU-based solid voxelization technique that outputs a linked list of starting and ending voxels for each row and column in the voxel grid. It uses an A-buffer approach to handle multiple fragments per pixel by building and sorting linked lists of fragment data. Choose this when you need depth-sorted solid voxels for accurate interior rendering.

## Operating characteristics
Not given in the sources.

## Background
3D Grid Rasterization is a fundamental technique in voxelization, originating from early computer graphics research on converting polygonal models to volume data. It is widely used in volume rendering pipelines, particularly in differentiable rendering and medical imaging. The specific implementations cited here are from open-source libraries: `differentiable_volumetric_rendering` [S1, unofficial], `voxelizer` [S2, unofficial], and `mesh_vox` [S6, unofficial].

## Sources
[S1] yuyujunjun/mesh-voxelize, https://github.com/yuyujunjun/mesh-voxelize, fetched 2026-10-26 (unofficial)
[S2] mesh voxelizer library, https://github.com/karimnaaji/voxelizer, fetched 2026-10-26 (unofficial)
[S3] Voxelization — Open3D 0.12.0 documentation, https://www.open3d.org/docs/0.12.0/tutorial/geometry/voxelization.html, fetched 2026-10-26
[S4] mesh voxelizer library, https://github.com/karimnaaji/voxelizer, fetched 2026-10-26 (unofficial)
[S5] Real-Time GPU-Based Voxelizer - GitHub Pages, https://freder.github.io/UnityGraphicsProgrammingBook1/html-translated/vol2/Chapter%201%20_%20Real-Time%20GPU-Based%20Voxelizer.html, fetched 2026-10-27
[S6] GitHub - Septaris/mesh_vox: Convert 3D models to voxels, https://github.com/Septaris/mesh_vox, fetched 2026-10-30 (unofficial)
