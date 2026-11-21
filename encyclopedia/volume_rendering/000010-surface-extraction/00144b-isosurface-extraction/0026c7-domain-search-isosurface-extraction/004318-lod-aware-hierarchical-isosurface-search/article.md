# LOD-Aware Hierarchical Isosurface Search

Also called: DYNAISOLOD, DYNAISOLOD Search Algorithm, Dynamic Isosurface Extraction and Level-of-Detail, Dynamic isosurface extraction search, Isosurface search in LODTree, LOD-aware isosurface search, Run-time Search and Rendering Algorithm

## Overview
LOD-Aware Hierarchical Isosurface Search (DYNAISOLOD) is a method for extracting isosurfaces from structured grid data by searching a multi-resolution voxel tree (LODTree) for cells containing the isovalue at a user-specified level-of-detail (LOD) [S1, S2, unofficial]. It solves the problem of interactive exploration of large simulation datasets where the isosurface threshold is unknown or dynamic, allowing scientists to select the detail of the resulting surface to maintain interactive frame rates [S1, unofficial]. Unlike [Octree-Based Isosurface Extraction](7786-octree-based-isosurface-extraction.md) which operates in geometry space assuming stable underlying geometry, DYNAISOLOD operates in voxel space, representing LOD as differently sized virtual voxels (e.g., 1x1x1, 2x2x2) [S1, unofficial]. This approach is necessary because dynamic isosurface thresholds imply no stable underlying geometry [S1, unofficial].

## How it works
**Inputs:** A pre-built LODTree file containing voxel data; an isosurface threshold (isovalue); and a desired level-of-detail (LOD) [S2, unofficial].
**Outputs:** A list of cells (virtual voxels) containing the isosurface at the specified LOD, which are then triangulated and rendered [S1, S2, unofficial].

**Data Structures:**
The core data structure is the LODTree, a 3D kd-tree (multidimensional search tree) created via a divide-and-conquer approach [S1, unofficial]. Each node in the tree represents a cell (virtual voxel) and stores three keys:
1.  Minimum data value in the cell.
2.  Maximum data value in the cell.
3.  Level-of-Detail (LOD) / Size of the cell (resolution) [S1, unofficial].

The tree is partitioned recursively: the first key partitions data into two halves (higher vs. lower than root); the second key partitions these halves; the third key partitions the resulting sub-branches; the process repeats for subtrees starting again with the first key [S1, unofficial].

**Algorithm Steps:**
1.  **Load LODTree:** Read the hierarchical voxel tree into memory [S2, unofficial].
2.  **Initialize Search:** Start at the root node. The search key cycles through the three dimensions: Minimum Value $\rightarrow$ Maximum Value $\rightarrow$ LOD $\rightarrow$ Minimum Value, etc. [S1, S2, unofficial].
3.  **Recursive Search:** Interrogate each node based on the current search key.
    *   **Minimum Key Search (`search-minimum`):**
        *   If `cell.min > isovalue`: The isosurface cannot intersect this cell or its left child (which contains smaller values). Recursively call `search-maximum` on the **left** branch [S1, S2, unofficial].
        *   Else (`cell.min <= isovalue`):
            *   If `cell.max > isovalue` AND `cell.LOD == desired_LOD`: Add cell to the result list [S1, S2, unofficial].
            *   Recursively call `search-maximum` on the **right** branch [S1, S2, unofficial].
    *   **Maximum Key Search (`search-maximum`):**
        *   If `cell.max < isovalue`: The isosurface cannot intersect this cell or its right child (which contains larger values). Recursively call `search-lod` on the **right** branch [S1, S2, unofficial].
        *   Else (`cell.max >= isovalue`):
            *   If `cell.min < isovalue` AND `cell.LOD == desired_LOD`: Add cell to the result list [S1, S2, unofficial].
            *   Recursively call `search-lod` on the **left** branch [S1, S2, unofficial].
    *   **LOD Key Search (`search-lod`):**
        *   If `cell.LOD > desired_LOD`: The cell is too coarse. Recursively call `search-minimum` on the **left** branch (smaller LODs) [S1, S2, unofficial].
        *   Else If `cell.LOD < desired_LOD`: The cell is too fine. Recursively call `search-minimum` on the **right** branch (larger LODs) [S1, S2, unofficial].
        *   Else (`cell.LOD == desired_LOD`):
            *   If `cell.min < isovalue` AND `cell.max > isovalue`: Add cell to the result list [S1, S2, unofficial].
            *   Recursively call `search-minimum` on the **right** branch, then on the **left** branch [S1, S2, unofficial].
