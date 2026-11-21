# NOISE (Near Optimal IsoSurface Extraction)

Also called: Livat, Shen, and Johnson method, Near Optimal IsoSurface Extraction

## Overview
NOISE is a rapid isosurface extraction algorithm for structured and unstructured grids that enables interactive, real-time changes to the isosurface threshold without full recalculation [S2, unofficial]. It solves the performance bottleneck of inspecting every cell in a volume by using a K-D tree embedded into span-space to quickly identify candidate cells containing an iso-surface, avoiding inspection of every cell [S1, unofficial]. It sets itself apart from sibling methods like [Standard Interval Tree for Isosurface Extraction](6739-standard-interval-tree-for-isosurface-extraction.md) by utilizing a multi-dimensional search tree (kd-tree) rather than a 1D interval tree, allowing for efficient spatial partitioning of the min/max value pairs [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset (structured or unstructured grid) and a user-specified isosurface threshold $v_{iso}$.
*   **Output:** A list of triangles representing the isosurface, generated via triangulation of the identified candidate voxels.

**Data Structures:**
The core data structure is a K-D tree built on "span-space." Each grid cell (voxel) is represented as a point in this 2D span-space.
*   **Point Representation:** Each cell $c$ is characterized by its lowest voxel value $v_{min}$ and its highest voxel value $v_{max}$ [S1, unofficial].
*   **Node Layout:** Each node in the K-D tree stores two scalar values per voxel: the minimum and maximum scalar values found within that voxel [S2, unofficial].

**Algorithm Steps:**
1.  **Span-Space Construction:** Represent each grid cell as a point in span-space characterized by its lowest ($v_{min}$) and highest ($v_{max}$) voxel value [S1, unofficial].
2.  **Tree Construction:** Construct a multi-dimensional search tree (kd-tree) storing the minimum and maximum scalar values for each voxel in the volumetric dataset [S2, unofficial].
3.  **Query:** Search the resulting "span space" of the kd-tree for the user-specified isosurface threshold $v_{iso}$ [S2, unofficial].
4.  **Candidate Identification:** Identify candidate cells where $v_{min} \le v_{iso}$ and $v_{max} \ge v_{iso}$ using the K-D tree [S1, unofficial]. The output is a set of only those voxels whose stored min/max ranges bracket the threshold, indicating potential intersection [S2, unofficial].
5.  **Triangulation:** Triangulate the returned voxels using the marching cubes algorithm [S2, unofficial].
6.  **Rendering:** Render the generated triangles [S2, unofficial].

**Parameters:**
*   Typical values: Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def noise_extract(volume, isovalue):
    """
    Extract isosurface using NOISE algorithm.
    volume: list of dicts, each with 'values' (list of scalars) and 'id'.
    isovalue: The threshold for extraction.
    Returns: List of triangles.
    """
    # Step 1 & 2: Build Span-Space K-D Tree
    span_points = []
    for cell in volume:
        v_min = min(cell['values'])
        v_max = max(cell['values'])
        span_points.append({'min': v_min, 'max': v_max, 'id': cell['id']})
    kd_tree = build_kd_tree(span_points, depth=0)

    # Step 3 & 4: Query for candidates
    candidate_cells = query_kd_tree(kd_tree, isovalue, volume)

    # Step 5: Triangulate
    triangles = []
    for cell in candidate_cells:
        # Standard Marching Cubes lookup table [G]
        cell_triangles = marching_cubes_lookup(cell, isovalue)
        triangles.extend(cell_triangles)

    # Step 6: Render
    return triangles

def build_kd_tree(points, depth):
    if not points:
        return None
    axis = depth % 2
    points.sort(key=lambda p: p['min' if axis == 0 else 'max'])
    mid = len(points) // 2
    node = points[mid]
    node['left'] = build_kd_tree(points[:mid], depth + 1)
    node['right'] = build_kd_tree(points[mid+1:], depth + 1)
    return node

def query_kd_tree(node, isovalue, volume):
    if node is None:
        return []
    results = []
    # Check span-space condition: min <= isovalue <= max
    if node['min'] <= isovalue <= node['max']:
        results.append(volume[node['id']])
    # Prune subtrees where min > isovalue (cannot satisfy min <= isovalue)
    if node['left'] is not None and node['left']['min'] <= isovalue:
        results.extend(query_kd_tree(node['left'], isovalue, volume))
    if node['right'] is not None and node['right']['min'] <= isovalue:
        results.extend(query_kd_tree(node['right'], isovalue, volume))
    return results

def marching_cubes_lookup(cell, isovalue):
    # Standard 256-entry lookup table [G]
    # Computes cell vertex signs relative to isovalue and returns triangles
    return []
```

**Edge Cases and Pitfalls:**
*   **Marching Cubes Ambiguity:** Standard marching cubes ambiguous cases require neighbor data or complex logic, which slows real-time frame rates or bloats file size [S2, unofficial]. The basic NOISE implementation does not include ambiguity resolution logic to preserve interactive performance [S2, unofficial].
*   **Occlusion:** Triangles are often generated that are later occluded during rendering. It is advisable to visit cells in front-to-back order and only extract and render triangles that fall outside previously occluded areas [S1, unofficial].

## Operating characteristics
*   **Performance:** Enables interactive, real-time changes to the isosurface threshold [S2, unofficial]. The K-D tree allows for rapid identification of candidate cells, avoiding the $O(N)$ inspection of all cells [S1, unofficial].
*   **Limitations:** The basic implementation does not include ambiguity resolution logic to preserve interactive performance, which can lead to topological errors in the extracted surface [S2, unofficial].
*   **Use Case:** Best suited for visualization environments requiring dynamic threshold adjustment [S2, unofficial].

## Background
NOISE was proposed by Y. Livnat et al. [S2, unofficial]. It was implemented at Sandia National Laboratories as a baseline for DYNAISOLOD [S2, unofficial]. The method was later improved by other researchers who used an interval tree instead of a K-D tree [S1, unofficial].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] Dynamic isosurface extraction and level-of-detail in voxel space, https://digital.library.unt.edu/ark:/67531/metadc707496/?q=marching cubes, fetched 2026-10-28 (unofficial)
[S3] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
