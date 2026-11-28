# Stack-Based Representation (SBR)
Also called: Stacks
## Overview
Stack-Based Representation (SBR) is a sparse volume data structure that decomposes a voxel grid into vertical columns, or "stacks," where each column is compressed via run-length encoding of attribute values along the vertical axis [S1, unofficial]. It solves the memory inefficiency of dense voxel grids for layered or stratified datasets by exploiting vertical coherence, storing only the boundaries between attribute changes [S1, unofficial]. Unlike general sparse structures like Hash Grids or SparseGrids that treat all spatial dimensions symmetrically, SBR assumes a strong directional anisotropy (typically vertical), making it distinct from siblings that rely on isotropic subdivision [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Input:** A voxel grid $V$ of resolution $w \times h \times d$, where each voxel $v_{x,y,z}$ stores one or more attributes (e.g., color, material, density) [S1, unofficial].
*   **Output:** A set $S$ of vertical stacks, where each stack $S_{x,y}$ corresponds to a column at $(x,y)$ and is represented as a sequence of intervals [S1, unofficial].

**Data Structures:**
1.  **Stack ($S_{x,y}$):** A list of intervals representing the column $V_{x,y}$ [S1, unofficial].
2.  **Interval ($i_k$):** A tuple $\langle a_k, h_k \rangle$ where:
    *   $a_k$: The attribute value shared by all voxels in the interval [S1, unofficial].
    *   $h_k$: The height (z-index) of the last voxel in the interval [S1, unofficial].
    *   The interval covers voxels from $z = 1 + h_{k-1}$ to $z = h_k$ [S1, unofficial].
    *   For the first interval ($k=1$), $h_0$ is assumed to be 0 [S1, unofficial].

**Algorithm Steps:**
1.  Iterate through every column $(x,y)$ in the grid $w \times h$ [S1, unofficial].
2.  For each column, iterate through the voxels $v_{x,y,z}$ from $z=1$ to $d$ [G].
3.  Identify contiguous sequences of voxels with identical attribute values [S1, unofficial].
4.  For each sequence, create an interval $i_k = \langle a_k, h_k \rangle$, where $a_k$ is the attribute value and $h_k$ is the z-index of the last voxel in the sequence [S1, unofficial].
5.  Append $i_k$ to the stack $S_{x,y}$ [G].
6.  Intervals are naturally sorted by height in ascending order ($h_k < h_{k+1}$) due to the sequential construction [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_sbr(voxel_grid, width, height, depth):
    """
    Builds the Stack-Based Representation from a dense voxel grid.
    voxel_grid: 3D array [width][height][depth] containing attribute values (0-based indexing).
    Returns: A 2D array of lists, stacks[width][height], where each list contains intervals.
    Note: The mathematical definition uses 1-based height indexing, so code height = index + 1.
    """
    stacks = [[[] for _ in range(height)] for _ in range(width)]

    for x in range(width):
        for y in range(height):
            current_stack = []
            current_attr = None
            current_height = 0
            
            for z_idx in range(depth):
                attr = voxel_grid[x][y][z_idx]
                height = z_idx + 1
                
                if attr != current_attr:
                    if current_attr is not None:
                        current_stack.append((current_attr, current_height))
                    current_attr = attr
                    current_height = height
                else:
                    current_height = height
            
            if current_attr is not None:
                current_stack.append((current_attr, current_height))
            
            stacks[x][y] = current_stack
            
    return stacks
```

**Parameters:**
*   **Attribute Type:** The specific data stored in $a_k$ (e.g., uint8 for material ID, float for density) [S1, unofficial].
*   **Sorting:** Intervals must be sorted by $h_k$ ascending. This is guaranteed by the sequential construction order [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Columns:** If a column contains no voxels (or all voxels are "empty" with a null attribute), the stack may be empty or contain a single interval with a null attribute [G].
*   **Precision:** Height values $h_k$ must be stored with sufficient bits to represent the maximum depth $d$ [G].
*   **Parallelism:** Construction is embarrassingly parallel as stack construction does not require information about neighboring stacks [S1, unofficial].

## Subtopics

### [QuadStack](1113-quadstack.md)
Extends SBR by organizing stacks into a quadtree to compress regions with identical interval sequences, achieving higher compression for datasets with spatial coherence in layer topology. It sets itself apart by handling horizontal redundancy that basic SBR misses, and should be chosen when vertical topology repeats across the xy-plane.

### [Group of Stacks (gstack)](1169-group-of-stacks-gstack.md)
Groups spatially adjacent stacks with identical attribute sequences into a single structure, separating attribute compression from geometric heightfield storage. It differs by compressing attributes independently of geometry, and is chosen when attribute values are constant across rectangular regions but heights vary.

### [Stack-Based Terrain Representation (SBRT)](15717-stack-based-terrain-representation-sbrt.md)
Generalizes SBR for terrain data, using 1D-like textures for better prefetching and caching performance compared to single-texture approaches. It sets itself apart by favoring hardware prefetching via 1D structures, and is chosen for terrain rendering where cache locality is critical.

## Operating characteristics
*   **Time Complexity:** Construction is $O(n)$, where $n = w \times h \times d$ is the total number of voxels, because each voxel is processed exactly once [S1, unofficial].
*   **Memory Complexity:** Depends on the number of layers/intervals [S1, unofficial]. In the worst case (every voxel has a different attribute), memory usage is proportional to the number of voxels [G]. In the best case (single attribute per column), memory is proportional to $w \times h$ [G].
*   **Strengths:** Embarrassingly parallel construction; efficient for layered/stratified data; simple data structure [S1, unofficial].
*   **Limitations:** Assumes vertical anisotropy; does not inherently compress horizontal redundancy (addressed by children like QuadStack) [S1, unofficial].
*   **When to Use:** Layered volumetric datasets (e.g., geological strata, terrain with subsurface layers) where vertical coherence is high [S1, unofficial].
*   **When Not to Use:** Isotropic volumes or data with significant horizontal coherence but vertical variation [G].

## Background
The Stack-Based Representation was introduced as a method for efficiently storing and rendering layered volumetric datasets [S1, unofficial]. The method addresses the limitations of traditional voxel grids for data that can be naturally decomposed into vertical columns with constant attributes [S1, unofficial].

## Sources
[S1] QuadStack: An Efficient Representation and Direct Rendering of Layered Datasets | DCGI, https://dcgi.fel.cvut.cz/publications/2021/graciano-tvcg-quadstack/, fetched 2026-10-29 (unofficial)
