# Monotone Path Construction in Trilinear Cells

Also called: Axis-Parallel Monotone Path

## Overview
Monotone Path Construction is an algorithm used to determine the start (low) and end (high) vertices of a monotone path passing through a specific sample point within a trilinear cell [S1, unofficial]. It solves the problem of identifying the relevant arcs in a contour tree by mapping a continuous sample position to discrete mesh vertices, enabling the traversal of the contour tree's branch decomposition [S1, unofficial]. It differs from analytic methods by relying on a deterministic, axis-parallel greedy descent/ascent strategy rather than solving polynomial roots for the exact intersection [S1, unofficial].

## How it works
**Inputs:**
*   $t$: The trilinear interpolant function defined on the domain $[0, 1]^3 \rightarrow \mathbb{R}$ [S1, unofficial].
*   $p$: The sample position vector $(x, y, z)$ inside the trilinear cell, where $p \in [0, 1]^3$ [S1, unofficial].

**Outputs:**
*   $p_{lo}$: The position of the mesh vertex representing the lower endpoint of the monotone path [S1, unofficial].
*   $p_{hi}$: The position of the mesh vertex representing the higher endpoint of the monotone path [S1, unofficial].

**Helper Functions:**
*   $\text{roundDown}(\text{axis}, p)$: Returns a new position vector identical to $p$, except the component corresponding to $\text{axis}$ is replaced with $0$ [S1, unofficial].
*   $\text{roundUp}(\text{axis}, p)$: Returns a new position vector identical to $p$, except the component corresponding to $\text{axis}$ is replaced with $1$ [S1, unofficial].

**Algorithm Steps:**

1.  **Initialize along X-axis:**
    *   Set $p_{lo} = \text{roundDown}(x, p)$.
    *   Set $p_{hi} = \text{roundUp}(x, p)$.
    *   If $t(p_{lo}) > t(p_{hi})$, swap $p_{lo}$ and $p_{hi}$ to ensure $p_{lo}$ is the lower-value endpoint [S1, unofficial].

2.  **Complete descending path to mesh vertex along Y-axis (for $p_{lo}$):**
    *   Set $\text{temp} = \text{roundUp}(y, p_{lo})$.
    *   Set $p_{lo} = \text{roundDown}(y, p_{lo})$.
    *   If $t(p_{lo}) > t(\text{temp})$, swap $p_{lo}$ and $\text{temp}$ [S1, unofficial].

3.  **Complete descending path to mesh vertex along Z-axis (for $p_{lo}$):**
    *   Set $\text{temp} = \text{roundUp}(z, p_{lo})$.
    *   Set $p_{lo} = \text{roundDown}(z, p_{lo})$.
    *   If $t(p_{lo}) > t(\text{temp})$, swap $p_{lo}$ and $\text{temp}$ [S1, unofficial].

4.  **Complete ascending path to mesh vertex along Y-axis (for $p_{hi}$):**
    *   Set $\text{temp} = \text{roundUp}(y, p_{hi})$.
    *   Set $p_{hi} = \text{roundDown}(y, p_{hi})$.
    *   If $t(p_{hi}) < t(\text{temp})$, swap $p_{hi}$ and $\text{temp}$ [S1, unofficial].

5.  **Complete ascending path to mesh vertex along Z-axis (for $p_{hi}$):**
    *   Set $\text{temp} = \text{roundUp}(z, p_{hi})$.
    *   Set $p_{hi} = \text{roundDown}(z, p_{hi})$.
    *   If $t(p_{hi}) < t(\text{temp})$, swap $p_{hi}$ and $\text{temp}$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def round_down(axis, p):
    """Returns p with the component corresponding to axis set to 0."""
    if axis == 'x':
        return (0, p[1], p[2])
    elif axis == 'y':
        return (p[0], 0, p[2])
    elif axis == 'z':
        return (p[0], p[1], 0)

def round_up(axis, p):
    """Returns p with the component corresponding to axis set to 1."""
    if axis == 'x':
        return (1, p[1], p[2])
    elif axis == 'y':
        return (p[0], 1, p[2])
    elif axis == 'z':
        return (p[0], p[1], 1)

def monotone_path_construction(t, p):
    """
    t: function mapping (x,y,z) to scalar value
    p: tuple (x, y, z) in [0,1]^3
    Returns: (p_lo, p_hi)
    """
    # Step 1: Initialize along X
    p_lo = round_down('x', p)
    p_hi = round_up('x', p)
    
    if t(p_lo) > t(p_hi):
        p_lo, p_hi = p_hi, p_lo
        
    # Step 2: Descending path for p_lo along Y
    temp = round_up('y', p_lo)
    p_lo = round_down('y', p_lo)
    if t(p_lo) > t(temp):
        p_lo = temp
        
    # Step 3: Descending path for p_lo along Z
    temp = round_up('z', p_lo)
    p_lo = round_down('z', p_lo)
    if t(p_lo) > t(temp):
        p_lo = temp
        
    # Step 4: Ascending path for p_hi along Y
    temp = round_up('y', p_hi)
    p_hi = round_down('y', p_hi)
    if t(p_hi) < t(temp):
        p_hi = temp
        
    # Step 5: Ascending path for p_hi along Z
    temp = round_up('z', p_hi)
    p_hi = round_down('z', p_hi)
    if t(p_hi) < t(temp):
        p_hi = temp
        
    return p_lo, p_hi
```

**Edge Cases and Pitfalls:**
*   **Degenerate Values:** If $t(p_{lo}) == t(p_{hi})$ or intermediate comparisons result in equality, the swap condition is false, and the current vertex is retained. The algorithm assumes strict inequality for swapping to maintain the monotone property [S1, unofficial].
*   **Boundary Conditions:** The `roundDown` and `roundUp` functions explicitly map to 0 or 1, ensuring the output vertices are always mesh vertices (corners of the cell) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Constant time $O(1)$, as it performs a fixed number of arithmetic operations and function evaluations regardless of cell resolution.
*   **Memory Complexity:** Constant space $O(1)$, requiring storage for a few temporary vectors.
*   **Strengths:** Deterministic and fast; suitable for real-time topology extraction [S1, unofficial].
*   **Limitations:** It constructs an *approximate* monotone path to vertices rather than finding the exact isosurface intersection. It is specifically designed for contour tree navigation, not for precise geometric ray-casting [S1, unofficial].

## Background
The method was introduced in the context of topology-controlled volume rendering to facilitate the construction of contour trees [S1, unofficial]. By finding a monotone path in the cell, the algorithm identifies the arc in the corresponding subgraph of the contour tree that contains the sample point [S1, unofficial]. This allows the renderer to walk from the high arc towards the low arc in the branch decomposition (a rooted tree) to find the path between branches [S1, unofficial].

## Sources

[S1] Topology-Controlled Volume Rendering, https://www.computer.org/csdl/journal/tg/2007/02/04069241/13rRUytF41s, fetched 2026-10-31 (unofficial)
