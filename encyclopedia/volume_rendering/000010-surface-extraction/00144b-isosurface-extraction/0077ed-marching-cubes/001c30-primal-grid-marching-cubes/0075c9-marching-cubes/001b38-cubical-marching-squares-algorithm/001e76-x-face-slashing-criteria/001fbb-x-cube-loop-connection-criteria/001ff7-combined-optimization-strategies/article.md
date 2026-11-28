# Combined optimization strategies

Also called: 1-a, 2-b, 3-c, 4-d

## Overview
Combined optimization strategies are four specific combinations of X-face slashing and X-cube loop connection criteria used to optimize the topological and combinatorial properties of the resulting isosurface mesh [S1]. They resolve the ambiguity in how to handle X-faces and X-cubes by prioritizing specific mesh metrics: minimizing triangles ($T$), maximizing triangles, maximizing shells ($S$), or quasi-minimizing shells [S1].

## How it works
The notation $\min(W)$ denotes the subset of all meshes $M$ such that the metric $W(m)$ achieves its minimum value for all $m$ in that subset [S1]. Similarly, $\max(W)$ denotes the maximum [S1]. Where a global maximum or minimum cannot be guaranteed, the terms High and Low are used [S1].

### Inputs and Outputs
*   **Input:** The state of the X-face graph and the set of X-cubes with free loops [S1].
*   **Output:** A triangulated isosurface mesh with optimized topological properties [S1].

### Data Structures
The implementation maintains the mesh topology variables:
*   $V$: Number of vertices.
*   $T$: Number of triangles.
*   $H$: Number of handles.
*   $S$: Number of shells (connected components of the boundary).
*   $L$: Number of free loops in X-cubes before connection.
*   $s$: Number of loops actually connected.

### Formulas
The relationships between these variables are governed by:
$$ T = 4V + 2L - 4s $$
$$ T = 2V + 4(H - S) $$
From these, it follows that:
$$ 4(H - S) = 2V + 2L - 4s $$

### Steps and Strategies
The four strategies are defined by their optimization goals and the resulting constraints on $L$ and $s$.

**1. Strategy 1-a: Minimize T**
*   **Goal:** Minimize the number of triangles $T$ and minimize $H - S$ [S1].
*   **Logic:** Maximize $L$ (the number of free loops) and do not connect any loops ($s = L$) [S1].
*   **Derivation:** Substituting $s = L$ into the triangle formula:
    $$ T = 4V + 2L - 4L = 4V - 2L $$
    Maximizing $L$ leads to $\min(T)$ [S1].
*   **Topological Consequence:** Since $T = 2V + 4(H - S)$, minimizing $T$ implies minimizing $H - S$ [S1]. This results in Low($H$) and High($S$) [S1].

**2. Strategy 2-b: Maximize T**
*   **Goal:** Maximize the number of triangles $T$ and maximize $H - S$ [S1].
*   **Logic:** Minimize $L$ [S1]. Since $s \le L$, connecting loops decreases $S$ [S1].
*   **Derivation:**
    $$ 4(H - S) = 2V + 2L - 4s \ge 2V - 2L $$
    Minimizing $L$ and connecting loops (lowering $L$ and increasing $s$) leads to High($H - S$) [S1].
*   **Topological Consequence:** This results in High($H$) and Low($S$) [S1].

**3. Strategy 3-c: Maximize S**
*   **Goal:** Maximize the number of shells $S$ [S1].
*   **Logic:** Do not take any decision that reduces the number of shells if it is possible to avoid it [S1]. Criterion $c$ connects parts of the boundary already in the same component [S1].
*   **Derivation:** Connecting parts of the same component does not change $S$ but increases $T$ [S1]. Since $T = 2V + 4(H - S)$, an increase in $T$ with constant $S$ and $V$ requires an increase in $H$ [S1].
*   **Topological Consequence:** This results in High($H$) [S1].

**4. Strategy 4-d: Quasi-minimize S**
*   **Goal:** Tend toward $\min(S)$ with Low($H$) [S1].
*   **Logic:** Always connect disconnected equivalence classes [S1].
*   **Limitation:** This strategy may not achieve the global minimum $S$ because ties are resolved randomly [S1]. If two slashing choices for an X-face yield the same number of equivalence classes, a random choice is made [S1]. This may connect vertices that could have been connected elsewhere, failing to connect vertices that could *only* be connected at that specific X-face [S1]. This event is rare but possible [S1].

### Edge Cases and Pitfalls
*   **Global Optimality:** The Max/Min global optimal values for $T$, $H$, or $S$ are reached provided that the X-face graph has no cycles [S1]. In most practical cases, the X-face graph is acyclic, so optimality is achieved [S1].
*   **Tie Situations:** Ties occur when two slashing choices for an X-face fulfill a given criterion equally well [S1].
    *   *Single Magnitude:* If targeting only one metric, a simple random selection can resolve the tie, producing a random mesh from the set of optimal meshes [S1].
    *   *Multiple Magnitudes:* A more robust approach uses an ordered pair of combined criteria to optimize one magnitude while keeping another small [S1].
    *   *Parallel Analysis:* Another approach is to analyze the two candidate paths in parallel [S1].
