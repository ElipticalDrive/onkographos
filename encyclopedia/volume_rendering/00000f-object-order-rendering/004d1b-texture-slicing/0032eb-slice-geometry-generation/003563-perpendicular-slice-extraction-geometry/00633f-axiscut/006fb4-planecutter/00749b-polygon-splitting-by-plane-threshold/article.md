# Polygon Splitting by Plane Threshold

Also called: Polygon Clipping by Plane, SPLITPOLYGON

## Overview
Polygon Splitting by Plane Threshold is a geometric primitive that partitions a 2D polygon (typically a triangle) into two sub-polygons based on a 3D plane threshold. It solves the problem of determining which parts of a polygon lie on either side of a cutting plane, returning the vertices of the resulting "below" and "above" polygons. It is distinguished from general polygon clipping algorithms by operating directly in barycentric coordinates relative to a reference simplex, avoiding explicit conversion to Cartesian space for the input vertices, and using a Sutherland-Hodgman style single-pass iteration [S1].

## How it works
**Inputs:**
*   $\mathbf{P}$: A list of vertices representing the input polygon in barycentric coordinates relative to a reference triangle (simplex).
*   $\mathbf{V}_\sigma \in \mathbb{R}^{3 \times 3}$: The vertices of the reference simplex, where each row $\mathbf{V}_{\sigma, i}$ is a 3D Cartesian coordinate vector.
*   $t$: The scalar threshold value for the cutting plane.
*   $\ell$: The index of the axis (0, 1, or 2) along which the plane is defined (i.e., the plane is $x_\ell = t$).

**Outputs:**
*   $\mathbf{P}_{below}$: A list of barycentric vertices for the polygonal region where the Cartesian coordinate $x_\ell \leq t$.
*   $\mathbf{P}_{above}$: A list of barycentric vertices for the polygonal region where the Cartesian coordinate $x_\ell > t$.

**Data Structures:**
*   Vertices are stored as lists of tuples or arrays.
*   Barycentric coordinates $\mathbf{b} = (w_0, w_1, w_2)$ satisfy $\sum w_i = 1$.
*   Cartesian conversion is implicit via dot product with the simplex's axis-specific coordinates.

**Steps:**
1.  Initialize two empty lists: $\mathbf{P}_{below} \leftarrow []$ and $\mathbf{P}_{above} \leftarrow []$.
2.  Extract the $\ell$-th Cartesian coordinate for each vertex of the reference simplex $\mathbf{V}_\sigma$. Let $\mathbf{c}^\ell$ be the vector containing these values: $\mathbf{c}^\ell_i = \mathbf{V}_{\sigma, i, \ell}$ for $i \in \{0, 1, 2\}$.
3.  Iterate through each edge of the polygon $\mathbf{P}$, defined by consecutive vertices $\mathbf{b}_{curr}$ and $\mathbf{b}_{next}$. The loop runs for $i$ from $0$ to $|\mathbf{P}| - 1$, where $\mathbf{b}_{curr} = \mathbf{P}[i]$ and $\mathbf{b}_{next} = \mathbf{P}[(i+1) \bmod |\mathbf{P}|]$.
4.  Compute the Cartesian coordinate $x$ along axis $\ell$ for the current and next vertices using barycentric interpolation:
    $$x_{curr} = \mathbf{b}_{curr} \cdot \mathbf{c}^\ell = \sum_{j=0}^{2} \mathbf{b}_{curr, j} \cdot \mathbf{V}_{\sigma, j, \ell}$$
    $$x_{next} = \mathbf{b}_{next} \cdot \mathbf{c}^\ell = \sum_{j=0}^{2} \mathbf{b}_{next, j} \cdot \mathbf{V}_{\sigma, j, \ell}$$
5.  Classify the current vertex $\mathbf{b}_{curr}$:
    *   If $x_{curr} \leq t$, append $\mathbf{b}_{curr}$ to $\mathbf{P}_{below}$.
    *   Otherwise, append $\mathbf{b}_{curr}$ to $\mathbf{P}_{above}$.
6.  Check if the edge $(\mathbf{b}_{curr}, \mathbf{b}_{next})$ crosses the plane threshold. This occurs if one endpoint is below or on the plane and the other is strictly above:
    $$\text{crosses} = (x_{curr} \leq t) \neq (x_{next} \leq t)$$