4.  **Triangulation:** For each selected cell, determine which corners are above/below the isovalue to identify intersecting edges. Use the modified marching cubes triangulation table by Montani et al. to generate triangles [S1, S2, unofficial].
5.  **Vertex/Normal Caching:** Store calculated vertices in a hash table keyed to the edge they lie on to avoid redundant calculations. Compute normals via cross-product and store them in the same hash table for later averaging with normals of other triangles sharing the vertex [S2, unofficial].
6.  **Render:** Render the triangulated surface [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def search_minimum(node, isovalue, desired_lod):
    # Node has fields: min_val, max_val, lod, left, right
    if node.min_val > isovalue:
        # Isosurface does not intersect cell or left branch (smaller values)
        search_maximum(node.left, isovalue, desired_lod)
    else:
        if node.max_val > isovalue and node.lod == desired_lod:
            add_cell_to_list(node)
        search_maximum(node.right, isovalue, desired_lod)

def search_maximum(node, isovalue, desired_lod):
    if node.max_val < isovalue:
        # Isosurface does not intersect cell or right branch (larger values)
        search_lod(node.right, isovalue, desired_lod)
    else:
        if node.min_val < isovalue and node.lod == desired_lod:
            add_cell_to_list(node)
        search_lod(node.left, isovalue, desired_lod)

def search_lod(node, isovalue, desired_lod):
    if node.lod > desired_lod:
        search_minimum(node.left, isovalue, desired_lod)
    elif node.lod < desired_lod:
        search_minimum(node.right, isovalue, desired_lod)
    else:
        if node.min_val < isovalue and node.max_val > isovalue:
            add_cell_to_list(node)
        search_minimum(node.right, isovalue, desired_lod)
        search_minimum(node.left, isovalue, desired_lod)
```

**Parameters:**
*   **Isosurface Threshold (isovalue):** Defines the material fraction or property level (e.g., 0.55 for 55% filled) [S1, unofficial]. Can range from near zero to large powers of ten [S1, unofficial].
*   **Desired LOD:** The resolution of the virtual voxels. Finer LODs (e.g., 1x1x1) provide higher detail; coarser LODs (e.g., 2x2x2) provide lower detail [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Ambiguous Cases:** Standard marching cubes has ambiguous cases requiring neighbor data. DYNAISOLOD uses a straightforward approach by Montani et al. that allows unambiguous triangulation of an arbitrary voxel without neighbor data, at the price of occasional surface anomalies [S1, unofficial].
*   **Redundant Calculation:** Many triangles share vertices. A hash table keyed to the edge is used to cache vertex coordinates and normals [S2, unofficial].
*   **Optimization Limits:** LOD key testing can be optimized if the desired LOD is the greatest or least possible (stop testing LOD keys if searching left subtree of minimum LOD node) [S1, S2, unofficial].

## Subtopics

### Run-time Search and Geometry Creation
This variant refers to the specific run-time phase of DYNAISOLOD where the search is performed and geometry is created on the fly. It differs from the NOISE algorithm by adding a desired resolution/LOD parameter to the search, allowing multi-resolution isosurface extraction [S1, unofficial]. It uses the same recursive key-cycling logic (min, max, LOD) but includes optimizations such as skipping minimum key tests for left branches if the parent's minimum is already below the isovalue [S1, unofficial].

## Operating characteristics
**Time Complexity:** The search time complexity is $O(\log N)$ [S2, unofficial].
**Memory Complexity:** The overhead added to the base simulation volume data is approximately 1.77 times the size of the original simulation volume data set [S2, unofficial].
**Performance:**
*   **Small Data Set (27M cells):**
    *   LOD 1: 560,688 triangles, Search 0.35s, Geometry Creation 10.29s, Rendering 0.96s [S2, unofficial].
    *   LOD 4: 33,760 triangles, Search 0.02s, Geometry Creation 2.42s, Rendering 0.54s [S2, unofficial].
*   **Medium Data Set (54M cells):**
    *   LOD 1: 465,856 triangles, Search 0.25s, Geometry Creation 6.16s, Rendering 3.01s [S2, unofficial].
*   **Large Data Set (100M cells):**
    *   LOD 1: 276,180 triangles, Search 0.23s, Geometry Creation 1.44s, Rendering 3.68s [S2, unofficial].
*   **File Creation:** Parallel file creation into an LOD-tree takes 1 minute on a four-processor SGI Onyx2 for a 27 million cell dataset [S2, unofficial].

**Strengths:**
*   Enables interactive exploration of large datasets by allowing dynamic isovalue and LOD selection [S1, unofficial].
*   Preserves spatial coherence of the resulting surface between virtual voxel resolutions due to the interpolative nature of marching cubes [S1, unofficial].
*   Scales to interactive speeds at lower levels of detail despite increasing problem sizes [S2, unofficial].

**Limitations:**
*   Occasional surface anomalies due to the unambiguous triangulation method that ignores neighbors [S1, unofficial].
*   Static geometry creation yields higher quality triangulation; DYNAISOLOD is best used in conjunction with static methods for stable data [S2, unofficial].
*   Performance is dependent on CPU, networking, and disk speeds during file creation [S2, unofficial].

## Background
DYNAISOLOD was developed as part of Sandia National Laboratories' EIGEN/VR project for analyzing structured and unstructured grid simulation data sets produced by massively parallel supercomputers [S1, unofficial]. The first application was a proof of concept using structured rectilinear grid data sets from the CTH simulation code [S1, unofficial]. It is a powerfid extension of the NOISE (Near Optimal IsoSurface Extraction) algorithm proposed by Y. Livnat et al. [S1, unofficial]. The triangulation table used is from C. Montani et al. [S1, unofficial]. It is used in Sandia's EIGENNR visualization tool [S2, unofficial].

## Sources
[S1] Dynamic Isosurface Extraction and Level-of-Detail in Voxel Space, https://digital.library.unt.edu/ark:/67531/metadc694729/?q=marching cubes, fetched 2026-10-28 (unofficial)
[S2] Dynamic isosurface extraction and level-of-detail in voxel space, https://digital.library.unt.edu/ark:/67531/metadc707496/?q=marching cubes, fetched 2026-10-28 (unofficial)
