# Hierarchical Out-of-Core Indexing for Orthogonal Slicing

Also called: 3-bit/1-bit shift hierarchical indexing, Disk I/O complexity for grid slicing, Hierarchical Z-order Indexing for Slicing, Out-of-Core Complexity Analysis for Orthogonal Slicing

## Overview
This method defines a data layout and indexing scheme for 3D rectilinear grids that minimizes disk I/O during orthogonal slicing, particularly at coarse resolutions or in progressive rendering scenarios. It solves the problem where standard blocking schemes suffer from severe performance degradation when subsampling, as entire large blocks must be loaded even if only a fraction is used. Unlike standard $h \times h \times h$ blocking, which degrades to $O(t)$ block transfers at coarse subsampling, this hierarchical scheme maintains a worst-case complexity of $O(\frac{t}{\sqrt[3]{b^2}})$ regardless of the subsampling rate or slicing direction [S1, unofficial]. It builds on [Global Static Indexing for Regular Grids](21466-global-static-indexing-for-regular-grids.md) by applying a specific hierarchical decomposition (oct-tree or binary tree) to the storage layout to ensure that block coverage scales efficiently with resolution.

## How it works
The method relies on a hierarchical decomposition of the grid data, mapping the 3D space to a structure that allows efficient subsampling. The two primary variants are the 3-bit shift (oct-tree based) and 1-bit shift (binary tree based) schemes.

### Inputs and Outputs
*   **Input:** A 3D rectilinear grid of size $g$ (total number of voxels).
*   **Output:** A sequence of sliced data values at a specified resolution and subsampling rate $k$.
*   **Disk Block Size:** $b$ bytes.

### Data Structures
The data is stored on disk in blocks of size $b$. The indexing scheme determines which blocks are accessed.

1.  **3-bit Shift Scheme (Oct-tree):**
    *   The grid is decomposed hierarchically. On a 2D slice, this maps to a quadtree structure.
    *   Block size: Typically 28KB (in the cited experimental context).
    *   Overhead factor in number of blocks: $1 + \frac{1}{4} + \frac{1}{16} + \frac{1}{64} + \dots < \frac{4}{3}$.
    *   At finest resolution, overhead is ~16.3% [S1, unofficial].

2.  **1-bit Shift Scheme (Binary Tree):**
    *   The grid is decomposed hierarchically.
    *   Block size: Typically 16KB (half the size of the 3-bit scheme to cover equivalent mesh regions).
    *   Overhead factor in number of blocks at finest resolution: 2 [S1, unofficial].

### Numbered Steps
1.  **Decompose Grid:** Partition the 3D grid into a hierarchical structure (oct-tree for 3-bit shift, binary tree for 1-bit shift).
2.  **Map to Disk:** Store the decomposed data in blocks of size $b$. The hierarchical structure ensures that blocks at coarser levels cover larger spatial regions.
3.  **Determine Slicing Plane:** Identify the orthogonal plane (e.g., XY, YZ, or XZ) and the subsampling rate $k$.
4.  **Identify Blocks:** For the target slice at subsampling rate $k$, identify the set of blocks that intersect the slice.
    *   In the 3-bit shift scheme, the number of blocks loaded is bounded by the geometric series sum $< \frac{4}{3}$ times the optimal number at fine resolution.
    *   In the 1-bit shift scheme, the number of blocks is bounded by a factor of 2 at fine resolution.
5.  **Transfer Data:** Load the identified blocks from disk.
6.  **Extract Samples:** Extract the required samples from the loaded blocks based on the subsampling rate $k$.

### Formulas
*   **Standard Row-Major Storage:**
    *   Best case: $O(t/b)$ blocks.
    *   Worst case: $O(t)$ blocks (for $b = \Omega(\sqrt[3]{g})$) [S1, unofficial].
*   **Standard $h \times h \times h$ Blocking ($h = \sqrt[3]{b}$):**
    *   Fine resolution: $O(\frac{t}{\sqrt[3]{b^2}})$ blocks.
    *   Coarse subsampling ($k < \sqrt[3]{b}$): $O(\frac{tk^2}{\sqrt[3]{b^2}})$ blocks, degrading to $O(t)$ [S1, unofficial].
*   **Hierarchical Scheme (Proposed):**
    *   Worst case (any subsampling rate, any direction): $O(\frac{t}{\sqrt[3]{b^2}})$ blocks.
    *   This is because each block covers a portion of $\sqrt[3]{b^2}$ [S1, unofficial].
*   **Subsampling Benefit:**
    *   In the hierarchical scheme, doubling the subsampling rate $k$ reduces the amount of data loaded from external memory by a factor of four [S1, unofficial].

