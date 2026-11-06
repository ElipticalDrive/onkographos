# Multi-Resolution Dual Marching Cubes with Manifold Preservation

Also called: MR-DMC

## Overview
Multi-Resolution Dual Marching Cubes with Manifold Preservation (MR-DMC) is a surface extraction method that generates closed, manifold meshes from segmented volumetric data using a generalized octree [S1, unofficial]. It extends Dual Marching Cubes (DMC) by introducing a cell-based criterion for dual vertex generation to prevent non-manifold configurations, specifically addressing ambiguities where standard DMC cases 17 and 20 share an ambiguous face [S1, unofficial]. Unlike Manifold Dual Contouring (MDC), which uses a bottom-up strategy, MR-DMC employs a top-down refinement strategy suitable for large datasets, and unlike standard DMC, it guarantees manifold surfaces without post-processing [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Indicator function $F(\mathbf{x}): \mathbb{R}^3 \rightarrow \mathbb{R}$, where $F(\mathbf{x}) = 1$ if $\mathbf{x}$ is inside the segmented object and $0$ otherwise [S1, unofficial].
*   Root cell of the octree $c$ [S1, unofficial].
*   `minLevel`: Minimal subdivision level of the octree [S1, unofficial].
*   `maxLevel`: Maximal subdivision level of the octree [S1, unofficial].
*   $\delta$: Curvature threshold parameter for subdivision, $\delta \in [0, 1]$ [S1, unofficial].

**Outputs:**
*   An octree regular until level `minLevel` and adaptive between `minLevel` and `maxLevel` [S1, unofficial].
*   A multi-resolution, closed manifold surface mesh [S1, unofficial].

### Data Structures
**Octree Cell Data:**
Each octree cell stores:
*   Initial and final cell coordinates [S1, unofficial].
*   Cell corner values (values of $F(\mathbf{x})$ at the 8 vertices) [S1, unofficial].
*   A collapsed flag [S1, unofficial].
*   Morton Code: Used for compact integer representation of hierarchical relationships and position [S1, unofficial].
*   A `toProcess` list (or queue) for iterative traversal [S1, unofficial].

### Definitions
*   **Cell $c$**: A graph $c = \{V, E\}$ with eight vertices $V = \{v_0, \dots, v_7\}$ and twelve edges $E = \{e_0, \dots, e_{11}\}$ with cube connectivity [S1, unofficial].
*   **Solid Vertex**: A vertex where $F(\mathbf{x}) = 1$. The set of solid vertices in cell $c$ is $S(c)$ [S1, unofficial].
*   **Empty Vertex**: A vertex where $F(\mathbf{x}) = 0$. The set of empty vertices in cell $c$ is $E(c)$ [S1, unofficial].
*   **Solid-Edge**: An edge with two solid endpoints [S1, unofficial].
*   **Empty-Edge**: An edge with two empty endpoints [S1, unofficial].
*   **Intersection-Edge**: An edge with one solid and one empty endpoint [S1, unofficial].
*   **Ambiguous Face**: A face containing two interleaved solid vertices [S1, unofficial].
*   **Connected Component $\phi(c)_s$**: A sub-graph of solid vertices in $c$ where all two solid vertices are connected by a path of solid-edges. $|\phi(c)_s|$ is the number of solid connected components [S1, unofficial].
*   **Connected Component $\phi(c)_e$**: A sub-graph of empty vertices in $c$ where all two empty vertices are connected by a path of empty-edges. $|\phi(c)_e|$ is the number of empty connected components [S1, unofficial].
*   **Dual Vertices $D(c)$**: The set of dual vertices generated inside cell $c$ [S1, unofficial].
*   **Complex Face**: A face that intersects $F(\mathbf{x})$, has all vertices either inside or outside $F(\mathbf{x})$, AND at least one of its edges intersects $F(\mathbf{x})$ more than once [S1, unofficial].
*   **Complex Cell**: A cell $C$ is complex if at least one of its faces is complex [S1, unofficial].
*   **Homogeneous Cell**: A cell where all corners are either inside or outside $F(\mathbf{x})$ [S1, unofficial].
*   **Non-Homogeneous Cell**: A cell that is not homogeneous [S1, unofficial].

### Algorithm Steps

#### 1. Octree Construction (Top-Down Refinement)
The octree is built iteratively. Cells are subdivided if they are inhomogeneous or complex, unless they have reached `maxLevel`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_octree(root_cell, min_level, max_level, delta):
    to_process = [root_cell]
    
    while to_process is not empty:
        current_cell = get_first_element(to_process)
        
        # Check if we can subdivide further
        if get_level(current_cell) < max_level:
            # Check if cell is complex
            if is_not_complex(current_cell):
                # Check if cell is planar (homogeneous or no surface intersection)
                if is_not_planar(current_cell):
                    # Cell is a leaf, mark as collapsed
                    mark_as_leaf(current_cell)
                    mark_as_collapsed(current_cell)
                else:
                    # Cell is inhomogeneous but not complex -> Subdivide
                    subdivide(current_cell)
            else:
                # Cell is complex -> Subdivide
                subdivide(current_cell)
        else:
            # Reached max level -> Subdivide (or mark as leaf depending on implementation, 
            # source says "Goto Subdivide" which implies processing children or marking)
            # Based on "Cells that have not been subdivided until the maximal octree depth 
            # are marked as collapsed", we likely mark as leaf/collapsed here if not already handled.
            # However, Algorithm 1 explicitly says "Goto Subdivide". 
            # Interpretation: If max_level is reached, we stop subdividing geometrically, 
            # but the logic flow in the source suggests handling the cell. 
            # Given the pitfall note, cells at max depth are collapsed.
            mark_as_leaf(current_cell)
            mark_as_collapsed(current_cell)

def subdivide(cell):
    # Divide cell into 8 children
    children = divide(cell)
    for child in children:
        add_to_process(child)

def is_not_complex(cell):
    # A cell is complex if at least one face is complex.
    # A face is complex if:
    # 1. It intersects F(x)
    # 2. All its vertices are either inside or outside F(x)
    # 3. At least one of its edges intersects F(x) more than once.
    for face in cell.faces:
        if is_complex_face(face):
            return False
    return True

def is_complex_face(face):
    if not intersects_surface(face):
        return False
    if not all_vertices_same_side(face):
        return False
    if not any_edge_intersects_more_than_once(face):
        return False
    return True

def is_not_planar(cell):
    # Returns True if cell is homogeneous (all vertices same side)
    # Or if no intersection edges exist? 
    # Source: "if isNotPlanar(currentCell) then Subdivide ... else Mark ... collapsed"
    # Planar usually means no surface intersection or homogeneous.
    # If it is NOT planar, it has surface intersection and is inhomogeneous.
    return is_homogeneous(cell) or no_intersection_edges(cell)

def is_homogeneous(cell):
    return len(cell.solid_vertices) == 0 or len(cell.solid_vertices) == 8

def no_intersection_edges(cell):
    return len(cell.intersection_edges) == 0
```

*Note on `isNotPlanar`: The source pseudocode uses `isNotPlanar` to trigger subdivision. In the context of DMC, a "planar" cell often refers to a cell where the isosurface does not pass through it or it is homogeneous. If a cell is inhomogeneous (has both solid and empty vertices), it is not planar and must be subdivided unless it is complex (handled above). The source states: "If a cell C is inhomogeneous or is complex, it has to be subdivided." The pseudocode checks `isNotComplex` first. If not complex, it checks `isNotPlanar`. If `isNotPlanar` is true (i.e., it is inhomogeneous), it subdivides. If `isNotPlanar` is false (i.e., it is planar/homogeneous), it marks as leaf/collapsed. This aligns with Rule 5.*

#### 2. Multi-Resolution Refinement (Curvature Criterion)
After the base octree is built (or during refinement), cells are subdivided based on curvature to adapt resolution.

**Criterion 1:**
Let $C$ be a cell and $I_i$ ($i=0 \dots n$) be its intersection points with $F(\mathbf{x})$. Let $n_i$ be the normals at $I_i$.
If $\max_{i \neq j} (n_i \cdot n_j) > \delta$, $C$ must be subdivided [S1, unofficial].

*   $\delta \in [0, 1]$ [S1, unofficial].
*   $\delta \to 0$: Produces an almost regular mesh [S1, unofficial].
*   $\delta \to 1$: Generates a highly simplified surface, potentially losing sharp features [S1, unofficial].

#### 3. Dual Vertex Generation (Manifold Preservation)
Dual vertices are generated in cells that intersect the surface.

**Rule 1:** Dual vertices are only created in cells where $0 < |S(c)| < 8$ [S1, unofficial].

**Rule 2:**
*   If $|S(c)| < 5$, then $|D(c)| = |\phi(c)_s|$ [S1, unofficial].
*   If $|S(c)| \ge 5$, then $|D(c)| = |\phi(c)_e|$ [S1, unofficial].

**Rule 3:** For a non-homogeneous cell, every dual vertex $d_i \in D(c)$ generated from a solid connected component $\phi(c)_s$ is associated with the set of intersection-edges that leave $\phi(c)_s$. Similarly for empty components $\phi(c)_e$ [S1, unofficial].

**Rule 4 (Manifold Preservation):**
1.  If cell $c$ is marked with a component (solid/empty), generate dual vertices using that component [S1, unofficial].
2.  Otherwise, use Rules 1-3 to determine the component (solid or empty) to be used [S1, unofficial].
3.  If $c$ has ambiguous faces, mark every adjacent cell through an ambiguous face with the component that has been used for $c$ [S1, unofficial].

*Note: This rule spreads the topological choice through ambiguous faces to ensure manifold correctness, specifically resolving cases 17 and 20 of the DMC look-up table [S1, unofficial].*

### Parameters
*   **$\delta$**: Curvature threshold, $[0, 1]$ [S1, unofficial].
*   **minLevel**: Minimal subdivision level [S1, unofficial].
*   **maxLevel**: Maximal subdivision level [S1, unofficial].

### Edge Cases and Pitfalls
*   **Non-manifold configurations**: Standard DMC can produce non-manifold edges when cases 17 and 20 of the look-up table share a common ambiguous face [S1, unofficial]. MR-DMC prevents this via Rule 4 [S1, unofficial].
*   **Sharp features**: If $\delta \to 1$, the generated surface may lose sharp features due to excessive simplification [S1, unofficial].
*   **Collapsed cells**: Cells that have not been subdivided until the maximal octree depth are marked as collapsed [S1, unofficial].
*   **Complex cells**: Complex cells (containing surface pieces smaller than the current subdivision level) must be subdivided to ensure all intersected cells are processed [S1, unofficial].

## Operating characteristics
*   **Strategy**: Top-bottom refinement, suitable for large datasets [S1, unofficial].
*   **Manifold Guarantee**: Guarantees manifold surfaces by introducing a cell-based criterion for dual vertex generation, avoiding post-processing repairs [S1, unofficial].
*   **Complexity**: Not given in the sources.
*   **Performance**: Not given in the sources.
*   **Strengths**: Generates multi-resolution, closed manifold surfaces; handles large datasets efficiently via top-down approach; no post-processing needed for manifold repair [S1, unofficial].
*   **Limitations**: High $\delta$ can lose sharp features; requires segmented volumetric data (indicator function) [S1, unofficial].

## Background
*   **Origin**: Proposed by Ricardo Uribe Lobello, Florent Dupont, and Florence Denis from Université de Lyon, CNRS, LIRIS [S1, unofficial].
*   **Publication**: Published in SCITEPRESS - Science and Technology Publications in 2012 [S1, unofficial].
*   **Context**: Addresses limitations of standard Dual Marching Cubes (which can produce non-manifold edges) and Manifold Dual Contouring (which uses a bottom-up strategy unsuitable for large datasets) [S1, unofficial].
*   **Development**: Extends DMC over a generalized octree and introduces a top-bottom refinement algorithm based on curvature [S1, unofficial].

## Sources
[S1] SCITEPRESS - SCIENCE AND TECHNOLOGY PUBLICATIONS, https://www.scitepress.org/PublishedPapers/2012/38588/, fetched 2026-10-28 (unofficial)
