# Parallel Projected Tetrahedra with KD-Tree Partitioning

Also called: Parallel Unstructured Volume Rendering in ParaView, Parallel Volume Rendering in ParaView, kd-tree based parallel volume rendering

## Overview
This method enables the parallel rendering of large-scale unstructured tetrahedral grids by partitioning the dataset using a kd-tree, allowing the application of a traditional sort-last compositing algorithm [S1]. Unlike approaches that sample the volume into a regular grid—which require extensive parallel distribution of samples and suffer from oversampling or undersampling in varying cell sizes—this method renders the unstructured grid natively [S1]. It builds on [Projected Tetrahedra](877-projected-tetrahedra.md) for the per-process rendering and [Parallelized Accurate Sorting for Projected Tetrahedra](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md) for the final image composition, but distinguishes itself by handling the spatial decomposition of the unstructured data itself rather than relying on uniform grid sampling or complex cell-transfer protocols after distribution [S1].

## How it works
**Inputs:** An unstructured tetrahedral grid dataset (vertices and tetrahedra connectivity) and a target number of rendering processes $P$.
**Outputs:** A single composited image.

**Data Structures:**
1.  **KD-Tree:** A binary tree structure used to partition the spatial domain. Each node represents a region of space defined by an axis-aligned splitting plane. The tree is built recursively by splitting the bounding box of the tetrahedra.
2.  **Partitioned Tetrahedra:** The original tetrahedra are split if they intersect with kd-tree region boundaries. This results in a set of clipped tetrahedra assigned to specific processes. The data structure for each tetrahedron retains its four vertices and associated scalar values, but the total count increases.

**Steps:**

1.  **KD-Tree Construction and Partitioning:**
    *   Build a kd-tree over the bounding box of the entire unstructured grid.
    *   The tree depth is determined by the number of processes $P$. Typically, a single level of the kd-tree doubles the number of partitions. For $P$ processes, the tree depth is $\lceil \log_2 P \rceil$ [S1].
    *   Assign each leaf node of the kd-tree to a specific process.

2.  **Cell Splitting and Distribution:**
    *   Iterate through all tetrahedra in the original dataset.
    *   For each tetrahedron, determine which kd-tree leaf regions it intersects.
    *   If a tetrahedron intersects multiple regions (i.e., it crosses a splitting plane), split the tetrahedron into smaller sub-tetrahedra along the plane. The scalar values at the new vertices are interpolated linearly from the original vertex values [S1].
    *   Distribute the resulting (possibly split) tetrahedra to the processes assigned to the corresponding kd-tree leaf regions.
    *   This partitioning step is performed whenever the data changes, not every render frame [S1].

3.  **Local Rendering:**
    *   Each process renders its assigned subset of tetrahedra using the [Projected Tetrahedra](877-projected-tetrahedra.md) method.
    *   This involves projecting the tetrahedra onto the image plane, sorting the projected polygons back-to-front, and compositing them locally to produce a partial image [S1].

4.  **Parallel Image Compositing (Sort-Last):**
    *   Transfer only the rendered image data (pixels) from each process to a designated compositing node (or use a parallel reduction algorithm).
    *   Perform a final sort-last compositing of the partial images to produce the final result. No further cell transfer is required after the initial distribution [S1].

**Parameters:**
*   **Number of Processes ($P$):** Determines the depth of the kd-tree.
*   **Image Resolution:** Experiments used 800x800 pixels [S1].

