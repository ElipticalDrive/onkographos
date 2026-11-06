# Wilhelms and Van Gelder Octree
Also called: Wilhelms and Van Gelder method, Wilhelms and Van Gelder octree subdivision

## Overview
The Wilhelms and Van Gelder method is an octree-based isosurface extraction technique that accelerates cell enumeration by pruning branches of the spatial hierarchy that cannot contain the target isovalue [S1, unofficial]. It differs from sibling methods like [IsoOctree](1744-isooctree.md) by relying on a simple interval-based culling test during a recursive traversal rather than complex topological consistency guarantees or view-dependent visibility culling [S1, unofficial].

## How it works
**Inputs and Outputs**
- Input: A scalar volume dataset $V$ defined on a rectilinear grid, and a scalar isovalue $t$.
- Output: A set of polygons (typically triangles) representing the isosurface $\{ (x,y,z) \mid f(x,y,z) = t \}$.

**Data Structures**
The method constructs an octree where each node $n$ corresponds to a rectilinear block $B$ of cells $C_{i,j,k}$ [S1, unofficial].
- Each node $n$ stores an interval $[n_{min}, n_{max})$.
- This interval is the union of the value ranges of all cells in the block:
  $$ [n_{min}, n_{max}) = \bigcup_{C_{i,j,k} \in B} [v_{min}, v_{max}) $$
- For a single cell $C_{i,j,k}$, the range is defined by the minimum and maximum scalar values at its vertices:
  $$ [v_{min}, v_{max}) = [\min_{V_{i,j,k}}, \max_{V_{i,j,k}}] $$
  where $V_{i,j,k}$ are the vertex values of the cell [S1, unofficial].

**Algorithm Steps**
1. **Tree Construction**: Build the octree over the volume data. For each node, compute the bounding interval $[n_{min}, n_{max})$ of the scalar values contained within its spatial block [S1, unofficial].
2. **Recursive Traversal**: Start at the root node.
3. **Culling Test**: For a current node $n$, check if the isovalue $t$ lies within its interval $[n_{min}, n_{max})$.
   - If $t \notin [n_{min}, n_{max})$, the node cannot contain the isosurface. Prune this branch and return [S1, unofficial].
   - If $t \in [n_{min}, n_{max})$, the node may contain the isosurface.
