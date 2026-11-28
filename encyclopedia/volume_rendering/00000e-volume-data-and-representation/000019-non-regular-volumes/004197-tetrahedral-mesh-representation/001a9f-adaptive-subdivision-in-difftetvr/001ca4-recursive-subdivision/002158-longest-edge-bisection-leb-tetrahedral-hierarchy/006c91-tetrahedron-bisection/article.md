# Tetrahedron Bisection

Also called: Recursive Bisection, Red-Refinement

## Overview
Tetrahedron bisection is a hierarchical subdivision method for non-regular volumes that recursively splits tetrahedra along their longest edge, starting from an initial decomposition of a cubic domain into six tetrahedra. It sets itself apart from sibling methods like [Supercube](18952-supercube.md) by producing a hierarchy where element quality is preserved through cyclic shape repetition, ensuring that recursive splitting yields well-shaped tetrahedra without degradation.

## How it works
**Inputs:** An initial cubic domain.
**Outputs:** A hierarchical tetrahedral mesh defined by a base mesh and a set of conforming updates.

**Data Structures:**
The hierarchy is defined by a base mesh $\mathcal{M}_b$ and a set of updates $U$. Each update $u$ is a pair $(\mathcal{F}_1, \mathcal{F}_2)$, where $\mathcal{F}_1$ is a tetrahedral cluster (tetrahedra to be removed) and $\mathcal{F}_2$ is a split-set (tetrahedra to be inserted).

**Steps:**
1.  **Initial Decomposition:** Decompose the cubic domain into six initial tetrahedra. These six tetrahedra are all of shape "1/8 pyramid" [S1].
2.  **Recursive Bisection:** For any tetrahedron in the mesh, identify its longest edge. Bisect the tetrahedron by cutting along the plane passing through the midpoint of the longest edge and the opposite edge [S1].
3.  **Shape Classification:** Regardless of the number of recursive splits, the resulting tetrahedra fall into one of three cyclic shapes [S1]:
    *   **1/2 Pyramid:** Obtained by splitting a 1/8 pyramid (or initial tetrahedron) along its longest edge. Two 1/2 pyramids share a square base diagonal [S1].
    *   **1/4 Pyramid:** Obtained by splitting a 1/2 pyramid along its longest edge [S1].
    *   **1/8 Pyramid:** Obtained by splitting a 1/4 pyramid along its longest edge. This shape is geometrically similar to the initial tetrahedra [S1].
    *   *Cycle:* Every three levels of decomposition result in a geometrically similar shape [S1].
4.  **Non-Conforming Updates:** A single bisection is a non-conforming update because it creates a hanging node (a vertex on an edge of a neighbor that is not a vertex of that neighbor) [S1].
5.  **Cluster Formation (Conforming Updates):** To maintain mesh conformity, non-conforming updates are merged into clusters. A cluster is a minimal set of non-conforming updates whose union is a conforming update [S1]. Clustering is performed by grouping tetrahedra along their splitting edge [S1]. There are three types of clusters [S1]:
    *   **Axis-aligned cluster:** Formed by eight 1/2 pyramids sharing an axis-aligned edge.
        *   Cluster size ($\mathcal{F}_1$): 4 tetrahedra.
        *   Split-set size ($\mathcal{F}_2$): 8 tetrahedra.
    *   **Plane-aligned cluster:** Formed by four 1/4 pyramids sharing a plane-aligned edge (parallel to a coordinate plane).
        *   Cluster size ($\mathcal{F}_1$): 6 tetrahedra.
        *   Split-set size ($\mathcal{F}_2$): 12 tetrahedra.
    *   **Non-aligned cluster:** Formed by twelve 1/8 pyramids sharing an edge not aligned to a coordinate axis or plane.
        *   Cluster size ($\mathcal{F}_1$): 8 tetrahedra.
        *   Split-set size ($\mathcal{F}_2$): 16 tetrahedra.
6.  **Average Cost:** On average, six tetrahedra are added by an update [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bisection(tetrahedron):
    """
    Performs a single non-conforming bisection of a tetrahedron.
    Returns the two resulting child tetrahedra.
    """
    # 1. Identify the longest edge
    edges = get_edges(tetrahedron)
    longest_edge = max(edges, key=lambda e: length(e))
    
    # 2. Find midpoint of longest edge
    mid_point = (longest_edge.v1 + longest_edge.v2) / 2.0
    
    # 3. Identify the opposite edge
    # The opposite edge connects the two vertices not in the longest edge
    opposite_edge = get_opposite_edge(tetrahedron, longest_edge)
    
    # 4. Define the cutting plane
    # Plane passes through mid_point and the opposite edge
    # This splits the tetrahedron into two new tetrahedra
    tet1, tet2 = split_along_plane(tetrahedron, mid_point, opposite_edge)
    
    return tet1, tet2

def classify_shape(tetrahedron, level):
    """
    Classifies the tetrahedron shape based on the recursion level.
    Level 0: Initial 1/8 pyramid.
    Level 1: 1/2 pyramid.
    Level 2: 1/4 pyramid.
    Level 3: 1/8 pyramid (same as Level 0).
    """
    shape_index = level % 3
    if shape_index == 0:
        return "1/8_pyramid"
    elif shape_index == 1:
        return "1/2_pyramid"
    else:
        return "1/4_pyramid"

def form_cluster(tetrahedra_to_split):
    """
    Groups non-conforming updates into a conforming cluster.
    Returns (cluster_tets, split_set_tets).
    """
    # Determine cluster type based on the alignment of the splitting edges
    # of the tetrahedra in the set.
    alignment = determine_edge_alignment(tetrahedra_to_split)
    
    if alignment == "axis_aligned":
        # 8 1/2 pyramids
        cluster_size = 4
        split_size = 8
    elif alignment == "plane_aligned":
        # 4 1/4 pyramids
        cluster_size = 6
        split_size = 12
    else:
        # 12 1/8 pyramids
        cluster_size = 8
        split_size = 16
        
    return cluster_size, split_size
```

## Subtopics

### Cluster Formation
A conforming update formed by merging a set of non-conforming updates such that their union is conforming and no proper subset is. It is chosen when mesh conformity is required after multiple bisections. The type (axis-aligned, plane-aligned, or non-aligned) depends on the geometric alignment of the splitting edges of the merged tetrahedra [S1].

## Operating characteristics
**Time and Memory Complexity:** Not given in the sources.
**Measured Performance:** Not given in the sources.
**Scaling:** The hierarchy ensures that element quality does not degrade because recursively splitting a tetrahedron three times yields smaller tetrahedra with exactly the same shape and quality as the initial one [Parent].
**Strengths:** Produces only well-shaped tetrahedra; preserves geometric similarity over cycles of three levels [Parent, S1].
**Limitations:** Single bisections are non-conforming, requiring clustering logic to maintain valid mesh topology [S1].

## Background
The method originates from the analysis of multiresolution tetrahedral meshes presented by [S1] in 2002. It is used in the context of adaptive subdivision in DiffTetVR and hierarchical tetrahedral (HT) meshes, where it serves as the fundamental subdivision rule for generating multi-resolution representations [S1, Parent].

## Sources
[S1] Multiresolution tetrahedral meshes: an analysis and a comparison, https://doi.org/10.1109/smi.2002.1003532, fetched 2026-10-26