**Edge Cases and Pitfalls:**
*   **Data Growth:** Splitting cells causes the dataset size to grow. On average, data size grows by about 15% per level of the kd-tree, with variance between 5% and 30% [S1]. Larger datasets exhibit smaller relative growth.
*   **Memory Constraints:** The redistribution process requires holding at least two copies of the data simultaneously (original and partitioned). If hardware memory is insufficient, rendering rates can drop impractically low [S1].
*   **Geometry Changes:** Changes in viewable geometry or data structure require lengthy redistribution and re-partitioning of the data [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_projected_tetrahedra_with_kd_tree(tetrahedra, num_processes, image_width, image_height):
    """
    Parallel rendering of unstructured tetrahedral grid using kd-tree partitioning.
    
    Args:
        tetrahedra: List of tetrahedra, each defined by 4 vertices and scalar values.
        num_processes: Number of parallel processes.
        image_width: Width of the output image.
        image_height: Height of the output image.
        
    Returns:
        Final composited image.
    """
    # Step 1: Build KD-Tree
    # The kd-tree partitions the spatial domain into num_processes regions.
    # Each leaf node corresponds to one process.
    kd_tree = build_kd_tree(tetrahedra, num_processes)
    
    # Step 2: Partition and Distribute Tetrahedra
    # Each process will receive a subset of tetrahedra.
    # If a tetrahedron crosses a boundary, it is split.
    process_tetrahedra = [ [] for _ in range(num_processes) ]
    
    for tet in tetrahedra:
        # Determine which kd-tree leaves this tetrahedron intersects
        intersecting_leaves = find_intersecting_leaves(tet, kd_tree)
        
        if len(intersecting_leaves) == 1:
            # Tetrahedron lies entirely within one partition
            process_tetrahedra[intersecting_leaves[0]].append(tet)
        else:
            # Tetrahedron crosses boundaries; split it
            split_tets = split_tetrahedron(tet, intersecting_leaves, kd_tree)
            for split_tet, leaf_idx in split_tets:
                process_tetrahedra[leaf_idx].append(split_tet)
                
    # Step 3: Local Rendering
    # Each process renders its assigned tetrahedra using Projected Tetrahedra
    local_images = []
    for proc_id in range(num_processes):
        # Render using the method described in [Projected Tetrahedra](877-projected-tetrahedra.md)
        local_img = render_projected_tetrahedra(process_tetrahedra[proc_id], image_width, image_height)
        local_images.append(local_img)
        
    # Step 4: Parallel Image Compositing
    # Combine local images using sort-last compositing
    final_image = composite_images(local_images)
    
    return final_image

def build_kd_tree(tetrahedra, num_processes):
    """
    Builds a kd-tree to partition the space into num_processes regions.
    """
    # Implementation details depend on specific kd-tree library or algorithm.
    # The goal is to create a binary tree with num_processes leaves.
    # Each split divides the bounding box of the tetrahedra.
    pass

def find_intersecting_leaves(tet, kd_tree):
    """
    Finds which leaf nodes of the kd-tree a tetrahedron intersects.
    """
    # Check intersection of tetrahedron with splitting planes in the kd-tree.
    # Return list of leaf indices.
    pass

def split_tetrahedron(tet, leaves, kd_tree):
    """
    Splits a tetrahedron that intersects multiple kd-tree leaves.
    Returns a list of (split_tet, leaf_index) pairs.
    """
    # Clip the tetrahedron against the splitting planes defined by the kd-tree.
    # Interpolate scalar values at new vertices.
    pass

def render_projected_tetrahedra(tetrahedra, width, height):
    """
    Renders a list of tetrahedra using the Projected Tetrahedra method.
    See [Projected Tetrahedra](877-projected-tetrahedra.md) for details.
    """
    pass

def composite_images(images):
    """
    Composites a list of images using sort-last compositing.
    """
    pass
```

## Subtopics

### Projected Tetrahedra
The core rendering primitive used by each process. It projects tetrahedra to 2D, sorts them, and composites them. [Link](877-projected-tetrahedra.md)

### Parallelized Accurate Sorting for Projected Tetrahedra
The compositing strategy used to combine the partial images from different processes. [Link](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md)

### Tetrahedral Representation and Traversal
The underlying data structure and traversal principles for unstructured grids. [Link](4396-tetrahedral-representation-and-traversal.md)

## Operating characteristics
*   **Time Complexity:** The partitioning step is performed infrequently (only when data changes). The rendering step scales with the number of tetrahedra per process. The compositing step scales with the number of processes and image resolution.
*   **Memory Complexity:** Requires holding at least two copies of the data during redistribution (original and partitioned) [S1]. The data size grows by an average of 15% per kd-tree level, with variance 5-30% [S1].
*   **Performance:** Rendering rates increase with the number of processors but show diminishing returns when adding more processors than necessary for a given dataset [S1]. Larger datasets show bigger speedups even if overall frame rates are lower [S1].
*   **Strengths:** Renders unstructured grids natively without sampling into a regular grid. No further cell transfer is required after initial distribution; only image data is communicated [S1].
*   **Limitations:** Data redistribution requires significant memory (two copies). Changes in viewable geometry require lengthy redistribution [S1]. Rendering rates can be impractically low if hardware memory is insufficient [S1].
*   **When to Use:** Large-scale unstructured grid visualization on distributed-memory parallel computers, especially when native unstructured rendering is preferred over grid sampling.
*   **When Not to Use:** When memory is constrained and cannot hold two copies of the data. When frequent geometry changes require constant redistribution.

## Background
This method was developed for ParaView and deployed at Sandia National Laboratories for visualizing simulations from supercomputers like Red Storm [S1]. It addresses the challenge of rendering large unstructured grids generated by high-fidelity physical simulations [S1]. The approach leverages the efficiency of the projected tetrahedra method and extends it to a parallel environment using kd-tree partitioning [S1].

## Sources
[S1] Parallel Unstructured Volume Rendering in ParaView, https://www.kennethmoreland.com/documents/ParallelVolRen.pdf, fetched 2026-10-27
