# BSAMR Data Representation and SFC Indexing

Also called: Brick for Brick Addressing Table (BFAT), Nested Hierarchy of Straight Grid

## Overview
BSAMR (Block-Structured AMR) is a hierarchical data representation for time-series volume data where each time step consists of a sequence of nested Level of Detail (LOD) grids [S1, unofficial]. It solves the problem of efficiently indexing and storing adaptive mesh refinement data by mapping 3D spatial positions of voxel groups (bricks) to unique 1D indices using Space-Filling Curves (SFC) [S1, unofficial]. Unlike unstructured AMR methods that rely on complex tree traversals or tetrahedral connectivity, BSAMR maintains a strict nested hierarchy where coarser LODs fully cover the domain, ensuring no empty space and enabling cache-friendly, out-of-core access patterns [S1, unofficial].

## How it works
**Inputs:** A sequence of regular Cartesian grids representing a time series.
**Outputs:** A BSAMR structure comprising nested LODs per time step, SFC indices for each brick, and a Brick for Brick Addressing Table (BFAT).

**Data Structures:**
1.  **Brick:** A group of voxels treated as a single loadable unit.
2.  **LOD (Level of Detail):** A grid resolution level. LOD0 is the coarsest, covering the entire domain. LOD1 is nested within LOD0, and so on.
3.  **BFAT (Brick for Brick Addressing Table):** A 2D sparse matrix.
    *   **Y-axis:** Temporal dimension (time steps).
    *   **X-axis:** LOD information, ordered from coarsest (left) to finest (right).
    *   **Cells:** Contain SFC-sorted brick indices. The matrix is sparse because AMR hierarchy varies; early time steps may have fewer refined LODs or different cardinalities.

**Algorithm Steps:**

1.  **Pre-processing (Conversion):** Convert the input sequence of regular Cartesian grids into a sequence of BSAMR structures. Ensure that each time step is refined into the same number of LODs and that the nested order is respected (LOD $k$ is nested inside LOD $k-1$) [S1, unofficial].
2.  **SFC Indexing:** Apply a Space-Filling Curve (SFC) method, specifically the Hilbert curve, to index each AMR brick uniquely. Different SFC orders may be applied per LOD to optimize locality [S1, unofficial].
3.  **Index Calculation:** Compute the unique 1D index $\beta_{i, \delta}$ for each brick, where $i$ is the LOD index and $\delta$ is the SFC index mapping the 3D position to 1D [S1, unofficial].
4.  **Storage:** Store individual scalar values of each brick in individual binary files. The BFAT is constructed to map time steps and LODs to these indices [S1, unofficial].

