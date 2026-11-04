# Linear Edge Intersection for Dual Contouring

Also called: Dual Contouring (Simple Mean Intersection), Simple Mean Intersection

## Overview
Linear Edge Intersection is a vertex placement method for Dual Contouring that computes a single point per grid cell intersecting the isosurface by averaging the intersection points on the cell edges [S1, unofficial]. It optionally refines this position using gradient information when available [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   `origin`: The 3D coordinates of the cell's corner with the lowest indices [S1, unofficial].
*   `dirs`: An array of 3 vectors representing the cell's edge directions (typically $\Delta x \hat{x}, \Delta y \hat{y}, \Delta z \hat{z}$) [S1, unofficial].
*   `vertices`: An array of 8 scalar values corresponding to the function values at the cell's 8 vertices, ordered in Morton (Z-order) [S1, unofficial].
*   `level`: The isovalue threshold (default 0.0) [S2, unofficial].
*   `gradients`: (Optional) An array of 3D vectors representing the gradient at each intersection point, used for refinement [S1, unofficial].

**Outputs:**
*   `surface_point`: A 3D vector representing the estimated isosurface location within the cell [S1, unofficial].
*   `valid`: A boolean indicating whether the cell intersects the isosurface [S1, unofficial].
*   `points`: (Optional) An $(N, 3)$ float32 tensor of intersection points on the edges [S2, unofficial].
*   `normals`: (Optional) An $(N, 3)$ float32 tensor of normals at those points [S2, unofficial].

### Data Structures
*   **Edge Lookup Table**: A static array of 24 integers defining the 12 edges of the cube. Each edge is defined by two vertex indices [S1, unofficial].
    ```python
    edges = [0, 4, 1, 5, 2, 6, 3, 7, 0, 2, 1, 3, 4, 6, 5, 7, 0, 1, 2, 3, 4, 5, 6, 7]
    ```

### Algorithm Steps
1.  **Check Intersection**: Adjust vertex values by subtracting `level`. If all adjusted values are $\ge 0$ or all are $< 0$, the cell does not intersect the isosurface; return `false` [S1, unofficial].
2.  **Initialize Accumulators**: Set `mean = (0, 0, 0)` and `count = 0` [S1, unofficial].
3.  **Iterate Edges**: Loop over $i$ from 0 to 11.
    a.  Get vertex indices: $v_1 = \text{edges}[2i]$, $v_2 = \text{edges}[2i+1]$ [S1, unofficial].
    b.  Get values: $a = \text{vertices}[v_1]$, $b = \text{vertices}[v_2]$ [S1, unofficial].
    c.  **Check Crossing**: If $a \cdot b > 0$, the edge does not cross the isosurface; skip to the next edge [S1, unofficial].
    d.  **Interpolate**: Calculate the interpolation parameter $x$:
        $$ \text{denom} = |b - a| $$
        $$ x = \begin{cases} 0.5 & \text{if } \text{denom} < \epsilon \\ \frac{|a|}{\text{denom}} & \text{otherwise} \end{cases} $$
        Where $\epsilon = 10^{-15}$ [S1, unofficial].
    e.  **Calculate Point**: Compute the intersection point $p$:
        $$ p = \text{origin} + \text{dirs}[0] \cdot \lfloor v_1 / 4 \rfloor + \text{dirs}[1] \cdot \lfloor (v_1 \bmod 4) / 2 \rfloor + \text{dirs}[2] \cdot (v_1 \bmod 2) + \text{dirs}[i // 4] \cdot x $$
        Add $p$ to `mean` and increment `count` [S1, unofficial].
4.  **Finalize Point**: If `count > 0`:
    $$ \text{surface\_point} = \text{origin} + \frac{\text{mean}}{\text{count}} $$
    Return `true` and `surface_point` [S1, unofficial].
5.  **Gradient-Based Refinement (Optional)**: If gradients are provided and `count >= 3`:
    a.  Construct matrix $A$ where row $i$ is the gradient at the $i$-th intersection point [S1, unofficial].
    b.  Construct vector $b$ where $b_i = \text{dot}(\text{gradient}_i, \text{point}_i)$ [S1, unofficial].
    c.  Solve $A q = b$ using least squares [S1, unofficial].
    d.  **Boundary Check**: If $q$ is outside the cell limits ($\text{origin}$ to $\text{origin} + \sum \text{dirs}$), fallback to the mean calculated in Step 4 [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vector3:
    def __init__(self, x=0.0, y=0.0, z=0.0):
        self.x, self.y, self.z = x, y, z
    def __add__(self, other): return Vector3(self.x + other.x, self.y + other.y, self.z + other.z)
    def __sub__(self, other): return Vector3(self.x - other.x, self.y - other.y, self.z - other.z)
    def __mul__(self, scalar): return Vector3(self.x * scalar, self.y * scalar, self.z * scalar)
    def __truediv__(self, scalar): return Vector3(self.x / scalar, self.y / scalar, self.z / scalar)
    def __iadd__(self, other):
        self.x += other.x; self.y += other.y; self.z += other.z
        return self
    def dot(self, other): return self.x * other.x + self.y * other.y + self.z * other.z

def compute_cell(origin, dirs, vertices, level=0.0, gradients=None):
    SMALL_NUMBER = 1e-15
    
    adjusted_vertices = [v - level for v in vertices]
    
    has_negative = any(v < 0 for v in adjusted_vertices)
    has_positive = any(v > 0 for v in adjusted_vertices)
    
    if not has_negative or not has_positive:
        return False, None

    edges = [0, 4, 1, 5, 2, 6, 3, 7, 0, 2, 1, 3, 4, 6, 5, 7, 0, 1, 2, 3, 4, 5, 6, 7]
    
    mean = Vector3(0, 0, 0)
    count = 0
    intersection_points = []
    
    for i in range(12):
        v1_idx = edges[2 * i]
        v2_idx = edges[2 * i + 1]
        
        a = adjusted_vertices[v1_idx]
        b = adjusted_vertices[v2_idx]
        
        if a * b > 0:
            continue
            
        denom = abs(b - a)
        
        if denom < SMALL_NUMBER:
            x = 0.5
        else:
            x = abs(a) / denom
            
        base_point = (
            dirs[0] * (v1_idx // 4) + 
            dirs[1] * ((v1_idx % 4) // 2) + 
            dirs[2] * (v1_idx % 2)
        )
        
        intersection_point = origin + base_point + dirs[i // 4] * x
        mean += intersection_point
        intersection_points.append(intersection_point)
        count += 1
        
    if count == 0:
        return False, None
        
    surface_point = origin + mean / count
    
    if gradients is not None and count >= 3:
        A = [gradients[k] for k in range(count)]
        b = [A[k].dot(intersection_points[k]) for k in range(count)]
        q = solve_least_squares(A, b)
        
        cell_max = origin + dirs[0] + dirs[1] + dirs[2]
        if not (all(origin.d <= q.d <= cell_max.d for d in ['x', 'y', 'z'] for q in [q])):
            surface_point = origin + mean / count
            
    return True, surface_point

def solve_least_squares(A, b):
    # Standard least squares solution: q = (A^T A)^-1 A^T b
    # Implemented here as a placeholder for standard linear algebra routines [G]
    import numpy as np
    return np.linalg.lstsq(np.array(A), np.array(b), rcond=None)[0]
```

### Parameters
*   `SMALL_NUMBER`: $10^{-15}$, used to prevent division by zero during interpolation [S1, unofficial].
*   `level`: Default 0.0 [S2, unofficial].

### Edge Cases and Pitfalls
*   **Degenerate Edges**: If $a \approx b$ (`denom < ε`), the interpolation parameter is set to 0.5 [S1, unofficial].
*   **No Intersection**: If all vertex values are on one side of the isosurface, the function returns `false`, and no vertex is generated for that cell [S1, unofficial].
*   **Gradient Fallback**: If gradient-based positioning is used but the resulting point lies outside the cell bounds, the method falls back to the mean intersection point [S1, unofficial].

## Subtopics

### [Dual Contouring QEF Solver](1424-dual-contouring-qef-solver.md)
Uses surface normals to minimize a Quadratic Error Function for more accurate vertex placement. Choose when surface normals are available and higher geometric fidelity is required.

### [Topologically Accurate Dual Isosurfacing Using Ray Intersection](800-topologically-accurate-dual-isosurfacing-using-ray-intersect.md)
Uses ray casting to ensure topological correctness in complex scenes. Choose when the linear method fails to preserve topology (e.g., thin structures).

### [Simplified Dual Contouring](1580-simplified-dual-contouring.md)
A variant that may reduce computational overhead or simplify quad generation. Choose for faster rendering with acceptable quality loss.

## Operating characteristics
Not given in the sources.

## Background
The method originates from the basic implementation of Dual Contouring, where the vertex position is approximated by the mean of edge intersections [S1, unofficial]. It is often referred to as "Simple Mean Intersection" in contrast to more advanced variants that use QEF solvers [S1, unofficial]. The implementation details are found in open-source libraries such as `salvipeter/dual-contouring` [S1, unofficial] and `GuangyanCai/isoext` [S2, unofficial].

## Sources
[S1] salvipeter/dual-contouring, https://github.com/salvipeter/dual-contouring, fetched 2026-10-26 (unofficial)
[S2] GuangyanCai/isoext, https://github.com/GuangyanCai/isoext, fetched 2026-10-27 (unofficial)
