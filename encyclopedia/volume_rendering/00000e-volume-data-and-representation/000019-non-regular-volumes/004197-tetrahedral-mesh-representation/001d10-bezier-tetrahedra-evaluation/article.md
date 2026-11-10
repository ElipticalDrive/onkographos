# Bezier Tetrahedra Evaluation

Also called: Bezier Tetrahedron, Cubic BT

## Overview
Bezier Tetrahedra (BT) evaluation computes scalar field values within a tetrahedral element using cubic Bernstein polynomials defined by 20 control weights and a bounding tetrahedron. It solves the problem of representing volumetric data on unstructured grids with higher-order continuity than linear interpolation, enabling efficient ray-tracing of isosurfaces by reducing intersection to univariate polynomial root finding. It differs from linear tetrahedral methods by providing $C^0$ continuity with smooth gradients and from quadratic variants by using cubic basis functions, balancing memory cost (20 weights) against approximation power.

## How it works
**Inputs:**
- A query point $P = (x, y, z, 1)$ in projective space.
- A bounding tetrahedron defined by four vertices $T = (T_1, T_2, T_3, T_4)$, where each $T_i$ is a 3D point.
- A set of 20 scalar weights $\{w_{ijkl}\}$ for $i+j+k+l=3$.

**Outputs:**
- A scalar value $b(P)$ representing the function value at $P$.

**Data Structures:**
- **Weights:** Stored as a flat array or list of 20 floats. The indices correspond to the multinomial combinations of degree 3. The tensor form (64 values) is never stored [S1].
- **Tetrahedron Vertices:** Stored as four 3D vectors.

**Steps:**
1. **Compute Barycentric Coordinates:**
   Construct the $4 \times 4$ matrix $\mathbf{T}$ where the rows are the homogeneous coordinates of the vertices $T_1, T_2, T_3, T_4$.
   Compute the inverse matrix $\mathbf{T}^{-1}$.
   Compute the barycentric coordinate vector $\mathbf{r} = (r, s, t, u)$ by multiplying the query point $P$ by $\mathbf{T}^{-1}$:
   $$ \mathbf{r} = P \cdot \mathbf{T}^{-1} $$
   [S1]

2. **Evaluate Bernstein Polynomial:**
   Compute the value $b(P)$ using the sum over all non-negative integers $i, j, k, l$ such that $i+j+k+l=3$:
   $$ b(P) = \sum_{i+j+k+l=3} w_{ijkl} \binom{3}{i,j,k,l} r^i s^j t^k u^l $$
   where $\binom{3}{i,j,k,l} = \frac{3!}{i!j!k!l!}$ is the multinomial coefficient.
   [S1]

**Parameters:**
- **Degree:** 3 (Cubic). This determines the number of weights (20) and the power of the barycentric coordinates.
- **Weights:** 20 scalar values. Typical values depend on the volumetric data being represented.

