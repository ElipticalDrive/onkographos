# Domain Search Isosurface Extraction

Also called: Tetrahedral Domain Search, Case-based Tetrahedral Marching

## Overview
Domain Search Isosurface Extraction is a surface extraction method that operates on a volumetric grid decomposed into tetrahedra. It determines the isosurface by evaluating the scalar field at the four vertices of each tetrahedron, classifying the cell into one of 16 topological cases based on which vertices lie below a specified isovalue, and generating intersection points via linear interpolation. It differs from standard Marching Cubes by using a tetrahedral decomposition, which avoids the ambiguity issues associated with cube cells, and from dual contouring by producing a triangulated mesh directly from interpolation rather than optimizing vertex positions via quadratic error functions.

## How it works

### Inputs and Outputs
**Inputs:**
*   `x_axis`, `y_axis`, `z_axis`: Integer dimensions of the volumetric grid. Typical value: 256, 256, 256 [S1, unofficial].
*   `scalar_field`: A data structure providing scalar values for grid vertices. In the reference implementation, this is accessed via an array `col` storing values for the current tetrahedron's 4 vertices [S1, unofficial].
*   `Vertices`: An array of `Vector3f` containing the 3D coordinates of the grid vertices. The reference implementation uses a global array `Vertices` indexed by vertex ID [S1, unofficial].
*   `isolevel`: The scalar threshold value defining the surface [S1, unofficial].

**Outputs:**
*   `newvert`: An array of `Vector3f` storing the extracted isosurface vertices. Size: $256 \times 256 \times 400$ in the reference implementation [S1, unofficial].
*   `index`: A buffer index (integer `p`) tracking the current write position in `newvert` [S1, unofficial].

### Data Structures
*   **`col`**: A vector or array of 4 integers/floats storing the scalar field values for the four vertices of the current tetrahedron being processed [S1, unofficial].
*   **`Vertices`**: An array of `Vector3f` (x, y, z coordinates) representing the spatial positions of the grid vertices. The reference implementation defines a maximum size of $2^{24}$ (16,777,216) vertices [S1, unofficial].
*   **`newvert`**: An array of `Vector3f` where generated surface vertices are stored sequentially. The reference implementation pre-allocates this with size $256 \times 256 \times 400$ [S1, unofficial].
*   **`index`**: An 8-bit unsigned integer (mask) representing the topology case of the current tetrahedron. Bit $i$ is set if vertex $i$ is inside the isosurface (value < isolevel) [S1, unofficial].

### Algorithm Steps

1.  **Tetrahedron Processing**: Iterate through the volume, processing each tetrahedron defined by four vertices with indices $a, b, c, d$ and corresponding scalar values $col[0], col[1], col[2], col[3]$.
2.  **Case Classification**: Compute the case index `index` by checking each vertex's scalar value against the `isolevel`.
    *   Initialize `index = 0`.
    *   For $i$ from 0 to 3: if $col[i] < isolevel$, set bit $i$ in `index`: `index |= (1 << i)` [S1, unofficial].
3.  **Boundary Check**: Before generating vertices, check if the current output buffer pointer `p` allows for the maximum possible vertex generation for the current case. The reference implementation uses a guard: `if p < (ind - 20)` [S1, unofficial]. If this condition fails, skip vertex generation to prevent buffer overflow.
4.  **Case Handling and Vertex Generation**: Execute the logic corresponding to the 16 possible values of `index` (0x00 to 0x0F).
    *   **Empty (0x00) and Full (0x0F)**: No surface intersects the tetrahedron. Do nothing [S1, unofficial].
    *   **Single Vertex Inside (0x01, 0x02, 0x04, 0x08)**: Generate 3 new vertices. Each new vertex is the intersection of the isosurface with an edge connecting the single inside vertex to one of the three outside vertices.
        *   Example (0x01, vertex 0 inside): Interpolate edges $(0,1)$, $(0,3)$, and $(0,2)$. Store results in `newvert[p]`, `newvert[p+1]`, `newvert[p+2]`. Increment `p` by 3 [S1, unofficial].
    *   **Two Vertices Inside (0x03, 0x05, 0x06, 0x09, 0x0A, 0x0C)**: Generate 6 new vertices forming two triangles. The edges intersected depend on the specific pair of inside vertices.
        *   Example (0x03, vertices 0,1 inside): Interpolate edges $(0,3)$, $(0,2)$, $(1,3)$, $(0,2)$, $(2,1)$, $(1,3)$. Note: The reference implementation generates 6 vertices for these cases, effectively creating two triangles. Increment `p` by 6 [S1, unofficial].
    *   **Three Vertices Inside (0x07, 0x0B, 0x0D, 0x0E)**: These are the complements of the single-vertex cases. The reference implementation notes "code continues in next part" for some cases, implying symmetry or explicit handling. Generally, these generate 3 vertices by intersecting the single outside vertex with the three inside vertices.