*   **Internal Vertices:** When using criteria 2 and 3, representing the intersection of two or more tunnels may require the introduction of internal vertices [S1].
*   **Triangulation Artifacts:** The implementation triangulates each sheet so that no triangle lies on the faces of the cubes [S1]. These faces would produce undesirable artifacts when actual data values are used for interpolating the vertex position along the grid edges [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_combined_strategy(strategy_id, x_faces, x_cubes):
    """
    Applies one of the four combined optimization strategies.
    Args:
        strategy_id: One of '1-a', '2-b', '3-c', '4-d'.
        x_faces: List of X-face objects with slashing choices.
        x_cubes: List of X-cube objects with free loops.
    Returns:
        Triangulated mesh.
    """
    mesh = []  # Initialize mesh structure [G]
    for x_face in x_faces:
        choices = x_face.get_slashing_choices()
        if strategy_id == '1-a':
            # Minimize T: Maximize L, do not connect loops (s = L)
            choice = max(choices, key=lambda c: c.loops_added)  # Evaluate impact on L [G]
            x_face.apply_slash(choice)
        elif strategy_id == '2-b':
            # Maximize T: Minimize L, connect loops
            choice = min(choices, key=lambda c: c.loops_added)  # Evaluate impact on L [G]
            x_face.apply_slash(choice)
        elif strategy_id == '3-c':
            # Maximize S: Do not reduce shells
            choice = select_choice_preserving_components(choices)  # Criterion c connects parts of boundary already in same component [S1]
            x_face.apply_slash(choice)
        elif strategy_id == '4-d':
            # Quasi-minimize S: Connect disconnected equivalence classes
            choice = select_choice_connecting_disconnected_classes(choices)
            if is_tie(choices):  # Ties occur when two slashing choices fulfill a given criterion equally well [S1]
                import random
                choice = random.choice(choices)  # Simple random selection resolves tie for single magnitude [S1]
            x_face.apply_slash(choice)
    for x_cube in x_cubes:
        loops = x_cube.get_free_loops()
        if strategy_id == '1-a':
            x_cube.keep_loops_unconnected(loops)  # Do not connect loops (s = L) [S1]
        elif strategy_id == '2-b':
            x_cube.connect_loops(loops)  # Connect loops to decrease S [S1]
        elif strategy_id == '3-c':
            x_cube.apply_criterion_c(loops)  # Connect if it doesn't reduce S [S1]
        elif strategy_id == '4-d':
            x_cube.connect_disconnected_classes(loops)  # Connect disconnected equivalence classes [S1]
    triangulate_sheets(mesh)  # Triangulate each sheet so no triangle lies on cube faces [S1]
    return mesh

def select_choice_preserving_components(choices):
    # Evaluate connectivity for each choice [G]
    # Return choice that merges existing components rather than splitting [G]
    return choices[0]

def select_choice_connecting_disconnected_classes(choices):
    # Evaluate equivalence classes for each choice [G]
    # Return choice that connects different classes [G]
    return choices[0]

def is_tie(choices):
    # Check if multiple choices yield identical metric values [G]
    return False

def triangulate_sheets(mesh):
    # Triangulate each sheet so no triangle lies on cube faces [S1]
    # These faces would produce undesirable artifacts when actual data values are used for interpolating the vertex position along the grid edges [S1]
    pass
```

## Operating characteristics
*   **Complexity:** The complexity is determined by the traversal of the X-face graph and X-cubes [S1]. The global optimality is guaranteed if the X-face graph is acyclic [S1].
*   **Performance:** The strategies are implemented and tested, with results discussed in the source literature [S1].
*   **Strengths:** Provides explicit control over mesh topology ($T, H, S$) [S1]. Strategy 1-a is optimal for minimizing triangle count [S1]. Strategy 4-d is effective for reducing shell count in most practical cases [S1].
*   **Limitations:** Strategy 4-d is quasi-optimal and may fail to achieve the global minimum $S$ due to random tie-breaking [S1]. Strategies 2 and 3 may require internal vertices for complex tunnel intersections [S1]. The other 12 combined decisions (1-b, 1-c, etc.) are not defined as they involve conflicting decisions [S1].

## Background
The Combined optimization strategies were introduced in the paper "Optimizing the topological and combinatorial complexity of isosurfaces" [S1]. The work builds upon the X-face slashing criteria and X-cube loop connection criteria [Parent]. The notation and theoretical framework for $T, H, S, L, s$ are derived from this source [S1].

## Sources
[S1] Optimizing the topological and combinatorial complexity of isosurfaces, https://doi.org/10.1016/j.cad.2004.09.013, fetched 2026-10-30
