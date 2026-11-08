# Gray Transform (G_o, G_s)

Also called: Surface to Density Conversion

## Overview
The Gray Transform converts a discrete object description or a continuous surface description into a volumetric scene (density field) suitable for physical modeling, manufacturing, or volume rendering [S1]. It distinguishes itself from sibling methods by explicitly defining density values based on interior/exterior status or signed distance to a surface, rather than relying on implicit volume generation or alpha-compositing parameters directly [S1, 1282-sdf-to-density-conversion.md].

## How it works
**Inputs:**
1.  **Object $Q$**: A set of voxels or discrete elements representing an object [S1].
2.  **Surface $S$**: A closed surface partitioning space into interior and exterior [S1].
3.  **Scene Domain $V$**: A voxel grid containing the object/surface [S1].

**Outputs:**
*   **Scene $(V, f)$**: A density function $f: V \rightarrow \mathbb{R}$ where $f(v)$ is the density at voxel $v$ [S1].

**Parameters:**
*   $H$: Density value for interior voxels (High) [S1].
*   $L$: Density value for exterior voxels (Low) [S1].
*   $L_0$: Density value for voxels intersecting the surface [S1].
*   $d_l(v, S)$: Distance measure from voxel $v$ to surface $S$ [S1].

**Data Structures:**
*   The scene is represented as a 3D array (or 2D slice) of density values.
*   For $G_o$, the input $Q$ is a boolean mask or set of indices.
*   For $G_s$, the input $S$ is a geometric representation (e.g., mesh, implicit function) allowing distance calculation [S1].

**Steps:**

1.  **Define Scene Domain $V$**: Ensure $V$ is large enough to contain the entire object $Q$ or surface $S$ [S1].
2.  **Compute Density $f(v)$**:
    *   **For Object $G_o(Q)$**:
        *   If $v \in Q$, set $f(v) = H$ [S1].
        *   If $v \in V - Q$, set $f(v) = L$ [S1].
        *   Formula:
            $$
            f(v) = \begin{cases} 
            H & \text{if } v \in Q \\
            L & \text{if } v \in V - Q 
            \end{cases}
            $$
            (Equation 29 in [S1])
    *   **For Surface $G_s(S)$**:
        *   Calculate $d_l(v, S)$, the distance from $v$ to $S$. This can be 2D distance within the slice containing $v$ or 3D distance [S1].
        *   Determine if $v$ is in the interior, exterior, or intersecting $S$.
        *   Formula:
            $$
            f(v) = \begin{cases} 
            d_l(v, S) & \text{if } v \text{ is in the interior of } S \\
            L_0 - d_l(v, S) & \text{if } v \text{ is in } V \text{ but in the exterior of } S \\
            L_0 & \text{if } v \in V \text{ and intersects } S 
            \end{cases}
            $$
            (Equation 30 in [S1])
3.  **Return Scene $(V, f)$**.

**Pitfalls:**
*   **Closed Surface Requirement**: For $G_s$ to be well-defined, the surface $S$ must be closed, meaning it partitions $V$ into distinct interior and exterior sets such that one cannot move from interior to exterior without crossing $S$ [S1]. If $S$ is not closed, $G_s$ is undefined [S1].
*   **Distance Metric**: The choice between 2D and 3D distance $d_l(v, S)$ affects the resulting density profile and should be chosen based on the application (e.g., slice-based processing vs. full volume processing) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gray_transform_object(Q, V, H, L):
    """
    Converts a discrete object Q into a density scene over domain V.
    Q: Set of voxel indices belonging to the object.
    V: List of all voxel indices in the scene domain.
    H: Density value for interior (in Q).
    L: Density value for exterior (not in Q).
    """
    scene = {}
    for v in V:
        if v in Q:
            scene[v] = H
        else:
            scene[v] = L
    return scene

def gray_transform_surface(S, V, L0):
    """
    Converts a closed surface S into a density scene over domain V.
    S: A surface object providing methods for distance and containment.
    V: List of all voxel indices in the scene domain.
    L0: Density value for voxels intersecting the surface.
    """
    scene = {}
    for v in V:
        # Calculate distance from v to surface S
        dist = S.distance(v)
        
        # Determine spatial relationship
        is_interior = S.is_interior(v)
        is_exterior = S.is_exterior(v)
        intersects = S.intersects(v) # Or check if distance is effectively zero within tolerance
        
        if intersects:
            scene[v] = L0
        elif is_interior:
            scene[v] = dist
        elif is_exterior:
            scene[v] = L0 - dist
        else:
            # Fallback for ambiguous cases, though closed surfaces should partition space
            scene[v] = L0 
    return scene
```

## Subtopics

### Object/Surface Filtering Composite
A composite operator that converts an object or surface into a scene, filters the scene, and extracts the filtered object or surface [S1]. It solves the problem of smoothing or modifying structures while managing the relationship between the original and filtered representations [S1].
*   **Steps**:
    1.  Convert object $Q$ to scene $G_o(Q)$ using Equation 29, or surface $S$ to scene $G_s(S)$ using Equation 30 [S1].
    2.  Filter the scene using a filter $F_s$ (e.g., smoothing) [S1].
    3.  Determine the filtered object $Q'$ from the filtered scene $F_s(G_o(Q))$ or filtered surface $S'$ from the filtered scene using a surface detector $B_s$ [S1].
*   **Parameters**: If $F_s$ is a smoothing filter, the extraction operator $O_s$ (for objects) may be a threshold operator. The threshold value is determined from knowledge of $L$ and $H$ in Equation 29 [S1].
*   **Pitfalls**: The filtered surface $S'$ and a surface detected from the filtered object $Q'$ may differ, even if they represent the same underlying continuous surface, due to different filtering effects on the scene representation [S1].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of voxels in $V$, $O(|V|)$, as each voxel is processed independently [S1].
*   **Memory Complexity**: $O(|V|)$ to store the density field [S1].
*   **Strengths**: Provides a direct physical model of surfaces/objects suitable for manufacturing (e.g., CNC milling) and volume rendering [S1]. $G_s$ captures shape aspects better than $G_o$ for interpolation [S1].
*   **Limitations**: Requires a closed surface for $G_s$ [S1]. The choice of distance metric (2D vs 3D) impacts accuracy and computational cost [S1].
*   **When to use**: When a discrete or geometric description needs to be converted into a volumetric density field for further processing (filtering, interpolation, rendering) [S1].

## Background
The Gray Transform was introduced in the context of imaging transforms for visualizing surfaces and volumes [S1]. It is part of a broader framework of structure-related operations including digitization, interpolation, and filtering [S1]. The method is particularly relevant in applications such as prosthesis design and radiation therapy planning, where continuous geometric objects must be converted into digital representations for manufacturing or simulation [S1].

## Sources
[S1] Imaging transforms for visualizing surfaces and volumes, https://doi.org/10.1007/bf03168529, fetched 2026-10-29
