# Y-Bucket Parallelization

Also called: Y-bucket grouping

## Overview
Y-Bucket Parallelization is a two-pass parallel algorithm for direct volume rendering that groups polygons representing isosurfaces or volume slices into scanline-based "y-buckets" to maximize coherence and avoid the difficulties of ray-casting intersecting cells [S1]. It solves the problem of rendering irregular and multiple grids efficiently in software by parallelizing on a scanline basis, which limits duplication compared to tile-based methods that break scanlines and lose coherence [S1].

## How it works
The method processes polygons representing volume data (stored in counter-clockwise order when viewed from outside the grid) to determine visibility order [S1].

**Inputs:**
- A set of polygons representing volume slices or isosurfaces.
- A set of processing units (processors).

**Outputs:**
- A rendered image where pixels are filled based on front-to-back visibility.

**Data Structures:**
1.  **Y-Bucket Array:** An array where each entry corresponds to a specific scanline (y-coordinate). It stores the count of polygons belonging to that scanline.
2.  **Output Array:** A common array partitioned to store the final pixel data.
3.  **X-Bucket Structures:** Linked lists associated with the leftmost pixel of a polygon's horizontal span. These lists maintain polygons in front-to-back visibility order.

**Steps:**

1.  **First Pass (Polygon Distribution):**
    - Each processor distributes polygons into y-buckets based on their vertical extent.
    - Polygons are assigned to the y-buckets corresponding to the scanlines they cover.
    - Invalidated polygons are marked or excluded.

2.  **Second Pass (Bucket Allocation and Structure Creation):**
    - The number of polygons belonging to each y-bucket is determined by accessing the Y-Bucket Array.
    - Space for each y-bucket is allocated by partitioning the common Output Array.
    - The space for one y-bucket is further partitioned into space for each processor to fill within that y-bucket.
    - Each processor creates appropriate y-bucket structures for all polygons it processed in the first pass, excluding those invalidated.
    - *Note:* This step is transitional. Its work per processor is proportional to the number of scanlines, not the number of polygons [S1].

3.  **Third Pass (Scanline Processing):**
    - Scanlines are processed in order from bottom to top.
    - For each active polygon, information is transferred to an x-bucket data structure associated with the leftmost pixel in which the polygon appears.
    - A linked list of these x-bucket structures is maintained in front-to-back visibility order.
    - The bulk of the computation (compositing) occurs here.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def y_bucket_parallelization(polygons, num_processors, num_scanlines):
    # Step 1: First Pass - Distribute polygons to y-buckets
    # Each processor handles a subset of polygons
    y_bucket_counts = [0] * num_scanlines
    
    for polygon in polygons:
        # Determine which scanlines this polygon covers
        y_min, y_max = polygon.vertical_extent()
        for y in range(y_min, y_max + 1):
            y_bucket_counts[y] += 1
            
    # Step 2: Second Pass - Allocate space and create structures
    # Partition common output array based on y_bucket_counts
    output_array = allocate_output_array(num_scanlines, y_bucket_counts)
    
    # Partition space for each processor within each y-bucket
    # This is a transitional step; work is proportional to scanlines
    for y in range(num_scanlines):
        # Allocate space for processor p in y-bucket y
        # This involves creating the necessary data structures
        create_y_bucket_structures(y, y_bucket_counts[y], num_processors)
        
    # Each processor creates structures for its polygons
    for processor in processors:
        for polygon in processor.polygons:
            if not polygon.is_invalidated():
                create_y_bucket_entry(polygon)
                
    # Step 3: Third Pass - Process scanlines
    for y in range(num_scanlines): # Bottom to top
        # Transfer active polygon info to x-buckets
        # X-buckets are linked lists in front-to-back order
        x_buckets = process_scanline(y)
        
        # Perform compositing for the scanline
        render_scanline(y, x_buckets)
```

**Parameters:**
- Number of processors: Determines the partitioning of space within y-buckets.
- Number of scanlines: Determines the size of the y-bucket array and the work in the second pass.

**Edge Cases and Pitfalls:**
- The second pass is a transitional step and can be parallelized by standard techniques, but its workload is small relative to the third pass [S1].
- Polygons must be stored in counter-clockwise order to correctly identify boundaries and visibility order [S1].

## Subtopics

### [Parallel Direct Volume Rendering with Y-Buckets and X-Buckets](2123-parallel-direct-volume-rendering-with-y-buckets-and-x-bucket.md)
This method extends Y-Bucket Parallelization by introducing "x-buckets" and "x-actives" lists to maintain front-to-back visibility order within a scanline. It is chosen when rendering irregular and multiple grids in software to maximize coherence.

## Operating characteristics
- **Time Complexity:** The second pass work is proportional to the number of scanlines, not the number of polygons. The third pass involves the bulk of the computation, processing each scanline and managing linked lists of active polygons [S1].
- **Memory Complexity:** Requires storage for y-bucket counts, the output array, and x-bucket linked lists.
- **Strengths:** Maximizes coherence by processing scanlines; avoids the difficulties of ray-casting intersecting cells; limits duplication compared to tile-based methods [S1].
- **Limitations:** The second pass is a transitional step with limited parallelism benefit; relies on software processing of polygons.

## Background
The method was introduced in the context of hierarchical and parallelizable direct volume rendering for irregular and multiple grids [S1]. It addresses the challenges of rendering complex volume data by leveraging scanline-based parallelization.

## Sources
[S1] Hierarchical and parallelizable direct volume rendering for irregular and multiple grids, https://doi.org/10.1109/visual.1996.567606, fetched 2026-10-24
