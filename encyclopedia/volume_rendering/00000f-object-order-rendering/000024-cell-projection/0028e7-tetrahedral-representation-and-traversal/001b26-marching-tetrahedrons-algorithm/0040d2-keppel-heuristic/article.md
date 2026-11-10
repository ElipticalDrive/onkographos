# Keppel Heuristic
Also called: Keppel Tetrahedron Method
## Overview
The Keppel Heuristic is a local decision method used in contour tiling to determine how to connect two parallel contour segments during volume rendering [S1, unofficial]. It resolves the ambiguity of which vertex to connect next by comparing the volumes of two candidate tetrahedra formed with interior reference points [S1, unofficial]. It sets itself apart from global optimization methods like [Planar Contour Tiling](10346-planar-contour-tiling.md) by using a greedy, local volume comparison rather than a graph-theoretic global optimum [S1, unofficial].
## How it works
**Inputs and Outputs**
- Inputs: Two parallel contour segments, defined by vertices $V_{i,j}$ and $V_{i+1,j}$ on one segment and $V_{i,j+1}$ and $V_{i+1,j+1}$ on the other. Interior points $O_p$ and $O_q$ located within the respective contour boundaries.
- Output: A decision to connect either to vertex $V_{i,j+1}$ (vertical step) or $V_{i+1,j}$ (horizontal step), and marking the corresponding arc.

**Data Structures**
- Contour vertices: Stored as 2D or 3D coordinates.
- Interior points: $O_p$ and $O_q$ are pre-computed points strictly inside the contour polygons.

**Steps**
1. Identify the current vertex pair to connect, typically starting from a known connection point (e.g., $V_{0,0}$) and proceeding along the contours.
2. Construct two tetrahedra using the current vertices and the interior points:
   - A "vertical" tetrahedron using points $P_j, Q_i, Q_{i+1}, O_p$.
   - A "horizontal" tetrahedron using points $P_j, Q_{j+1}, Q_i, O_q$.
   *Note: The source notation uses $P$ and $Q$ interchangeably with contour indices. Based on the formulas, $P_j$ corresponds to a vertex on one contour and $Q_i, Q_{i+1}$ to vertices on the adjacent contour.*
3. Calculate the volume of the vertical tetrahedron $T_{i,j}^v$.
4. Calculate the volume of the horizontal tetrahedron $T_{i,j}^h$.
5. Compare the volumes:
   - If $T_{i,j}^v > T_{i,j}^h$, select the next vertex $V_{i,j+1}$ (moving along the second contour).
   - Otherwise, mark the arc and select the next vertex $V_{i+1,j}$ (moving along the first contour).
6. Repeat for the next segment.

**Formulas**
The volume of a tetrahedron defined by vertices $A, B, C, D$ is given by:
$$ V = \frac{1}{6} | (B-A) \cdot ((C-A) \times (D-A)) | $$

The specific volumes for the heuristic are:
$$ T_{i,j}^v = \text{Volume}(P_j, Q_i, Q_{i+1}, O_p) $$
$$ T_{i,j}^h = \text{Volume}(P_j, Q_{j+1}, Q_i, O_q) $$

Where:
- $P_j$ is a vertex on the first contour segment.
- $Q_i, Q_{i+1}$ are adjacent vertices on the second contour segment.
- $O_p$ is an interior point associated with the vertical configuration.
- $O_q$ is an interior point associated with the horizontal configuration.

**Parameters**
- Interior points $O_p, O_q$: Not given in the sources. Must be pre-computed to lie strictly inside the respective contour regions.

