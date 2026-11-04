# Wilhelms and Van Gelder Octree
Also called: Wilhelms-Van Gelder method, Octree-based cell extraction

## Overview
A hierarchical spatial subdivision method using an octree to accelerate isosurface extraction by pruning branches that cannot contain the isosurface [S1, unofficial]. It differs from uniform grid methods by adapting resolution to data density and from sibling octree methods by relying on strict value-range bounding intervals for early termination during traversal [S1, unofficial].

## How it works
**Inputs:** Volume data $V$, isovalue $t$, octree structure.
**Outputs:** Set of cells intersecting the isosurface $t$.

**Data Structures:**
Each node $n$ in the octree corresponds to a rectilinear block $B$ of cells $C_{i,j,k}$. Each node is attributed an interval $[n_{min}, n_{max})$, defined as the union of all intervals $[v_{min}, v_{max})$ of the cells belonging to the block.
$$ [n_{min}, n_{max}) = \bigcup_{C_{i,j,k} \in B} [v_{min}, v_{max}) $$
where for each cell $C_{i,j,k}$:
$$ v_{min} = \min_{V_{i,j,k}}, \quad v_{max} = \max_{V_{i,j,k}} $$
This interval represents the range of values the interpolation function $\hat{f}$ attains in the portion of the volume corresponding to node $n$ [S1, unofficial].

**Algorithm Steps:**
1.  **Build Octree:** Construct the octree from the volume data. For each node, compute $n_{min}$ and $n_{max}$ by aggregating the min/max values of its child cells or children nodes.
2.  **Recursive Traversal:** Start at the root node.
3.  **Pruning:** For a current node $n$, check if the isovalue $t$ lies within the node's bounding interval $[n_{min}, n_{max})$.
    *   If $t \notin [n_{min}, n_{max})$, ignore this branch entirely [S1, unofficial].
    *   If $t \in [n_{min}, n_{max})$, proceed to children (if internal) or process cells (if leaf).
4.  **Leaf Processing:** For leaf nodes $n$ where $t \in [n_{min}, n_{max})$, investigate cells cell-by-cell using the standard enumeration algorithm (e.g., Marching Cubes) to extract the isosurface geometry [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_isosurface_octree(node, t):
    """
    Recursively traverse the octree to find cells intersecting isovalue t.
    node: OctreeNode with fields .min_val, .max_val, .is_leaf, .children, .cells
    t: float, the isovalue
    Returns: List of cells intersecting the isosurface
    """
    # Pruning step: if t is outside the range of values in this node's block,
    # no cell in this block can contain the isosurface.
    if not (node.min_val <= t < node.max_val):
        return []

    if node.is_leaf:
        # Leaf node: process cells individually
        intersecting_cells = []
        for cell in node.cells:
            # Check if cell contains isovalue t
            if cell.min_val <= t < cell.max_val:
                intersecting_cells.append(cell)
        return intersecting_cells
    else:
        # Internal node: recurse into children
        all_cells = []
        for child in node.children:
            all_cells.extend(extract_isosurface_octree(child, t))
        return all_cells
```

**Parameters:**
*   $t$: Isovalue. Typical values depend on the dataset.
*   Octree depth: Determined by the desired resolution or data characteristics. Not given in the sources.

**Edge Cases:**
*   **Precision:** Floating-point comparisons for $t \in [n_{min}, n_{max})$ must handle boundary conditions carefully. The interval is half-open $[n_{min}, n_{max})$ [S1, unofficial].
*   **Degenerate Input:** If $n_{min} = n_{max}$, the interval is empty or a point. If $t = n_{min}$, it is included; if $t = n_{max}$, it is excluded.

## Subtopics

### [View-Dependent Isosurface Extraction](2309-view-dependent-isosurface-extraction.md)
A method that extracts and renders only the visible portions of an isosurface to reduce polygon count and rendering time, exploiting depth complexity and occlusion in large datasets. It differs from non-view-dependent isocontouring by skipping internal sections that are occluded by the external isosurface, and from Greene's method by marking triangles as visible in software and forwarding them to graphics hardware for final visibility determination rather than rendering the visible portion in software.

## Operating characteristics
*   **Time Complexity:** Linear time complexity $O(n)$ with respect to the size of the volume data in the worst case, but considerable acceleration is achieved in practice by ignoring irrelevant branches [S1, unofficial].
*   **Memory Complexity:** Moderate additional CPU storage space is required to store the octree structure and the bounding intervals for each node [S1, unofficial].
*   **Strengths:** Accelerates cell extraction by avoiding unnecessary processing of cells that do not contain the isosurface.
*   **Limitations:** Requires additional memory for the octree. Performance gain depends on the sparsity of the isosurface relative to the volume.

## Background
Proposed by Wilhelms and Van Gelder [22] (cited in [S1, unofficial]) as a method to accelerate cell extraction using hierarchical spatial subdivision. The method is discussed in the context of optimal hybrid memory constrained isosurface extraction [S1, unofficial].

## Sources
[S1] Optimal hybrid memory constrained isosurface extraction, http://nbn-resolving.de/urn:nbn:de:bsz:352-130091, fetched 2026-10-30 (unofficial)
