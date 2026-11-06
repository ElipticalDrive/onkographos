# Uniform Grid Construction

Also called: Grid Resolution and Object Partitioning, PBRT Uniform Grid Construction

## Overview
Uniform Grid Construction creates a spatial acceleration structure by partitioning scene objects into a regular 3D grid, allowing objects to overlap multiple cells if necessary [S1, unofficial]. It builds on [Grid Resolution Selection via Lambda](3270-grid-resolution-selection-via-lambda.md) to determine cell dimensions, then assigns objects to cells based on their bounding boxes. It is distinguished from hierarchical methods like [Hierarchical Grid Organization with Macrocells](5025-hierarchical-grid-organization-with-macrocells.md) by its flat structure and from sparse approaches like [Coarse Grid for Large Models](5481-coarse-grid-for-large-models.md) by its dense, regular allocation.

## How it works
**Inputs:**
- `N`: Total number of objects in the scene.
- `boxMin`: The minimum corner of the global axis-aligned bounding box (AABB) containing all objects, with coordinates $(x_{min}, y_{min}, z_{min})$.
- `boxSize`: The dimensions of the global AABB, with values $(x_{size}, y_{size}, z_{size})$.
- `objects`: A list of objects, each with an AABB defined by `minBound` and `maxBound`.
- `n`: Target average number of objects per cell.
- `MaxResolution`: User-specified maximum resolution per axis.

**Outputs:**
- `xResolution`, `yResolution`, `zResolution`: The number of cells along each axis.
- `grid`: A 3D array of lists (or linked lists) of size `xResolution` $\times$ `yResolution` $\times$ `zResolution`, where each cell contains a list of object references.

**Data Structures:**
- The grid is stored as a contiguous 3D array or flattened 1D array. Indices $(i, j, k)$ map to memory address $base + (i \cdot y_{res} \cdot z_{res} + j \cdot z_{res} + k) \cdot sizeof(list\_header)$.
- Each cell holds a dynamic list of object pointers.

**Parameters:**
- `n`: Typical value is $3^3 = 27$ [S1, unofficial].
- `MaxResolution`: Typical value is 64 [S1, unofficial].

**Formulas:**
1.  **Cell Size ($a$):**
    $$ a = \sqrt[3]{\frac{n \cdot x_{size} \cdot y_{size} \cdot z_{size}}{N}} $$
    Alternatively, following PBRT assumptions (cube of max dimension, $n=27$):
    $$ a = \frac{3 \cdot \max(x_{size}, y_{size}, z_{size})}{\sqrt[3]{N}} $$
    [S1, unofficial]

2.  **Resolution:**
    $$ x_{res} = \text{clamp}\left(\text{round}\left(\frac{x_{size}}{a}\right), 1, \text{MaxResolution}\right) $$
    $$ y_{res} = \text{clamp}\left(\text{round}\left(\frac{y_{size}}{a}\right), 1, \text{MaxResolution}\right) $$
    $$ z_{res} = \text{clamp}\left(\text{round}\left(\frac{z_{size}}{a}\right), 1, \text{MaxResolution}\right) $$
    *Note: The source text explicitly writes `ySize` for the z-resolution calculation, but context implies `zSize` [S1, unofficial].*

3.  **Grid Index for a Point $P(x_p, y_p, z_p)$:**
    $$ i = \frac{x_p - x_{min}}{x_{size}} \cdot x_{res} $$
    $$ j = \frac{y_p - y_{min}}{y_{size}} \cdot y_{res} $$
    $$ k = \frac{z_p - z_{min}}{z_{size}} \cdot z_{res} $$
    [S1, unofficial]