### Parameters
*   **Block Size ($b$):** Determined by disk page size (e.g., 10KB pages, leading to 28KB or 16KB blocks in experiments) [S1, unofficial].
*   **Subsampling Rate ($k$):** The ratio of input samples to output samples.

### Edge Cases and Pitfalls
*   **Coarse Subsampling with Standard Blocking:** Standard $h \times h \times h$ blocking performs poorly at coarse subsampling because entire blocks are loaded regardless of how little data is used. The hierarchical scheme avoids this [S1, unofficial].
*   **Precision:** Not given in the sources.
*   **Boundaries:** Not given in the sources.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_slice(grid, slice_plane, subsampling_rate, block_size):
    """
    Perform orthogonal slicing using hierarchical out-of-core indexing.
    
    Args:
        grid: 3D rectilinear grid data.
        slice_plane: 'XY', 'YZ', or 'XZ'.
        subsampling_rate: Integer k, sample every k-th voxel.
        block_size: Size of disk blocks in bytes.
        
    Returns:
        List of sliced values.
    """
    # Determine hierarchical structure based on shift type
    # 3-bit shift (oct-tree) or 1-bit shift (binary tree)
    # For this example, we assume 3-bit shift (oct-tree)
    
    # Identify blocks that intersect the slice
    # The number of blocks is bounded by < 4/3 * optimal_blocks at fine res
    # and scales down by factor of 4 when subsampling_rate doubles
    
    blocks_to_load = identify_intersecting_blocks(
        grid, slice_plane, subsampling_rate, block_size
    )
    
    loaded_data = []
    for block in blocks_to_load:
        # Load block from disk
        block_data = load_block_from_disk(block)
        
        # Extract samples based on subsampling_rate
        samples = extract_samples(block_data, slice_plane, subsampling_rate)
        loaded_data.extend(samples)
        
    return loaded_data

def identify_intersecting_blocks(grid, slice_plane, subsampling_rate, block_size):
    """
    Identify blocks that intersect the slice.
    Uses hierarchical decomposition to minimize block count.
    """
    # Implementation details depend on specific hierarchical indexing scheme
    # For 3-bit shift:
    # Map 3D grid to oct-tree, then to quadtree on slice
    # Calculate block indices that cover the slice region at given subsampling
    
    # Placeholder for specific indexing logic
    # The key property is that block count is O(t / b^(2/3)) regardless of k
    pass

def extract_samples(block_data, slice_plane, subsampling_rate):
    """
    Extract samples from block data based on subsampling rate.
    """
    # Extract every k-th sample along the slice plane
    pass
```

## Operating characteristics
*   **Time Complexity:** The number of disk blocks transferred is $O(\frac{t}{\sqrt[3]{b^2}})$ in the worst case, independent of subsampling rate and slicing direction [S1, unofficial].
*   **Memory Complexity:** Depends on the number of blocks loaded simultaneously. At fine resolution, it is comparable to standard blocking. At coarse resolution, it is significantly lower than standard blocking because fewer blocks are needed [S1, unofficial].
*   **Measured Performance:**
    *   In an 8GB dataset with 10KB disk pages and 4MB slice size:
        *   Standard blocking ($32 \times 32 \times 32$ blocks): Loads 128MB (32x output) at fine resolution. Performance does not improve with subsampling up to rate 32 [S1, unofficial].
        *   3-bit shift scheme: Overhead of ~16.3% at fine resolution. Data loaded reduces by factor of 4 when subsampling rate doubles [S1, unofficial].
        *   1-bit shift scheme: Overhead of 2 at fine resolution. Data loaded reduces by factor of 4 when subsampling rate doubles [S1, unofficial].
*   **Strengths:** Consistent performance across all subsampling rates and slicing directions. Significant I/O reduction at coarse resolutions.
*   **Limitations:** Slightly higher overhead at finest resolution compared to optimal standard blocking (16.3% for 3-bit shift, 100% for 1-bit shift).
*   **When to use:** Progressive rendering, interactive visualization of large datasets, scenarios with varying subsampling rates.
*   **When not to use:** When only finest resolution rendering is needed and standard blocking is already optimal.

## Background
This method was proposed in the context of out-of-core volume rendering of large rectilinear grids. It addresses the limitations of standard row-major and block-wise storage layouts. The work is based on the analysis of I/O complexity for orthogonal slicing.

## Sources
[S1] Multi-Resolution Indexing for Hierarchical Out-of-Core Traversal of Rectilinear Grids, https://digital.library.unt.edu/ark:/67531/metadc724429/?q=out-of-core volume rendering, fetched 2026-10-28 (unofficial)
