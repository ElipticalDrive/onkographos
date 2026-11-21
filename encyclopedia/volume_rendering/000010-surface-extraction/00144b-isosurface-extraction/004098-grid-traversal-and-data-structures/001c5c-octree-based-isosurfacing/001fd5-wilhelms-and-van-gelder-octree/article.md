# Wilhelms and Van Gelder Octree
Also called: Wilhelms and Van Gelder method, Octree-based cell extraction acceleration

## Overview
The Wilhelms and Van Gelder Octree is a hierarchical spatial subdivision method that accelerates isosurface extraction by pruning branches of an octree that cannot contain the isosurface [S1, unofficial]. It solves the problem of excessive computational cost in exhaustive cell enumeration by using value range intervals to skip inactive regions [S1, unofficial]. It differs from its sibling [Standard Interval Tree for Isosurface Extraction](1230-standard-interval-tree-for-isosurface-extraction.md) by using a strict octree hierarchy rather than a general interval tree, and from [IsoOctree](1744-isooctree.md) by focusing on static value-range pruning rather than view-dependent or specific topological constraints [S1, unofficial][S2].

## How it works
**Inputs:** A volumetric dataset $V$ of scalar values; an isovalue $t \in \mathbb{R}$.
**Outputs:** A set of cells (or vertices/edges) intersecting the isosurface defined by $t$.

**Data Structures:**
The method constructs an octree where each node $n$ corresponds to a rectilinear block $B$ of cells $C_{i,j,k}$ in the volume data [S1, unofficial].
Each node $n$ is attributed an interval $[n_{min}, n_{max})$ [S1, unofficial].
This interval is defined as the union of the intervals $[v_{min}, v_{max})$ of all cells $C_{i,j,k}$ belonging to the block $B$ [S1, unofficial].
Mathematically, for a node $n$ covering a set of cells $S_n$:
$$ n_{min} = \min_{C \in S_n} (\min_{(x,y,z) \in C} \hat{f}(x,y,z)) $$
$$ n_{max} = \max_{C \in S_n} (\max_{(x,y,z) \in C} \hat{f}(x,y,z)) $$
where $\hat{f}$ is the interpolation function and $[v_{min}, v_{max})$ denotes the range of values the interpolation function attains in the portion of the volume corresponding to the cell [S1, unofficial].
In practice, for trilinear interpolation within a cell, the min and max are the minimum and maximum of the 8 corner values of that cell [S1, unofficial]. Thus, $n_{min}$ is the minimum of all corner values in the block, and $n_{max}$ is the maximum of all corner values in the block [S1, unofficial].

**Algorithm Steps:**
1.  **Construction:** Build the octree from the root down to the leaf level corresponding to the voxel resolution. For each node, compute $n_{min}$ and $n_{max}$ by aggregating the min/max values of its children (or directly from voxel values at leaf level) [S1, unofficial].
2.  **Traversal:** Perform a recursive tree traversal starting from the root node [S1, unofficial].
3.  **Pruning:** For a current node $n$, check if the isovalue $t$ lies within the node's interval $[n_{min}, n_{max})$ [S1, unofficial].
    *   If $t \notin [n_{min}, n_{max})$, ignore (prune) this entire branch [S1, unofficial].
    *   If $t \in [n_{min}, n_{max})$, continue traversal to children if $n$ is not a leaf [S1, unofficial].