**Steps:**
1.  Calculate $a$ using the chosen formula.
2.  Compute $x_{res}, y_{res}, z_{res}$ using the resolution formula and clamp to $[1, \text{MaxResolution}]$.
3.  Initialize the grid array of size $x_{res} \times y_{res} \times z_{res}$ with empty lists.
4.  For each object in `objects`:
    a.  Determine the object's AABB min $(x_{obj\_min}, y_{obj\_min}, z_{obj\_min})$ and max $(x_{obj\_max}, y_{obj\_max}, z_{obj\_max})$.
    b.  Compute grid indices for the min corner: $(i_{min}, j_{min}, k_{min}) = \text{gridIndex}(\text{minBound})$.
    c.  Compute grid indices for the max corner: $(i_{max}, j_{max}, k_{max}) = \text{gridIndex}(\text{maxBound})$.
    d.  Clamp all indices to valid ranges:
        - If index $< 0$, set to 0.
        - If index $\ge$ resolution, set to resolution $- 1$.
    e.  Iterate $i$ from $i_{min}$ to $i_{max}$, $j$ from $j_{min}$ to $j_{max}$, and $k$ from $k_{min}$ to $k_{max}$.
    f.  Add the object reference to the list at `grid[i][j][k]`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def uniform_grid_construction(objects, box_min, box_size, n=27, max_resolution=64):
    x_size, y_size, z_size = box_size
    N = len(objects)
    
    # Calculate cell size 'a'
    # Using the general formula from source
    box_volume = x_size * y_size * z_size
    a = (n * box_volume / N) ** (1/3)
    
    # Calculate resolutions
    x_res = clamp(round(x_size / a), 1, max_resolution)
    y_res = clamp(round(y_size / a), 1, max_resolution)
    z_res = clamp(round(z_size / a), 1, max_resolution) # Source typo noted, corrected to z_size
    
    # Initialize grid
    grid = [[[] for _ in range(z_res)] for _ in range(y_res)]
    for i in range(x_res):
        grid[i] = [[[] for _ in range(z_res)] for _ in range(y_res)]
        
    # Helper for grid index
    def get_grid_index(point):
        x, y, z = point
        ix = int((x - box_min[0]) / x_size * x_res)
        iy = int((y - box_min[1]) / y_size * y_res)
        iz = int((z - box_min[2]) / z_size * z_res)
        
        # Clamp indices
        ix = max(0, min(ix, x_res - 1))
        iy = max(0, min(iy, y_res - 1))
        iz = max(0, min(iz, z_res - 1))
        return (ix, iy, iz)

    # Partition objects
    for obj in objects:
        min_bound = obj.min_bound
        max_bound = obj.max_bound
        
        i_min, j_min, k_min = get_grid_index(min_bound)
        i_max, j_max, k_max = get_grid_index(max_bound)
        
        for i in range(i_min, i_max + 1):
            for j in range(j_min, j_max + 1):
                for k in range(k_min, k_max + 1):
                    grid[i][j][k].append(obj)
                    
    return grid, x_res, y_res, z_res

def clamp(value, min_val, max_val):
    return max(min_val, min(value, max_val))
```

**Edge Cases and Pitfalls:**
- **Boundary Precision:** Rays entering from the maximum end may produce an index equal to the resolution (e.g., $x_{res}$), which is out of bounds. Rays entering from the minimum end may produce $-1$ due to floating-point errors. Clamping is mandatory [S1, unofficial].
- **Object Duplication:** Objects overlapping multiple cells are duplicated in each cell's list. This increases memory usage but simplifies traversal [S1, unofficial].

## Subtopics

### [Hierarchical Grid Organization with Macrocells](5025-hierarchical-grid-organization-with-macrocells.md)
Uses a hierarchy of grids to reduce memory overhead for large scenes. Choose when memory is constrained or scene density varies significantly.

### [Coarse Grid for Large Models](5481-coarse-grid-for-large-models.md)
Uses a lower-resolution grid to handle very large models efficiently. Choose when the scene is too large for a fine uniform grid.

## Operating characteristics
- **Time Complexity:** Construction is $O(N \cdot k)$, where $N$ is the number of objects and $k$ is the average number of cells an object overlaps. Traversal is $O(C + I)$, where $C$ is the number of cells intersected by the ray and $I$ is the number of objects in those cells.
- **Memory Complexity:** $O(x_{res} \cdot y_{res} \cdot z_{res} + N \cdot k)$. The grid structure itself is $O(V)$, and object references add overhead proportional to duplication.
- **Performance:** Higher resolution reduces the number of objects per cell (faster intersection tests) but increases memory overhead and the number of cells to traverse along a ray, which can counterbalance speed-ups [S1, unofficial].
- **Strengths:** Simple to implement, robust, and effective for scenes with uniform object distribution.
- **Limitations:** Inefficient for scenes with large empty spaces or highly clustered objects due to memory waste or excessive duplication.

## Background
The method originates from standard ray tracing acceleration techniques, popularized by the PBRT textbook [S1, unofficial]. It is a foundational technique in volume rendering and ray tracing for accelerating ray-object intersection tests.

## Sources
[S1] Uniform Grid Acceleration | Sumant Pattanaik | Observable, https://observablehq.com/@spattana/ray-uniformgrid-traversal, fetched 2026-10-25 (unofficial)
