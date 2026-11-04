# Dual Vertex Positioning via Least Squares

Also called: Dual Vertex Localization via Quadratic Error Minimization, Ju et al. 2002 Method, Least Squares Dual Vertex, Minimizing Distance to Intersections, Quadratic Error Function (QEF) Vertex Placement

## Overview
This method computes the position of a dual vertex within a voxel cell by minimizing the squared distance to all edge intersection points on the isosurface boundary of that cell [S1, unofficial]. It sets itself apart from simpler centering methods by using a weighted least-squares fit of the local surface geometry (normals and intersections) rather than just averaging coordinates, providing a more accurate representation of the underlying isosurface [S1, unofficial]. It builds on [Mass Point Centering for Dual Contouring](3939-mass-point-centering-for-dual-contouring.md) by centering the linear system solution around the centroid of the sample points to improve numerical stability [S1, unofficial].

## How it works
**Inputs:**
- A voxel cell intercepted by the isosurface $\partial V$.
- The scalar field values at the voxel corners.
- The scalar field volume data for gradient computation.

**Outputs:**
- The 3D position $x$ of the dual vertex associated with the cell.

**Data Structures:**
- $p_i$: 3D coordinate of the $i$-th edge intersection point.
- $n_i$: 3D normal vector at the $i$-th intersection point.
- $A$: An $n \times 3$ matrix where each row $i$ is the normal vector $n_i^T$.
- $b$: An $n$-dimensional vector where each element $b_i = n_i \cdot p_i$.
- $p$: The center of mass (mean) of all intersection points $p_i$.

**Steps:**

1.  **Identify Intersections and Normals:**
    For each edge of the cell intercepted by the isosurface $\partial V$:
    a.  Identify the two transition voxels (vertices of the voxel grid) belonging to that edge.
    b.  Calculate the intersection point $p_i$ using linear interpolation between the two transition voxels based on their coordinates and scalar values [S1, unofficial].
    c.  Calculate the normal $n_i$ at $p_i$ by applying the Prewitt operator to the volume data. The Prewitt operator is a difference operator that approximates the gradient by convolving $3 \times 3 \times 3$ masks with the volume [S1, unofficial].

2.  **Compute Center of Mass:**
    Calculate $p$, the center of mass of the intersection points:
    $$ p = \frac{1}{n} \sum_{i=1}^{n} p_i $$
    where $n$ is the number of intercepted edges [S1, unofficial].

3.  **Construct Linear System:**
    Construct the matrix $A$ and vector $b$ as defined above. The goal is to solve the overdetermined system $Ax = b$ in the least-squares sense, which minimizes the error function:
    $$ E[x] = \sum_i (n_i \cdot (x - p_i))^2 $$
    [S1, unofficial]

4.  **Solve for Offset:**
    Compute the offset $c$ from the center of mass using the Moore-Penrose pseudo-inverse of $A^T A$:
    $$ c = (A^T A)^+ (A^T b - A^T A p) $$
    where $(A^T A)^+$ is computed via Singular Value Decomposition (SVD) [S1, unofficial].

5.  **Compute Final Position:**
    Calculate the dual vertex position:
    $$ x = c + p $$
    [S1, unofficial]

