# Polygon Rasterization (2D)
Also called: Edge Walking, Rasterization algorithm for polygons, Rasterization algorithms for triangles, Scan-line polygon rasterization, Scanline Triangle Rasterization, Triangle Filling

## Overview
This method rasterizes a 2D polygon into a P-Buffer (presence buffer) by decomposing the polygon into triangles and flipping presence bits for covered positions [S1, unofficial]. It differs from standard framebuffer rasterization by using a bit-flip parity mechanism to determine voxel inclusion, which is specifically designed for 3D voxelization workflows where the 2D projection serves as a slice or projection step [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- A 2D polygon defined by a list of vertices $V = \{v_1, v_2, ..., v_n\}$ in 2D space [S1, unofficial].
- A color value associated with the polygon [S1, unofficial].

**Outputs:**
- A P-Buffer: A 2D grid of bits where each bit represents a voxel's presence (1 if inside, 0 if outside) [S1, unofficial].
- A Framebuffer: The final colored voxels extracted from the P-Buffer [S1, unofficial].

### Data Structures
**P-Buffer:**
- A 2D array of bits [S1, unofficial].
- Dimensions: Corresponds to the minimal bounding box of the polygon [S1, unofficial].
- Value: `0` (empty) or `1` (filled) [S1, unofficial].
- Operation: Bitwise XOR (flip) during rasterization [S1, unofficial].

### Algorithm Steps

#### Phase 1: Polygon Decomposition
1. Compute the minimal bounding box of the polygon [S1, unofficial].
2. Compute the centroid $C$ of the polygon [S1, unofficial].
3. For each edge of the polygon:
   - Construct a triangle using the centroid $C$ and the current edge vertices [S1, unofficial].
   - Rasterize this triangle into the P-Buffer (see Phase 2) [S1, unofficial].
   - Repeat until all edges are processed [S1, unofficial].

#### Phase 2: Triangle Rasterization (Scanline)
The triangle rasterization uses a scanline approach with slope updates for edge walking [S1, unofficial]. An alternative integer-based traversal using error accumulation is also described [S2, unofficial].

**Preprocessing:**
1. Sort the triangle vertices $A, B, C$ by their y-coordinate such that $y_C \le y_B \le y_A$ (where $C$ is the bottom vertex) [S1, unofficial]. Let the sorted vertices be $v_1, v_2, v_3$ with coordinates $(x_1, y_1), (x_2, y_2), (x_3, y_3)$ [S2, unofficial].
2. Initialize scanline variables: $x_l = x_r = x_1$, $y = y_1$ [S1, unofficial].
3. Calculate slopes: $i_l = \frac{x_2 - x_1}{y_2 - y_1}$ (slope of left edge $v_1-v_2$), $i_r = \frac{x_3 - x_1}{y_3 - y_1}$ (slope of right edge $v_1-v_3$) [S1, unofficial].
4. Ensure $i_l \le i_r$. If $i_r < i_l$, swap $i_l$ and $i_r$, and swap $x_l$ and $x_r$ [S1, unofficial].

**First Half (Upper Triangle):**
1. Loop while $y < y_2$:
   - Flip the P-Buffer bits for the range $[x_l, x_r]$ at row $y$ [S1, unofficial].
   - Increment $y$.
   - Update $x_l \leftarrow x_l + i_l$, $x_r \leftarrow x_r + i_r$ [S1, unofficial].
2. If $y_1 == y_2$ (flat-top triangle), the first half loop does not execute. The algorithm proceeds directly to the second half logic [S2, unofficial].

**Second Half (Lower Triangle):**
1. If $y_1 == y_2$, set $y = y_1$. Otherwise, $y$ is already at $y_2$ from the first half [S2, unofficial].
2. If $y$ reaches $y_2$, update the slope for the edge connecting $v_2$ to $v_3$: $i_{new} = \frac{x_3 - x_2}{y_3 - y_2}$ [S1, unofficial].
3. Assign $i_{new}$ to the variable corresponding to the edge connected to $v_2$ and $v_3$. If a swap occurred in preprocessing, update $i_l$; otherwise, update $i_r$ [S1, unofficial].
4. Initialize error terms for horizontal traversal if using integer arithmetic: $dx_1 = |x_2 - x_1|$, $dy_1 = |y_2 - y_1|$, $dx_2 = |x_3 - x_1|$, $dy_2 = |y_3 - y_1|$, $e_1 = dx_1 \gg 1$, $e_2 = dx_2 \gg 1$, $t_{1x} = x_1$, $t_{2x} = x_1$ [S2, unofficial].
5. Loop while $y \le y_3$:
   - Determine $min_x = \min(t_{1x}, t_{2x})$ and $max_x = \max(t_{1x}, t_{2x})$ [S2, unofficial].
   - Flip P-Buffer bits for range $[min_x, max_x]$ at row $y$ [S2, unofficial].
   - Advance $t_{1x}$ and $t_{2x}$ using error accumulation: $e_1 \leftarrow e_1 + dy_1$, if $e_1 \ge dx_1$: $e_1 \leftarrow e_1 - dx_1$, $t_{1x} \leftarrow t_{1x} + \text{sign}(x_2-x_1)$; similarly for $e_2, t_{2x}$ [S2, unofficial].
   - Increment $y$.
   - If $y > y_3$, break [S2, unofficial].

**Note on Slope Updates:**
- When transitioning from the first half to the second half (at $y=y_2$), the slope for the edge connecting $v_2$ to $v_3$ must replace the previous slope for that side. The variable holding the slope for the edge connected to the middle vertex $v_2$ is updated [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rasterize_polygon_2d(polygon_vertices, color, p_buffer):
    """
    Rasterizes a 2D polygon into a P-Buffer using centroid decomposition.
    
    Args:
        polygon_vertices: List of (x, y) tuples.
        color: Color value to apply.
        p_buffer: 2D array of bits (0 or 1).
    """
    # Step 1: Compute centroid
    n = len(polygon_vertices)
    cx = sum(v[0] for v in polygon_vertices) / n
    cy = sum(v[1] for v in polygon_vertices) / n
    centroid = (cx, cy)
    
    # Step 2: Decompose into triangles and rasterize
    for i in range(n):
        v1 = polygon_vertices[i]
        v2 = polygon_vertices[(i + 1) % n]
        triangle = [centroid, v1, v2]
        rasterize_triangle_2d(triangle, p_buffer)
        
    # Step 3: Transfer to framebuffer
    for y in range(p_buffer.height):
        for x in range(p_buffer.width):
            if p_buffer[y][x] == 1:
                p_buffer[y][x] = color  # Store color in framebuffer

def rasterize_triangle_2d(triangle, p_buffer):
    """
    Rasterizes a single triangle into the P-Buffer using slope-based scanline.
    """
    # Sort vertices by y: C (bottom), B (middle), A (top)
    vertices = sorted(triangle, key=lambda v: v[1])
    C, B, A = vertices
    xC, yC = C
    xB, yB = B
    xA, yA = A
    
    xl = xr = xC
    y = yC
    
    # Calculate slopes
    il = (xA - xC) / (yA - yC) if yA != yC else 0.0
    ir = (xB - xC) / (yB - yC) if yB != yC else 0.0
    
    # Ensure il <= ir
    if ir < il:
        il, ir = ir, il
        xl, xr = xr, xl
        
    # First half: yC to yB
    while y < yB:
        # Flip bits in P-Buffer from xl to xr
        for x in range(int(xl), int(xr) + 1):
            p_buffer[y][x] ^= 1
        y += 1
        xl += il
        xr += ir
        
    # Second half: yB to yA
    # Update slope for the edge connected to B and A
    new_slope = (xA - xB) / (yA - yB) if yA != yB else 0.0
    
    # Determine which variable corresponds to the edge B-A
    # If swapped, il holds the original right slope (C-A), ir holds left (C-B)
    # The edge B-A replaces the right side in the second half.
    if ir < il: # Swap occurred
        il = new_slope
    else:
        ir = new_slope
        
    while y <= yA:
        for x in range(int(xl), int(xr) + 1):
            p_buffer[y][x] ^= 1
        y += 1
        xl += il
        xr += ir
```

### Edge Cases and Pitfalls
- **Flat-top triangles:** If $y_1 == y_2$, the first half loop does not execute. The second half must start from $y_1$ and use the correct slopes. The source notes "skip to second half" [S2, unofficial].
- **Horizontal edges:** If $dy = 0$, the slope is undefined. The source implies handling via error accumulation or skipping. In the pseudocode, slopes are set to 0 if $dy=0$.
- **Precision:** Floating-point slopes can accumulate error. The source notes integer arithmetic with error accumulation as an alternative for robustness [S2, unofficial].
- **Degenerate triangles:** If vertices are collinear, the triangle has no area. The algorithm should handle this by checking for zero area or skipping.

## Operating characteristics
- **Time Complexity:** $O(N \cdot H)$, where $N$ is the number of edges in the polygon and $H$ is the height of the bounding box. Each triangle is rasterized in $O(H)$ time.
- **Memory Complexity:** $O(W \cdot H)$ for the P-Buffer, where $W$ and $H$ are the width and height of the bounding box.
- **Strengths:** Simple implementation, suitable for voxelization where parity is sufficient.
- **Limitations:** Not optimized for modern GPUs; relies on CPU-side loop. Precision issues with floating-point slopes.

## Background
This method originates from work on voxelization of solids [S1, unofficial]. It is designed for 3D rasterization where the 2D algorithm serves as a core component for projecting 3D solids into a voxel grid. The use of a P-Buffer with bit-flipping is a key innovation for determining solid occupancy.

## Sources
[S1] Voxelization of solids using simplicial coverings, http://wscg.zcu.cz/wscg2004/Papers_2004_Short/J53.pdf, fetched 2026-10-26 (unofficial)
[S2] CDevJoud/Damascene, https://github.com/CDevJoud/Damascene, fetched 2026-10-27 (unofficial)