**Formulas:**
The unique index for a brick is denoted as $\beta_{i, \delta}$ [S1, unofficial].
*   $i$: The Level of Detail (LOD) index.
*   $\delta$: The SFC index, representing the mapping of the 3D position of the brick into a 1D sequence.
*   $|\Gamma|$: The cardinality of the set of bricks at a specific LOD and time step. This varies per LOD and per time step. For LOD0, $|\Gamma| = 1$ because it consists of a single brick covering the entire domain [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_bsamr(time_series_grids):
    """
    Converts a sequence of regular Cartesian grids into a BSAMR structure.
    time_series_grids: List of grids, where each grid is a sequence of LODs.
    Returns: BFAT (2D sparse matrix) and brick storage files.
    """
    num_time_steps = len(time_series_grids)
    # Determine max LODs across all time steps to define BFAT width
    max_lods = max(len(grids) for grids in time_series_grids)
    
    # Initialize BFAT as a 2D sparse matrix (list of lists or dictionary)
    bfat = [[None for _ in range(max_lods)] for _ in range(num_time_steps)]
    
    for t, lod_grids in enumerate(time_series_grids):
        for i, lod_grid in enumerate(lod_grids):
            # lod_grid contains bricks at LOD i for time step t
            bricks = lod_grid.bricks
            
            # Apply SFC (Hilbert curve) to index bricks
            # SFC order may vary per LOD i
            sfc_indices = [hilbert_index(brick.position) for brick in bricks]
            
            # Sort bricks by SFC index to ensure spatial locality
            sorted_bricks = sorted(zip(sfc_indices, bricks), key=lambda x: x[0])
            
            # Store the SFC indices in the BFAT
            # The cell contains the list of SFC indices for this LOD at this time step
            bfat[t][i] = [idx for idx, _ in sorted_bricks]
            
            # Store individual scalar values of each brick in binary files
            for idx, brick in sorted_bricks:
                save_brick_to_binary_file(brick, idx)
                
    return bfat

def hilbert_index(position):
    """
    Maps a 3D position to a 1D Hilbert curve index.
    position: (x, y, z) coordinates of the brick.
    Returns: 1D integer index.
    """
    # Implementation of Hilbert curve mapping
    # Not given in the sources. [G]
    pass
```

**Parameters:**
*   **SFC Order:** The order of the Hilbert curve can be adjusted per LOD. Not given in the sources. [S1, unofficial]
*   **Brick Size:** The number of voxels per brick. Not given in the sources. [S1, unofficial]

**Edge Cases and Pitfalls:**
*   **Hierarchy Variation:** The AMR hierarchy may vary across time steps, leading to a sparse BFAT where some LODs may not exist or have different cardinalities in early time steps. The implementation must handle missing LODs gracefully. [S1, unofficial]
*   **Nested Order:** The nested order must be strictly respected. LOD0 always covers the entirety of the domain, ensuring no empty space. Violating this leads to incorrect indexing and rendering artifacts. [S1, unofficial]
*   **Precision:** SFC indexing requires precise mapping of 3D positions to 1D indices. Floating-point inaccuracies in brick positions can lead to incorrect sorting or indexing. [G]

## Subtopics

### TET-AMR Data Structure
Uses tetrahedral elements instead of voxel bricks. Chosen when unstructured mesh topology is required. [1094-tet-amr-data-structure.md]

### Exact AMR Rendering
Focuses on precise ray-element intersection for AMR data. Chosen when geometric accuracy is critical over performance. [4399-exact-amr-rendering.md]

### AMR Basis Reconstruction Filter
Applies filters to reconstruct scalar values from AMR data. Chosen when smoothing or anti-aliasing is needed. [5581-amr-basis-reconstruction-filter.md]

## Operating characteristics
*   **Time Complexity:** Pre-processing involves SFC indexing, which is $O(N \log N)$ for $N$ bricks due to sorting. Storage and retrieval are $O(1)$ with BFAT lookup. Not given in the sources. [S1, unofficial]
*   **Memory Complexity:** The BFAT is a 2D sparse matrix, storing only valid indices. Memory usage scales with the number of bricks and LODs. Not given in the sources. [S1, unofficial]
*   **Performance:** Enables out-of-core rendering by allowing efficient loading of bricks based on SFC locality. Performance depends on the efficiency of the Hilbert curve implementation and I/O throughput. Not given in the sources. [S1, unofficial]
*   **Strengths:** Cache-friendly access patterns due to SFC ordering; simple indexing via 1D arrays; supports out-of-core processing. [S1, unofficial]
*   **Limitations:** Requires pre-processing to convert regular grids to BSAMR; SFC indexing may not be optimal for all data distributions; sparse BFAT can lead to memory overhead if not implemented efficiently. [S1, unofficial]
*   **When to Use:** Interactive volume visualization of AMR time series where out-of-core processing and cache efficiency are priorities. [S1, unofficial]
*   **When Not to Use:** When unstructured mesh topology is required (use TET-AMR) or when exact geometric intersection is critical (use Exact AMR Rendering). [1094-tet-amr-data-structure.md][4399-exact-amr-rendering.md]

## Background
BSAMR was introduced in the context of GPU-based out-of-core architecture for interactive volume visualization of AMR time series. The method leverages Block-Structured AMR and Space-Filling Curves to enable efficient data access and rendering. The approach is detailed in a video presentation by an unspecified author. [S1, unofficial]

## Sources
[S1] GPU-based out-of-core architecture for interactive volume visualization of AMR time series., https://www.youtube.com/watch?v=pbikH-Rd9hI, fetched 2026-10-27 (unofficial)