6.  **Boundary Check:**
    If the resulting position $x$ falls outside the spatial bounds of the cell, use the center of mass $p$ directly as the dual vertex position [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_dual_vertex(cell_edges, scalar_field, voxel_size):
    """
    Computes the dual vertex position for a cell using least squares minimization.
    
    Args:
        cell_edges: List of tuples (edge_index, v1_coord, v1_scalar, v2_coord, v2_scalar)
                    representing edges intercepted by the isosurface.
        scalar_field: 3D array of scalar values.
        voxel_size: Size of a single voxel (assumed uniform).
        
    Returns:
        x: 3D position of the dual vertex.
    """
    intersections = []
    normals = []
    
    # Step 1: Identify Intersections and Normals
    for edge in cell_edges:
        # Linear interpolation for intersection point p_i
        # Assuming v1 and v2 are corner coordinates and scalars
        v1_coord, v1_scalar, v2_coord, v2_scalar = edge[1], edge[2], edge[3], edge[4]
        
        # Linear interpolation factor t where scalar crosses 0 (or isosurface value)
        # Assuming isosurface value is 0 for simplicity, or normalized
        total_diff = v2_scalar - v1_scalar
        if abs(total_diff) < 1e-9:
            t = 0.5 # Fallback if scalars are equal
        else:
            t = -v1_scalar / total_diff
            
        # Clamp t to [0, 1] to stay within the edge
        t = max(0.0, min(1.0, t))
        
        p_i = v1_coord + t * (v2_coord - v1_coord)
        intersections.append(p_i)
        
        # Compute normal n_i using Prewitt operator
        # Prewitt gradient approximation at the center of the voxel containing the edge midpoint
        # For implementation, we approximate the gradient at p_i using the scalar field
        n_i = compute_prewitt_normal(scalar_field, p_i, voxel_size)
        normals.append(n_i)
        
    n = len(intersections)
    if n == 0:
        return None # No intersection
        
    # Step 2: Compute Center of Mass
    p = sum(intersections) / n
    
    # Step 3: Construct Linear System Ax = b
    # A is n x 3, b is n x 1
    A = []
    b = []
    for i in range(n):
        A.append(normals[i])
        b.append(dot(normals[i], intersections[i]))
        
    # Step 4: Solve for Offset c
    # Compute A^T A
    AtA = matrix_multiply(transpose(A), A)
    
    # Compute A^T b
    Atb = matrix_multiply(transpose(A), b)
    
    # Compute Moore-Penrose pseudo-inverse of AtA using SVD
    # AtA = U * S * V^T
    # (AtA)^+ = V * S^+ * U^T
    U, S, Vt = svd(AtA)
    
    # Invert singular values (handle near-zero)
    S_inv = []
    for s in S:
        if s > 1e-9:
            S_inv.append(1.0 / s)
        else:
            S_inv.append(0.0)
            
    # Construct pseudo-inverse matrix
    # S_diag is a diagonal matrix with S_inv
    # (AtA)^+ = V^T^T * S_diag * U^T = V * S_diag * U^T
    # Note: SVD returns Vt, so V = Vt^T
    pseudo_inv = matrix_multiply(matrix_multiply(transpose(Vt), diag(S_inv)), U)
    
    # c = (A^T A)^+ (A^T b - A^T A p)
    # First compute A^T A p
    AtA_p = matrix_multiply(AtA, p)
    
    # Then compute the term in parenthesis
    rhs = subtract(Atb, AtA_p)
    
    c = matrix_multiply(pseudo_inv, rhs)
    
    # Step 5: Compute Final Position
    x = add(c, p)
    
    # Step 6: Boundary Check
    if not is_inside_cell(x, cell_edges):
        return p
        
    return x

def compute_prewitt_normal(scalar_field, point, voxel_size):
    """
    Approximates the gradient using the Prewitt operator.
    This is a simplified implementation assuming point is within a voxel.
    """
    # Find the voxel index containing the point
    # In a real implementation, this would involve grid lookup
    # For this pseudocode, we assume a helper to get voxel center and neighbors
    
    # Prewitt masks for 3D are 3x3x3. 
    # We approximate the gradient by finite differences.
    # dx = (f(x+1, y, z) - f(x-1, y, z)) / 2
    # dy = (f(x, y+1, z) - f(x, y-1, z)) / 2
    # dz = (f(x, y, z+1) - f(x, y, z-1)) / 2
    
    # This requires accessing scalar_field at neighboring voxel centers.
    # For brevity, we assume a function get_scalar_at_grid_point exists.
    
    # Note: The source specifies Prewitt operator with 3x3x3 masks.
    # A standard 3D Prewitt gradient is:
    # Gx = sum(f(x+1,y,z) - f(x-1,y,z)) over y,z neighbors
    # Gy = sum(f(x,y+1,z) - f(x,y-1,z)) over x,z neighbors
    # Gz = sum(f(x,y,z+1) - f(x,y,z-1)) over x,y neighbors
    
    # Simplified gradient approximation for pseudocode completeness
    # Using central differences as a proxy for Prewitt if full mask convolution is too verbose
    # In practice, one would implement the full 3x3x3 convolution.
    
    # Placeholder for actual Prewitt convolution logic
    # [G] Implement full 3x3x3 Prewitt mask convolution on scalar_field around point
    # Return normalized gradient vector
    
    # For the sake of valid Python, we return a zero vector if not implemented
    # A real implementation must fill this.
    return [0.0, 0.0, 0.0]

def is_inside_cell(x, cell_edges):
    """
    Checks if point x is within the bounding box of the cell defined by cell_edges.
    """
    # Get min and max coordinates of the cell
    min_coord = [float('inf')] * 3
    max_coord = [float('-inf')] * 3
    
    for edge in cell_edges:
        v1_coord = edge[1]
        v2_coord = edge[3]
        for k in range(3):
            min_coord[k] = min(min_coord[k], v1_coord[k], v2_coord[k])
            max_coord[k] = max(max_coord[k], v1_coord[k], v2_coord[k])
            
    for k in range(3):
        if x[k] < min_coord[k] or x[k] > max_coord[k]:
            return False
    return True
```

**Parameters:**
- Isosurface value: Typically 0 for binary volumes, or a threshold for scalar volumes. Not given in the sources.
- Prewitt mask weights: Standard Prewitt masks are used. Not given in the sources.

**Edge Cases and Pitfalls:**
- The minimization result $x$ may fall outside the spatial bounds of the cell. In this case, use the center of mass $p$ directly as the dual vertex position [S1, unofficial].
- If the matrix $A^T A$ is singular (e.g., all normals are parallel), the SVD pseudo-inverse handles it by zeroing out the corresponding singular values, but the solution may be unstable. Not given in the sources.

## Operating characteristics
- **Time Complexity:** Dominated by the SVD of a $3 \times 3$ matrix ($A^T A$), which is constant time $O(1)$ per cell. The gradient computation is $O(1)$ per intersection point.
- **Memory Complexity:** $O(1)$ per cell, storing the intersection points and normals.
- **Strengths:** Provides a geometrically accurate vertex position that respects the local surface orientation. More robust than simple averaging for curved surfaces.
- **Limitations:** Requires gradient computation, which can be noisy. The resulting vertex may lie outside the cell, requiring clamping.
- **When to use:** When high-quality isosurface extraction is required and computational cost is acceptable.
- **When not to use:** When real-time performance is critical and simpler methods suffice.

## Background
This method originates from the work of Ju et al. (2002) and is based on the Quadratic Error Function (QEF) minimization introduced by Garland and Heckbert (1998) [S1, unofficial]. It is a standard approach in dual contouring for locating dual vertices. The specific implementation described here uses the Prewitt operator for gradient estimation and centers the solution around the cell's center of mass for numerical stability [S1, unofficial].

## Sources
[S1] EXTRAÇÃO DE ISOSUPERFÍCIES DE DOMOS DE SAL EM VOLUMES BINÁRIOS MASSIVOS, https://doi.org/10.17771/pucrio.acad.51290, fetched 2026-10-26 (unofficial)