4. **Leaf Processing**: If node $n$ is a leaf, process its constituent cells individually using a standard cell enumeration algorithm (e.g., Marching Cubes) to extract the surface geometry [S1, unofficial].
5. **Internal Node Processing**: If node $n$ is not a leaf, recursively apply steps 3-4 to all eight children [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wilhelms_van_gelder_extract(volume, isovalue, node):
    """
    Recursively extract isosurface from an octree node.
    
    Args:
        volume: The scalar field data.
        isovalue: The target scalar value t.
        node: The current octree node object.
              node.min_val: n_min
              node.max_val: n_max
              node.is_leaf: Boolean
              node.children: List of child nodes (if not leaf)
              node.cells: List of cells (if leaf)
    """
    # Culling test: if isovalue is outside the node's value range, skip
    if isovalue < node.min_val or isovalue >= node.max_val:
        return []

    polygons = []

    if node.is_leaf:
        # Leaf node: process cells individually using enumeration algorithm
        for cell in node.cells:
            # Extract polygons for this cell using standard method (e.g., Marching Cubes)
            cell_polys = extract_cell_surface(cell, isovalue)
            polygons.extend(cell_polys)
    else:
        # Internal node: recurse into children
        for child in node.children:
            child_polys = wilhelms_van_gelder_extract(volume, isovalue, child)
            polygons.extend(child_polys)

    return polygons
```

**Parameters**
- Typical values: Not specified in sources. The method is general for any isovalue $t$.

**Edge Cases and Pitfalls**
- **Precision**: The interval test $t \in [n_{min}, n_{max})$ assumes exact arithmetic or sufficient floating-point precision. If $t$ is very close to $n_{min}$ or $n_{max}$, precision errors might lead to incorrect culling or inclusion [G].
- **Boundaries**: The interval is defined as half-open $[n_{min}, n_{max})$ in the source [S1, unofficial]. Implementation must be consistent with this definition to avoid double-counting or missing boundaries if values are shared.
- **Degenerate Input**: If the volume has constant values, the interval width is zero. The culling test works correctly (either includes or excludes).

## Subtopics

### [View-Dependent Isosurface Extraction](2309-view-dependent-isosurface-extraction.md)
A method that extracts and renders only the visible portions of an isosurface to reduce polygon count and rendering time, exploiting depth complexity and occlusion in large datasets. It differs from non-view-dependent isocontouring by skipping internal sections that are occluded by the external isosurface, and from Greene's method by marking triangles as visible in software and forwarding them to graphics hardware for final visibility determination rather than rendering the visible portion in software.

### [Branch-on-Need Octree (BONO)](6935-branch-on-need-octree-bono.md)
The Branch-on-Need Octree (BONO) is a space-efficient geometric decomposition method for isosurface extraction that partitions volume data based on geometric positions to prune branches during traversal. Unlike even-subdivision octrees, BONO minimizes node count for non-power-of-two dimensions by aligning subdivisions to the largest possible power-of-two blocks, allowing the extraction algorithm to skip nodes whose value ranges do not span the isovalue.

### Octree Search Structure
A hierarchical spatial index used to skip analysis of inactive cells during isosurface extraction, reducing computational load at the cost of construction time and memory [S2].
- **Steps**: 1. Construct octree over the volume dataset. 2. Use octree to identify and skip inactive cells during extraction. 3. Extract isosurface only from active regions. 4. If extracting multiple surfaces, reuse the same index [S2].
- **Pitfalls**: Construction overhead makes it slower than exhaustive WWMC for single or dual surface extraction. Requires downloading 219 MB of index data if processing on different resources. Parallel construction requires global coordinator, likely yielding sublinear speedup [S2].

## Operating characteristics
- **Time Complexity**: The method achieves linear time complexity $O(n)$ with respect to the size of the volume data in the worst case, but offers considerable acceleration by ignoring branches that cannot contain the isosurface [S1, unofficial].
- **Memory Complexity**: Requires moderate additional CPU storage space for the octree structure [S1, unofficial].
- **Performance**: In experiments, using the octree allowed extraction of certain datasets (e.g., XT-180) in about one third of the time compared to exhaustive WWMC, as it could skip up to 99% of cells [S2]. However, for datasets with high active cell density (e.g., XT-25, where >50% of cells are active), the performance was comparable to WWMC [S2].
- **Trade-offs**: The construction time of the octree is significant. When including construction time, the total time for a single surface extraction is often worse than exhaustive WWMC [S2]. Reusing the index for multiple surfaces improves the amortized cost, but WWMC can still be faster in some multi-surface scenarios (165.4 sec for WWMC vs 169 sec for Octree) [S2].
- **Strengths**: Significant speedup for sparse isosurfaces or datasets with large inactive regions.
- **Limitations**: High construction overhead; poor performance gain for dense isosurfaces; parallel construction is complex and yields sublinear speedup [S2].

## Background
The method was proposed by Wilhelms and Van Gelder as a hierarchical spatial subdivision technique to accelerate cell extraction in isosurface computation [S1, unofficial]. It is a foundational approach in octree-based isosurfacing, enabling adaptive resolution by focusing computation only on regions where the isovalue exists.

## Sources
[S1] Optimal hybrid memory constrained isosurface extraction, http://nbn-resolving.de/urn:nbn:de:bsz:352-130091, fetched 2026-10-30 (unofficial)
[S2] Parallel Isosurface Extraction for 3D Data Analysis Workflows in Distributed Environments, https://doi.org/10.1002/cpe.1710, fetched 2026-10-26