**Edge Cases and Pitfalls:**
- **Degenerate Tetrahedra:** If the determinant of $\mathbf{T}$ is zero (volume is zero), $\mathbf{T}^{-1}$ does not exist. The implementation must check for this condition. [G]
- **Precision:** Barycentric coordinate computation involves matrix inversion. For nearly flat tetrahedra, numerical instability may occur. [G]
- **Tensor Form:** The tensor form of BT weights (64 values) is less compact than the tetrahedral form (20 values). Implementations should never store the tensor form; they must expand from the tetrahedral form for transformations and encode back to tetrahedral form. [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_bezier_tetrahedron(P, T, weights):
    """
    Evaluate a cubic Bezier Tetrahedron at point P.
    
    Args:
        P: tuple (x, y, z, 1) - Point in projective space.
        T: list of 4 tuples - Vertices (x, y, z) of the bounding tetrahedron.
        weights: list of 20 floats - Control weights w_ijkl for i+j+k+l=3.
        
    Returns:
        float: The scalar value at P.
    """
    # Step 1: Compute Barycentric Coordinates
    # Construct 4x4 matrix T_mat from vertices
    # Rows are T1, T2, T3, T4 in homogeneous coordinates (x, y, z, 1)
    T_mat = [
        [T[0][0], T[0][1], T[0][2], 1.0],
        [T[1][0], T[1][1], T[1][2], 1.0],
        [T[2][0], T[2][1], T[2][2], 1.0],
        [T[3][0], T[3][1], T[3][2], 1.0]
    ]
    
    # Compute inverse of T_mat
    T_inv = invert_matrix_4x4(T_mat)
    
    # Multiply P by T_inv to get barycentric coords (r, s, t, u)
    # P is (x, y, z, 1)
    r = P[0]*T_inv[0][0] + P[1]*T_inv[1][0] + P[2]*T_inv[2][0] + P[3]*T_inv[3][0]
    s = P[0]*T_inv[0][1] + P[1]*T_inv[1][1] + P[2]*T_inv[2][1] + P[3]*T_inv[3][1]
    t = P[0]*T_inv[0][2] + P[1]*T_inv[1][2] + P[2]*T_inv[2][2] + P[3]*T_inv[3][2]
    u = P[0]*T_inv[0][3] + P[1]*T_inv[1][3] + P[2]*T_inv[2][3] + P[3]*T_inv[3][3]
    
    # Step 2: Evaluate Polynomial
    # Precompute powers of barycentric coordinates
    r2 = r*r
    r3 = r2*r
    s2 = s*s
    s3 = s2*s
    t2 = t*t
    t3 = t2*t
    u2 = u*u
    u3 = u2*u
    
    value = 0.0
    
    # Iterate over all combinations of i,j,k,l such that i+j+k+l=3
    # There are 20 such combinations.
    # Weights are assumed to be stored in a specific order corresponding to these combinations.
    # The order must match the summation. Common order is lexicographic or based on index mapping.
    # Here we assume weights are indexed by a function map(i,j,k,l) -> index.
    # For implementation, one might hardcode the 20 terms or use a lookup table.
    
    # Example terms (i,j,k,l) and their multinomial coefficients:
    # (3,0,0,0): 1, (0,3,0,0): 1, (0,0,3,0): 1, (0,0,0,3): 1
    # (2,1,0,0): 3, (2,0,1,0): 3, (2,0,0,1): 3
    # (1,2,0,0): 3, (0,2,1,0): 3, (0,2,0,1): 3
    # (1,0,2,0): 3, (0,1,2,0): 3, (0,0,2,1): 3
    # (1,0,0,2): 3, (0,1,0,2): 3, (0,0,1,2): 3
    # (1,1,1,0): 6, (1,1,0,1): 6, (1,0,1,1): 6, (0,1,1,1): 6
    
    # This part depends on the specific storage order of 'weights'.
    # Assuming weights are stored in lexicographic order of (i,j,k,l):
    
    # i=3
    value += weights[0] * 1.0 * r3
    # j=3
    value += weights[1] * 1.0 * s3
    # k=3
    value += weights[2] * 1.0 * t3
    # l=3
    value += weights[3] * 1.0 * u3
    
    # i=2, j=1
    value += weights[4] * 3.0 * r2 * s
    # i=2, k=1
    value += weights[5] * 3.0 * r2 * t
    # i=2, l=1
    value += weights[6] * 3.0 * r2 * u
    
    # j=2, i=1
    value += weights[7] * 3.0 * s2 * r
    # j=2, k=1
    value += weights[8] * 3.0 * s2 * t
    # j=2, l=1
    value += weights[9] * 3.0 * s2 * u
    
    # k=2, i=1
    value += weights[10] * 3.0 * t2 * r
    # k=2, j=1
    value += weights[11] * 3.0 * t2 * s
    # k=2, l=1
    value += weights[12] * 3.0 * t2 * u
    
    # l=2, i=1
    value += weights[13] * 3.0 * u2 * r
    # l=2, j=1
    value += weights[14] * 3.0 * u2 * s
    # l=2, k=1
    value += weights[15] * 3.0 * u2 * t
    
    # i=1, j=1, k=1
    value += weights[16] * 6.0 * r * s * t
    # i=1, j=1, l=1
    value += weights[17] * 6.0 * r * s * u
    # i=1, k=1, l=1
    value += weights[18] * 6.0 * r * t * u
    # j=1, k=1, l=1
    value += weights[19] * 6.0 * s * t * u
    
    return value

def invert_matrix_4x4(M):
    """
    Computes the inverse of a 4x4 matrix.
    This is a standard linear algebra operation.
    """
    # Implementation of 4x4 matrix inversion omitted for brevity, 
    # as it is a standard math operation.
    pass
```

## Subtopics

### [Quadratic Super Splines](7082-quadratic-super-splines.md)
Quadratic Super Splines is a technique for reconstructing iso-surfaces from volumetric data with pseudo $C^1$ continuity along cell boundaries, producing smooth contours by partitioning cells into tetrahedra and using Bernstein-Bézier coefficients. It differs from standard linear or cubic tetrahedral methods by employing a univariate piecewise quadratic polynomial that allows for efficient ray-iso-surface intersection via quadratic equation solving, balancing smoothness against the high memory cost of storing 65 coefficients per cell.

## Operating characteristics
- **Time Complexity:** Evaluation is $O(1)$ per tetrahedron, involving a matrix inversion (constant time for $4 \times 4$) and a fixed number of arithmetic operations (20 weights). Ray tracing requires solving a univariate polynomial, which is also constant time but involves root-finding algorithms.
- **Memory Complexity:** 20 scalar weights per tetrahedron plus 4 vertices. This is more memory-intensive than linear tetrahedra (4 weights) but less than quadratic super splines (65 coefficients) [S1][S2, unofficial].
- **Measured Performance:** Loop and Blinn [2006] implemented this on graphics hardware, suggesting efficient rasterization or ray-tracing performance. Specific benchmarks are not given in the sources.
- **Strengths:** Smooth representation, efficient ray-tracing via univariate polynomial reduction, hardware-friendly.
- **Limitations:** Higher memory cost than linear methods, potential numerical instability with degenerate tetrahedra.
- **When to Use:** When smooth iso-surfaces are required and memory allows for higher-order coefficients.
- **When Not to Use:** When memory is extremely constrained or when linear interpolation is sufficient.

## Background
Bezier Tetrahedra were developed by Charles Loop and James Blinn in 2006 for interactive volume isosurface rendering on graphics hardware [S1]. The method extends the concept of Bezier curves and surfaces to 3D tetrahedral elements, allowing for efficient ray-tracing by transforming the problem into screen space.

## Sources
[S1] Interactive volume isosurface rendering using BT volumes, https://doi.org/10.1145/1342250.1342257, fetched 2026-10-30