**Edge Cases and Pitfalls**
- **Pre-processing Requirement**: The method requires the contours to be decomposed into a series of convex or concave parts. This necessitates significant pre-processing to identify these parts and compute valid interior points [S1, unofficial].
- **Degenerate Volumes**: If the tetrahedra are flat (volume $\approx 0$), the heuristic may fail or produce unstable results. This is likely if the contour segments are collinear or the interior points are poorly chosen.
- **Local Greedy Nature**: As a heuristic, it does not guarantee a globally optimal surface, potentially leading to artifacts in complex topologies compared to global methods.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def keppel_heuristic_step(P_j, Q_i, Q_i_plus_1, Q_j_plus_1, O_p, O_q):
    """
    Determines the next vertex to connect in contour tiling.
    
    Args:
        P_j: Vertex on contour 1.
        Q_i: Vertex on contour 2.
        Q_i_plus_1: Next vertex on contour 2.
        Q_j_plus_1: Next vertex on contour 1 (conceptually, though notation varies).
        O_p: Interior point for vertical tetrahedron.
        O_q: Interior point for horizontal tetrahedron.
        
    Returns:
        'vertical' if T_v > T_h, else 'horizontal'.
    """
    # Calculate vertical tetrahedron volume: T_{i,j}^v = f(P_j, Q_i, Q_{i+1}, O_p)
    # Vertices: P_j, Q_i, Q_{i+1}, O_p
    vol_v = tetrahedron_volume(P_j, Q_i, Q_i_plus_1, O_p)
    
    # Calculate horizontal tetrahedron volume: T_{i,j}^h = f(P_j, Q_{j+1}, Q_i, O_q)
    # Note: Source formula uses Q_{j+1}. Assuming Q_j_plus_1 corresponds to the next vertex 
    # on the contour where P_j resides, or the corresponding vertex in the grid.
    # Based on standard grid notation in contour tiling:
    # Vertical step moves to (i, j+1). Horizontal step moves to (i+1, j).
    # The formula T_{i,j}^h uses Q_{j+1}. This implies Q indices might refer to the other contour.
    # Let's stick strictly to the source formula variables.
    # T_{i,j}^h = Volume(P_j, Q_{j+1}, Q_i, O_q)
    vol_h = tetrahedron_volume(P_j, Q_j_plus_1, Q_i, O_q)
    
    if vol_v > vol_h:
        return 'vertical' # Next vertex V_{i,j+1}
    else:
        return 'horizontal' # Next vertex V_{i+1,j}

def tetrahedron_volume(A, B, C, D):
    """
    Computes the volume of a tetrahedron defined by points A, B, C, D.
    V = 1/6 * | (B-A) . ((C-A) x (D-A)) |
    """
    # Vector subtraction
    AB = vector_sub(B, A)
    AC = vector_sub(C, A)
    AD = vector_sub(D, A)
    
    # Cross product (C-A) x (D-A)
    cross = vector_cross(AC, AD)
    
    # Dot product (B-A) . cross
    dot = vector_dot(AB, cross)
    
    # Absolute value and scaling
    return abs(dot) / 6.0

def vector_sub(v1, v2):
    return (v1[0]-v2[0], v1[1]-v2[1], v1[2]-v2[2])

def vector_cross(v1, v2):
    return (
        v1[1]*v2[2] - v1[2]*v2[1],
        v1[2]*v2[0] - v1[0]*v2[2],
        v1[0]*v2[1] - v1[1]*v2[0]
    )

def vector_dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]
```
## Subtopics

### [Planar Contour Tiling](10346-planar-contour-tiling.md)
A global optimization method using graph theory to find the optimal surface, contrasting with Keppel's local greedy heuristic.

## Operating characteristics
- **Time Complexity**: Not given in the sources. Likely $O(N)$ per contour segment pair, where $N$ is the number of vertices, assuming interior points are pre-computed.
- **Memory Complexity**: Not given in the sources. Requires storage for contour vertices and interior points.
- **Strengths**: Simple to implement locally; computationally inexpensive per step.
- **Limitations**: Requires significant pre-processing to decompose contours into convex/concave parts and compute interior points [S1, unofficial]. Does not guarantee global optimality.
- **When to use**: When pre-processing is acceptable and a quick, local tiling solution is sufficient.
- **When not to use**: When global surface optimality is required or when contours are complex and difficult to decompose.
## Background
The Keppel Heuristic was described by Keppel, as cited in Ganapathy and DeRose [GD82] [S1, unofficial]. It is one of the major heuristic methods for contour tiling, alongside Fuchs and Christianson/Ganapathy heuristics [S1, unofficial].
## Sources
[S1] An Overview of Rendering from Volume Data --- including Surface and Volume Rendering, https://core.ac.uk/download/63614.pdf, fetched 2026-10-30 (unofficial)