7.  If the edge crosses the plane:
    *   Compute the intersection parameter $\alpha$:
        $$\alpha = \frac{t - x_{curr}}{x_{next} - x_{curr}}$$
    *   Compute the barycentric coordinates of the intersection point $\mathbf{b}_{int}$ by linearly interpolating between $\mathbf{b}_{curr}$ and $\mathbf{b}_{next}$:
        $$\mathbf{b}_{int} = (1 - \alpha) \cdot \mathbf{b}_{curr} + \alpha \cdot \mathbf{b}_{next}$$
    *   Append $\mathbf{b}_{int}$ to both $\mathbf{P}_{below}$ and $\mathbf{P}_{above}$.
8.  After processing all edges, return the pair $(\mathbf{P}_{below}, \mathbf{P}_{above})$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def split_polygon_by_plane_threshold(P, V_sigma, t, l):
    """
    Split a polygon P in barycentric coords by a plane x_l = t.
    
    Args:
        P: List of barycentric vertices (tuples/lists of length 3).
        V_sigma: 3x3 array of simplex vertices (rows are vertices, cols are x,y,z).
        t: Threshold value.
        l: Axis index (0, 1, or 2).
        
    Returns:
        (P_below, P_above): Lists of barycentric vertices.
    """
    P_below = []
    P_above = []
    
    # Extract the l-th coordinate of each simplex vertex
    c_l = [V_sigma[i][l] for i in range(3)]
    
    n = len(P)
    for i in range(n):
        b_curr = P[i]
        b_next = P[(i + 1) % n]
        
        # Compute Cartesian coordinates along axis l
        x_curr = sum(b_curr[j] * c_l[j] for j in range(3))
        x_next = sum(b_next[j] * c_l[j] for j in range(3))
        
        # Classify current vertex
        if x_curr <= t:
            P_below.append(b_curr)
        else:
            P_above.append(b_curr)
            
        # Check for edge crossing
        curr_below = (x_curr <= t)
        next_below = (x_next <= t)
        
        if curr_below != next_below:
            # Compute intersection parameter
            denom = x_next - x_curr
            # Guard against division by zero if x_next == x_curr, 
            # though this implies no crossing if signs differ, 
            # so denom != 0 if crossing occurs.
            alpha = (t - x_curr) / denom
            
            # Compute intersection point in barycentric coords
            b_int = [
                (1 - alpha) * b_curr[j] + alpha * b_next[j]
                for j in range(3)
            ]
            
            P_below.append(b_int)
            P_above.append(b_int)
            
    return (P_below, P_above)
```

**Parameters and Edge Cases:**
*   **Threshold $t$**: Determines the plane position. Vertices with $x_\ell = t$ are included in $\mathbf{P}_{below}$ [S1].
*   **Degenerate Edges**: If $x_{curr} = x_{next}$, the edge is parallel to the plane. The condition $(x_{curr} \leq t) \neq (x_{next} \leq t)$ will be false, so no intersection is computed. This is correct behavior.
*   **Precision**: Floating-point comparisons for $x_{curr} \leq t$ should be consistent. The sources use strict inequality for the "above" set implicitly by the complement of $\leq$ [S1].

## Operating characteristics
*   **Time Complexity**: $O(N)$, where $N$ is the number of vertices in the input polygon $\mathbf{P}$. Each vertex and edge is processed exactly once.
*   **Memory Complexity**: $O(N)$ for the output lists, as the total number of vertices in $\mathbf{P}_{below}$ and $\mathbf{P}_{above}$ is at most $N + 2$ (each crossing adds one vertex to each list).
*   **Strengths**: Efficient single-pass algorithm. Avoids explicit 3D-to-2D projection or coordinate transformation overhead by working in barycentric space relative to the simplex. Suitable for differentiable rendering pipelines as it is composed of differentiable operations (linear interpolation, comparisons) [S1].
*   **Limitations**: Assumes the input polygon is defined relative to a specific simplex $\mathbf{V}_\sigma$. The output polygons are in barycentric coordinates, which may require conversion back to Cartesian space for further geometric processing.

## Background
The method is described in the context of differentiable voxelization of surface representations. It serves as a core geometric primitive for generating slice geometry in volume rendering pipelines, specifically within the PlaneCutter filter which cuts hypertree grids using planes [S1]. The algorithm is presented as "Algorithm 6: SPLITPOLYGON" in the source literature.

## Sources
[S1] Differentiable Voxelization of Surface Representations, https://doi.org/10.1145/3799902.3811203, fetched 2026-10-27
