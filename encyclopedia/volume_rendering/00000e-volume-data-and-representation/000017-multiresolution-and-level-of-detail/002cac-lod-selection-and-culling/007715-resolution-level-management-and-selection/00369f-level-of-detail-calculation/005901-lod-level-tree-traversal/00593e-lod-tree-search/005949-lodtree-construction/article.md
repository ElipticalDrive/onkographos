# LODTree Construction

Also called: 3D LOD Search Tree, DYNAISOLOD / LODTree Construction, LODTree file creation algorithm

## Overview
LODTree Construction is a parallel divide-and-conquer algorithm that builds a multidimensional (multikey) search tree for voxel data, enabling fast retrieval of cells intersecting an isosurface at a specific level of detail (LOD) [S1, unofficial]. It partitions data using three keys—minimum value, maximum value, and LOD—to create a 3D binary tree structure that supports efficient run-time search [S1, unofficial]. Unlike static hierarchical representations, this method optimizes for parallel file creation and compact storage via bit-mapped vertex coordinates, specifically targeting structured (rectilinear) grids [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A structured (rectilinear) voxel grid; MPI processor group; parameters for LOD levels, voxel size increment, and overlap factor [S1, unofficial].
*   **Output:** A binary file containing the LODTree structure, including a header, an array of base indices, and compressed vertex coordinate data [S1, unofficial].

**Data Structures**
The file layout consists of a header and two primary arrays [S1, unofficial]:

1.  `lod_file_header_struct`:
    *   `char signature[SIZE]`: Version signature.
    *   `char source[SIZE]`: Source identifier.
    *   `int width, height, depth`: Dimensions of the data set.
    *   `int num_data_points`: Total number of data points.
    *   `float min, max`: Global minimum and maximum values for the tree.
    *   `int size`: Number of LOD nodes.
    *   `int resolutions`: Number of resolutions.
    *   *Note:* All numeric fields are signed; maximum array index is limited to a 32-bit signed integer for Java compatibility [S1, unofficial].

2.  `lod_tree_struct` (in-memory representation written to file):
    *   `lod_file_header_t header`: The header defined above.
    *   `int* array_base_indices`: Array of indices into the original simulation data for the base cell vertex.
    *   `char* min_max_vertex_coords`: Compressed bit-mapped representation.
        *   Format: `Xlllxlll` (8 bits per char).
        *   First 4 bits (`lll`): Canonical vertex coordinate of the cell vertex holding the **minimum** data value.
        *   Last 4 bits (`lll`): Canonical vertex coordinate of the cell vertex holding the **maximum** data value.
        *   `X`: Unused bit.
    *   `char* cell_dimensions`:
        *   First 3 bits: Flags indicating if the cell is irregular in width, height, or depth.
        *   Last 4 bytes: Regular size of the cell.
        *   *Constraint:* Largest possible virtual voxel is $16 \times 16 \times 16$ [S1, unofficial].

**Algorithm Steps**
The construction uses a Quicksort-like recursive partitioning scheme parallelized via MPI [S1, unofficial].

1.  **Initialization:** The root processor initializes MPI, reads the voxel file, and calculates the minimum and maximum data values for all voxel cells [S1, unofficial].
2.  **Recursive Partitioning (Quicksort-like):**
    *   The algorithm partitions the current set of node indices into two halves based on the current key (cycling through: Minimum Value, Maximum Value, LOD) [S1, unofficial].
    *   A median node is found. The top half of the node array is scheduled to be sent to another processor in the group [S1, unofficial].
    *   The root processor proceeds to perform the same partitioning algorithm recursively on the bottom part locally [S1, unofficial].
    *   This process repeats, cycling keys: Min $\rightarrow$ Max $\rightarrow$ LOD $\rightarrow$ Min $\rightarrow$ Max $\rightarrow$ LOD, creating a 3D binary tree where each branch represents a new dimension [S1, unofficial].
3.  **Parallelization Strategy:**
    *   At each recursive call, the top half of the node array is partitioned to another processor until the number of processors in the array is reached [S1, unofficial].
    *   Subsequent partitioning is done locally on each processor [S1, unofficial].
    *   A heavily recursive subroutine deterministically predicts message sizes for each processor upon recursive split. This allows processors to prepare for work and the root processor to determine the correct receive order [S1, unofficial].
    *   *Note:* The algorithm is most efficient when the number of processors is a power of two, but works correctly for arbitrary numbers [S1, unofficial].
4.  **Bit-Mapping and Compression:**
    *   Canonical vertex coordinates for min/max values are stored in 4-bit fields within a `char` to reduce search tree size [S1, unofficial].
    *   This compressed representation works only for structured (rectilinear) grids [S1, unofficial].
5.  **Parameter Handling:**
    *   **Virtual Voxel Size Increment:** Defines the size increment per dimension per LOD level (default: 1) [S1, unofficial].
    *   **Overlap Factor:** Specifies by how much an edge voxel can expand to prevent rows of small voxels at grid edges [S1, unofficial].

**Pseudocode**
The following pseudocode illustrates the recursive partitioning logic. Note that the provided search pseudocode in the sources is for *run-time search*, not construction. The construction logic is described textually as a Quicksort-like partition. The search logic is included below as it is integral to the tree's utility and defined in the source [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Run-time Search Pseudocode (from sources)
# Assumes tree nodes have: min_val, max_val, lod, left_child, right_child

def search_minimum(branch, isovalue, desired_lod):
    """
    Search branch based on minimum value key.
    """
    if branch.min_val > isovalue:
        # Isosurface does not intersect cell based on min value
        # Prune left branch (all cells in left branch have smaller keys)
        # Actually, source says: search_maximum(left branch)
        # Wait, source logic: if min > isovalue, isosurface doesn't intersect.
        # But the source pseudocode says: search_maximum( left branch )
        # Let's follow source exactly.
        search_maximum(branch.left_child, isovalue, desired_lod)
    else:
        # Min value <= isovalue. Isosurface might intersect.
        if branch.max_val > isovalue and branch.lod == desired_lod:
            add_cell(branch)
        # Source says: search_maximum( right branch )
        search_maximum(branch.right_child, isovalue, desired_lod)

def search_maximum(branch, isovalue, desired_lod):
    """
    Search branch based on maximum value key.
    """
    if branch.max_val < isovalue:
        # Isosurface does not intersect cell based on max value
        search_lod(branch.right_child, isovalue, desired_lod)
    else:
        # Max value >= isovalue.
        if branch.min_val < isovalue and branch.lod != desired_lod:
            add_cell(branch)
        search_lod(branch.left_child, isovalue, desired_lod)

def search_lod(branch, isovalue, desired_lod):
    """
    Search branch based on LOD key.
    """
    if branch.lod > desired_lod:
        search_minimum(branch.left_child, isovalue, desired_lod)
    elif branch.lod < desired_lod:
        search_minimum(branch.right_child, isovalue, desired_lod)
    else:
        # LOD == desired_lod
        if branch.min_val < isovalue and branch.max_val > isovalue:
            add_cell(branch)
        search_minimum(branch.right_child, isovalue, desired_lod)
        search_minimum(branch.left_child, isovalue, desired_lod)
```

**Edge Cases and Pitfalls**
*   **Grid Structure:** The compressed bit-mapped representation works only for structured (rectilinear) grids [S1, unofficial].
*   **Processor Count:** Efficiency drops if the number of processors is not a power of two, though correctness is maintained [S1, unofficial].
*   **Java Compatibility:** Array indices are limited to 32-bit signed integers [S1, unofficial].
*   **Irregular Voxels:** The `cell_dimensions` bitmap handles irregularly shaped virtual voxels at grid edges, with a maximum virtual voxel size of $16 \times 16 \times 16$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The construction is $O(N \log N)$ per processor due to the Quicksort-like partitioning, where $N$ is the number of data points [S1, unofficial]. The parallelization reduces wall-clock time by distributing the partitioning work [S1, unofficial].
*   **Memory Complexity:** The bit-mapped representation significantly reduces the size of the search tree compared to storing full coordinates [S1, unofficial]. The use of a recursive subroutine to predict message sizes allows for efficient memory preparation [S1, unofficial].
*   **Performance:** The algorithm is optimized for parallel file creation. Run-time search is described as "very fast" due to the 3D binary tree structure allowing pruning based on any combination of the three keys [S1, unofficial].
*   **Strengths:** Fast search, compact storage, parallel construction, handles irregular edge voxels.
*   **Limitations:** Restricted to structured (rectilinear) grids; Java compatibility constraints limit array size to 32-bit signed integers [S1, unofficial].

## Background
LODTree Construction was developed as part of the DYNAISOLOD project for dynamic isosurface extraction and level-of-detail in voxel space [S1, unofficial]. It extends the NOISE algorithm with parallelization (MPI) and bit-mapped compression [S1, unofficial]. The method is designed to support efficient run-time geometry creation for large-scale scientific visualization [S1, unofficial].

## Sources
[S1] Dynamic Isosurface Extraction and Level-of-Detail in Voxel Space, https://digital.library.unt.edu/ark:/67531/metadc694729/?q=marching cubes, fetched 2026-10-28 (unofficial)