5.  **Interpolation**: For each edge intersection, calculate the position using linear interpolation between the two endpoint vertices.

### Interpolation Formula
Given two vertices with positions $P_1=(x_1, y_1, z_1)$ and $P_2=(x_2, y_2, z_2)$, and scalar values $a$ and $b$ respectively, the intersection point $P$ at `isolevel` is calculated component-wise. For a component $x$:

$$
\text{if } a == b: \quad x = \frac{x_1 + x_2}{2.0}
$$

$$
\text{else}: \quad \text{interp} = \left| \frac{\text{isolevel} - a}{b - a} \right|, \quad \text{oneMinusInterp} = 1 - \text{interp}
$$

$$
x = x_1 \cdot \text{oneMinusInterp} + x_2 \cdot \text{interp}
$$

The same logic applies to $y$ and $z$ components [S1, unofficial].

### Pseudocode

```python
def interpolate(x1, y1, z1, x2, y2, z2, val1, val2, isolevel):
    """
    Calculates the intersection point of the isosurface with the edge
    connecting (x1,y1,z1) with value val1 and (x2,y2,z2) with value val2.
    """
    if val1 == val2:
        # Guard against division by zero; use midpoint
        return (x1 + x2) / 2.0, (y1 + y2) / 2.0, (z1 + z2) / 2.0
    
    # Linear interpolation factor
    # Note: Source uses abs(), though standard linear interpolation 
    # typically uses (isolevel - val1) / (val2 - val1). 
    # The source code: interp = abs((isolevel - a) / (b - a))
    interp = abs((isolevel - val1) / (val2 - val1))
    one_minus_interp = 1.0 - interp
    
    x = x1 * one_minus_interp + x2 * interp
    y = y1 * one_minus_interp + y2 * interp
    z = z1 * one_minus_interp + z2 * interp
    
    return x, y, z

def process_tetrahedron(vertices, col, isolevel, newvert, p, ind):
    """
    Processes a single tetrahedron defined by 4 vertex indices in 'vertices'
    and their scalar values in 'col'.
    
    vertices: list of 4 Vector3f objects (or tuples)
    col: list of 4 scalar values
    isolevel: float
    newvert: list of Vector3f (output buffer)
    p: integer (current index in newvert)
    ind: integer (total capacity of newvert)
    
    Returns: new_p (updated index)
    """
    # 1. Case Classification
    index = 0
    for i in range(4):
        if col[i] < isolevel:
            index |= (1 << i)
            
    # 2. Boundary Check
    # Reference uses p < (ind - 20) to prevent overflow
    if p >= (ind - 20):
        return p
        
    # Helper to get vertex coordinates
    def get_coords(idx):
        return vertices[idx].x, vertices[idx].y, vertices[idx].z

    # 3. Case Handling
    # Note: The reference implementation explicitly lists cases 0x00, 0x0F, 
    # 0x01-0x08, 0x03, 0x05, 0x09, 0x06, 0x0C. 
    # Cases 0x07, 0x0B, 0x0D, 0x0E (3 vertices inside) and 0x0A (1,3 inside) 
    # are partially shown or implied. 
    # We implement the explicit cases shown in the source.
    
    if index == 0x00 or index == 0x0F:
        # All inside or all outside
        pass
        
    elif index == 0x01:
        # Only vertex 0 inside
        # Interpolate edges (0,1), (0,3), (0,2)
        v0 = vertices[0]
        v1 = vertices[1]
        v2 = vertices[2]
        v3 = vertices[3]
        
        x, y, z = interpolate(v0.x, v0.y, v0.z, v1.x, v1.y, v1.z, col[0], col[1], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v0.x, v0.y, v0.z, v3.x, v3.y, v3.z, col[0], col[3], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v0.x, v0.y, v0.z, v2.x, v2.y, v2.z, col[0], col[2], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
    elif index == 0x02:
        # Only vertex 1 inside
        v0 = vertices[0]
        v1 = vertices[1]
        v2 = vertices[2]
        v3 = vertices[3]
        
        x, y, z = interpolate(v1.x, v1.y, v1.z, v0.x, v0.y, v0.z, col[1], col[0], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v1.x, v1.y, v1.z, v2.x, v2.y, v2.z, col[1], col[2], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v1.x, v1.y, v1.z, v3.x, v3.y, v3.z, col[1], col[3], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
    elif index == 0x04:
        # Only vertex 2 inside
        v0 = vertices[0]
        v1 = vertices[1]
        v2 = vertices[2]
        v3 = vertices[3]
        
        x, y, z = interpolate(v2.x, v2.y, v2.z, v0.x, v0.y, v0.z, col[2], col[0], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v2.x, v2.y, v2.z, v3.x, v3.y, v3.z, col[2], col[3], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v2.x, v2.y, v2.z, v1.x, v1.y, v1.z, col[2], col[1], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
    elif index == 0x08:
        # Only vertex 3 inside
        v0 = vertices[0]
        v1 = vertices[1]
        v2 = vertices[2]
        v3 = vertices[3]
        
        x, y, z = interpolate(v3.x, v3.y, v3.z, v1.x, v1.y, v1.z, col[3], col[1], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v3.x, v3.y, v3.z, v2.x, v2.y, v2.z, col[3], col[2], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v3.x, v3.y, v3.z, v0.x, v0.y, v0.z, col[3], col[0], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
    elif index == 0x03:
        # Vertices 0, 1 inside
        v0 = vertices[0]
        v1 = vertices[1]
        v2 = vertices[2]
        v3 = vertices[3]
        
        # Edges: (0,3), (0,2), (1,3), (0,2), (2,1), (1,3)
        # Note: Source generates 6 vertices.
        
        x, y, z = interpolate(v0.x, v0.y, v0.z, v3.x, v3.y, v3.z, col[0], col[3], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v0.x, v0.y, v0.z, v2.x, v2.y, v2.z, col[0], col[2], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v1.x, v1.y, v1.z, v3.x, v3.y, v3.z, col[1], col[3], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v0.x, v0.y, v0.z, v2.x, v2.y, v2.z, col[0], col[2], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v2.x, v2.y, v2.z, v1.x, v1.y, v1.z, col[2], col[1], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v1.x, v1.y, v1.z, v3.x, v3.y, v3.z, col[1], col[3], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
    elif index == 0x05:
        # Vertices 0, 2 inside
        v0 = vertices[0]
        v1 = vertices[1]
        v2 = vertices[2]
        v3 = vertices[3]
        
        # Edges: (0,1), (0,3), (1,2), (2,1), (0,3), (3,2)
        
        x, y, z = interpolate(v0.x, v0.y, v0.z, v1.x, v1.y, v1.z, col[0], col[1], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v0.x, v0.y, v0.z, v3.x, v3.y, v3.z, col[0], col[3], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v1.x, v1.y, v1.z, v2.x, v2.y, v2.z, col[1], col[2], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v2.x, v2.y, v2.z, v1.x, v1.y, v1.z, col[2], col[1], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v0.x, v0.y, v0.z, v3.x, v3.y, v3.z, col[0], col[3], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v3.x, v3.y, v3.z, v2.x, v2.y, v2.z, col[3], col[2], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
    elif index == 0x09:
        # Vertices 0, 3 inside
        v0 = vertices[0]
        v1 = vertices[1]
        v2 = vertices[2]
        v3 = vertices[3]
        
        # Edges: (0,1), (1,3), (0,2), (3,1), (2,3), (0,2)
        
        x, y, z = interpolate(v0.x, v0.y, v0.z, v1.x, v1.y, v1.z, col[0], col[1], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v1.x, v1.y, v1.z, v3.x, v3.y, v3.z, col[1], col[3], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v0.x, v0.y, v0.z, v2.x, v2.y, v2.z, col[0], col[2], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v3.x, v3.y, v3.z, v1.x, v1.y, v1.z, col[3], col[1], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v2.x, v2.y, v2.z, v3.x, v3.y, v3.z, col[2], col[3], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v0.x, v0.y, v0.z, v2.x, v2.y, v2.z, col[0], col[2], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
    elif index == 0x06:
        # Vertices 1, 2 inside
        v0 = vertices[0]
        v1 = vertices[1]
        v2 = vertices[2]
        v3 = vertices[3]
        
        # Edges: (0,1), (1,3), (0,2), (0,2), (1,3), (3,2)
        
        x, y, z = interpolate(v0.x, v0.y, v0.z, v1.x, v1.y, v1.z, col[0], col[1], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v1.x, v1.y, v1.z, v3.x, v3.y, v3.z, col[1], col[3], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v0.x, v0.y, v0.z, v2.x, v2.y, v2.z, col[0], col[2], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v0.x, v0.y, v0.z, v2.x, v2.y, v2.z, col[0], col[2], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v1.x, v1.y, v1.z, v3.x, v3.y, v3.z, col[1], col[3], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v3.x, v3.y, v3.z, v2.x, v2.y, v2.z, col[3], col[2], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
    elif index == 0x0C:
        # Vertices 2, 3 inside
        v0 = vertices[0]
        v1 = vertices[1]
        v2 = vertices[2]
        v3 = vertices[3]
        
        # Edges: (3,1), (0,3), (0,2), (0,2), (1,3), (1,2)
        
        x, y, z = interpolate(v3.x, v3.y, v3.z, v1.x, v1.y, v1.z, col[3], col[1], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v0.x, v0.y, v0.z, v3.x, v3.y, v3.z, col[0], col[3], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v0.x, v0.y, v0.z, v2.x, v2.y, v2.z, col[0], col[2], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v0.x, v0.y, v0.z, v2.x, v2.y, v2.z, col[0], col[2], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v1.x, v1.y, v1.z, v3.x, v3.y, v3.z, col[1], col[3], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
        x, y, z = interpolate(v1.x, v1.y, v1.z, v2.x, v2.y, v2.z, col[1], col[2], isolevel)
        newvert[p] = Vector3f(x, y, z); p += 1
        
    # Cases 0x07, 0x0B, 0x0D, 0x0E, 0x0A are not fully detailed in the provided source snippet.
    # In a complete implementation, these would be handled similarly, 
    # typically by symmetry or explicit edge intersection lists.
    
    return p

## Operating characteristics

**Time Complexity**: $O(N)$, where $N$ is the number of tetrahedra in the volume. Each tetrahedron is processed in constant time.
**Memory Complexity**: $O(V + E)$, where $V$ is the number of grid vertices and $E$ is the number of generated surface vertices. The reference implementation pre-allocates `newvert` with size $256 \times 256 \times 400$ [S1, unofficial].
**Performance**: The method is straightforward to implement and parallelize. However, the reference implementation uses a fixed-size buffer and explicit case handling, which may lead to memory inefficiency if the buffer is over-allocated or overflow if under-allocated.
**Strengths**:
*   Avoids the topological ambiguities of Marching Cubes by using tetrahedral decomposition.
*   Simple logic based on linear interpolation.
**Limitations**:
*   Requires tetrahedral decomposition of the grid, which adds preprocessing overhead.
*   The reference implementation's buffer management is rigid (`ind-20` guard) and may not scale well to arbitrary volumes without dynamic resizing.
*   Linear interpolation may result in less smooth surfaces compared to methods using higher-order interpolation or optimization.
*   The use of `abs()` in the interpolation factor calculation is non-standard for linear interpolation and may produce incorrect results if the isolevel is not between `a` and `b` (though the case classification ensures one vertex is inside and one is outside, so `isolevel` is between `a` and `b` if `a < isolevel <= b` or `b < isolevel <= a`). However, if `a` and `b` are on the same side, the case would not trigger an intersection. The `abs()` might be a safeguard or a specific design choice in the source [S1, unofficial].

## Background

The method is documented in an unofficial GitHub repository [S1, unofficial]. The source code includes a GNU General Public License header, indicating it is open-source software. The implementation appears to be a student or hobbyist project demonstrating tetrahedral isosurface extraction. The specific case handling and buffer management suggest a focus on correctness for a specific test case (e.g., a teapot, as hinted by `teapotColour` in the source) rather than a general-purpose library.

## Sources

[S1] yashwanth850/Domain-search-isosurface-extraction, https://github.com/yashwanth850/Domain-search-isosurface-extraction, fetched 2026-10-27 (unofficial)