4.  **Extraction:** For leaf nodes $n$ where $t \in [n_{min}, n_{max})$, investigate the cells in the leaf block cell-by-cell using the standard enumeration algorithm (e.g., Marching Cubes) to extract the isosurface geometry [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wilhelms_van_gelder_extract(volume, isovalue, octree_root):
    """
    volume: 3D array of scalar values.
    isovalue: float t.
    octree_root: Root node of the pre-constructed octree.
    Returns: List of extracted cells/vertices.
    """
    extracted_cells = []
    
    # Recursive helper function
    def traverse(node):
        # Check pruning condition
        if not (node.min_val <= isovalue < node.max_val):
            return
        
        # If leaf node, process cells
        if node.is_leaf:
            for cell in node.cells:
                # Standard enumeration algorithm (e.g., Marching Cubes)
                # Check if isosurface intersects this specific cell
                if isovalue_in_cell_range(cell.min_val, cell.max_val, isovalue):
                    cell.extract(isovalue, extracted_cells)
        else:
            # Recurse to children
            for child in node.children:
                traverse(child)

    traverse(octree_root)
    return extracted_cells

def isovalue_in_cell_range(cell_min, cell_max, t):
    """
    Helper to check if t is within the cell's value range.
    For trilinear interpolation, cell_min/max are min/max of 8 corners.
    """
    return cell_min <= t < cell_max
```

**Parameters:**
*   **Octree Depth:** Determined by the volume resolution. Not given in the sources.
*   **Interpolation:** Trilinear is implied by the definition of cell intervals based on corner values [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The interval check $t \in [n_{min}, n_{max})$ relies on floating-point comparisons. Degenerate cases where $t$ is exactly equal to $n_{min}$ or $n_{max}$ must be handled consistently with the enumeration algorithm to avoid cracks or missing geometry [S1, unofficial].
*   **Empty Branches:** If a branch is entirely empty (no data), it should be pruned immediately [S1, unofficial].
*   **Construction Overhead:** The method requires building the octree, which adds CPU storage space and construction time [S1, unofficial].

## Subtopics

### Octree Search Structure
The Octree Search Structure is a hierarchical spatial index used to skip analysis of inactive cells during isosurface extraction, reducing computational load at the cost of construction time and memory [S2]. It differs from the standard Wilhelms and Van Gelder method by being explicitly framed as a reusable search structure that can be applied to multiple extractions or different computational resources [S2]. When extracting multiple surfaces, the same index can be reused [S2]. However, construction overhead makes it slower than exhaustive WWMC for single or dual surface extraction [S2]. Parallel construction requires a global coordinator, likely yielding sublinear speedup [S2]. If processing on different resources, downloading the octree index may require transferring significant data (e.g., 219 MB for medium datasets) [S2].

## Operating characteristics
*   **Time Complexity:** The cell extraction phase has linear time complexity $O(n)$ with respect to the number of cells in the worst case (when all cells are active) [S1, unofficial]. However, for sparse isosurfaces, the octree traversal provides considerable acceleration by skipping inactive branches [S1, unofficial].
*   **Memory Complexity:** Requires moderate additional CPU storage space for the octree structure [S1, unofficial]. The size of the octree index can be significant (e.g., 219 MB for medium datasets) [S2].
*   **Measured Performance:** For datasets where the isosurface is sparse (e.g., skipping up to 99% of cells), the method can extract the isosurface in about one third of the time compared to exhaustive WWMC [S2]. However, when considering the total time including construction, it is often slower than WWMC for single or dual surface extraction [S2]. For dense isosurfaces (e.g., more than half of cells active), the performance gain is negligible [S2].
*   **Strengths:** Significant acceleration for sparse isosurfaces; reduces computational load by skipping inactive cells [S1, unofficial][S2].
*   **Limitations:** Construction overhead; not beneficial for single/dual surface extraction if construction time is included; parallel construction suffers from coordinator bottlenecks [S2].
*   **When to Use:** Large volumetric datasets with sparse isosurfaces where multiple extractions are performed using the same index [S2].
*   **When Not to Use:** Single surface extraction where construction time dominates; dense isosurfaces where pruning is ineffective [S2].

## Background
The Wilhelms and Van Gelder method was proposed by Wilhelms and Van Gelder as an octree subdivision technique for accelerating cell extraction in isosurface computation [S1, unofficial]. It is based on the idea that hierarchical spatial subdivision can achieve considerable acceleration at the cost of moderate additional CPU storage space [S1, unofficial]. The method is cited in the context of optimal hybrid memory constrained isosurface extraction [S1, unofficial].

## Sources
[S1] Optimal hybrid memory constrained isosurface extraction, http://nbn-resolving.de/urn:nbn:de:bsz:352-130091, fetched 2026-10-30 (unofficial)
[S2] Parallel Isosurface Extraction for 3D Data Analysis Workflows in Distributed Environments, https://doi.org/10.1002/cpe.1710, fetched 2026-10-26
