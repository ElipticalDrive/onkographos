# Y-Bucket Parallelization

Also called: Y-bucket grouping

## Overview
Y-Bucket Parallelization is a two-pass parallel algorithm designed for direct volume rendering of irregular or multiple grids, specifically addressing the inefficiencies of ray-casting intersecting cells [S1]. It solves the problem of rendering irregular polygonal data by grouping polygons into scanline-based "y-buckets" to maximize spatial coherence and avoid the duplication inherent in tile-based parallelization [S1]. Unlike [Tiled Parallel Compositing](1192-tiled-parallel-compositing.md) which breaks scanlines, this method parallelizes on a scanline basis to maintain front-to-back visibility order efficiently [S1].

## How it works
The method processes polygons representing isosurfaces or volume slices. It assumes polygons are stored in counter-clockwise order when viewed from outside the grid, allowing easy recognition of boundaries and visibility order [S1].

**Inputs:**
- A set of polygons representing the volume data.
- A number of processors $P$.
- The image resolution (width $W$, height $H$).

**Outputs:**
- A rendered image buffer.

**Data Structures:**
1.  **Y-Bucket Array:** An array of size $H$, where each entry corresponds to a scanline (y-coordinate). It stores a list of polygons intersecting that scanline.
2.  **Output Array:** A common output array partitioned to hold the final pixel contributions for each y-bucket.
3.  **X-Bucket:** A data structure associated with the leftmost pixel of a polygon on a scanline. It contains a linked list of active polygons in front-to-back visibility order.

**Steps:**

1.  **Pass 1: Polygon Distribution**
    - Each processor distributes polygons into the Y-Bucket Array based on their y-coordinate range.
    - This step identifies which polygons belong to which scanlines.

2.  **Pass 2: Bucket Allocation and Structure Creation**
    - The number of polygons belonging to each y-bucket is determined by accessing the Y-Bucket Array.
    - Space for each y-bucket is allocated by partitioning the common output array.
    - The space for one y-bucket is further partitioned into space for each processor to fill within that y-bucket.
    - Each processor creates appropriate y-bucket structures for all polygons it processed in Pass 1, excluding any invalidated polygons.
    - *Note:* While this step can be parallelized, its work per processor is proportional to the number of scanlines, not the number of polygons [S1].

3.  **Pass 3: Scanline Processing (X-Bucket Construction)**
    - Process each scanline in order from bottom to top.
    - For each active polygon on the scanline, transfer information to an X-Bucket data structure associated with the leftmost pixel where the polygon appears.
    - Maintain a linked list of these X-Buckets in front-to-back visibility order.
    - Perform the bulk of the rendering computation (e.g., compositing) using the active polygon lists.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def y_bucket_parallelization(polygons, num_processors, image_height):
    # Pass 1: Distribute polygons into Y-Buckets
    # Each processor handles a subset of polygons
    y_buckets = [ [] for _ in range(image_height) ]
    
    for poly in polygons:
        # Determine which scanlines (y-coordinates) the polygon intersects
        y_min, y_max = get_y_range(poly)
        for y in range(y_min, y_max + 1):
            y_buckets[y].append(poly)
            
    # Pass 2: Allocate output space and create structures
    # Partition common output array based on y-bucket sizes
    output_array = allocate_output_array(y_buckets)
    
    # Each processor creates y-bucket structures for its polygons
    # Work is proportional to number of scanlines, not polygons
    for y in range(image_height):
        # Partition space for this y-bucket among processors
        # Create structures for polygons in y_buckets[y]
        pass # Implementation details for partitioning omitted as per source focus on logic
        
    # Pass 3: Process scanlines bottom-to-top
    for y in range(image_height):
        # Initialize X-Buckets for this scanline
        x_buckets = {}
        
        # Get polygons for this scanline
        active_polys = y_buckets[y]
        
        # Sort polygons by depth/visibility order if not already sorted
        # Create linked lists in X-Buckets in front-to-back order
        for poly in active_polys:
            x_start = get_leftmost_pixel_x(poly)
            if x_start not in x_buckets:
                x_buckets[x_start] = Linkedlist()
            x_buckets[x_start].append(poly)
            
        # Perform bulk computation (compositing) using X-Buckets
        for x_start, bucket in x_buckets.items():
            render_scanline_segment(x_start, bucket)
```

**Edge Cases and Pitfalls:**
- Invalidated polygons must be excluded in Pass 2 [S1].
- The second pass is a transitional step; parallelizing it further yields diminishing returns as work is scanline-proportional [S1].

## Subtopics

### [Parallel Direct Volume Rendering with Y-Buckets and X-Buckets](2123-parallel-direct-volume-rendering-with-y-buckets-and-x-bucket.md)
Extends the Y-Bucket method by introducing X-Buckets and X-Actives lists to process pixels within buckets, maintaining front-to-back visibility order. Chosen when rendering irregular grids in software to maximize coherence.

## Operating characteristics
- **Time Complexity:** Pass 2 work per processor is proportional to the number of scanlines, not the number of polygons [S1]. Pass 3 involves processing each scanline, with computation proportional to the number of active polygons per scanline.
- **Memory Complexity:** Requires storage for the Y-Bucket Array and the Output Array. Space is partitioned per y-bucket and per processor within each bucket.
- **Strengths:** Maximizes coherence by processing scanlines; avoids difficulties of ray-casting intersecting cells; efficient for irregular and multiple grids [S1].
- **Limitations:** Pass 2 is a transitional step with limited parallelism benefits; requires careful management of polygon distribution and validity [S1].
- **When to Use:** Software rendering of irregular/multiple grids where ray-casting is inefficient.
- **When Not to Use:** When hardware-accelerated ray-casting or tile-based methods are preferred for regular grids.

## Background
The method was introduced in 1996 by the authors of "Hierarchical and parallelizable direct volume rendering for irregular and multiple grids" [S1]. It was developed to address the challenges of rendering irregular and multiple grids efficiently in software, where traditional ray-casting methods struggle with intersecting cells and lack of coherence.

## Sources
[S1] Hierarchical and parallelizable direct volume rendering for irregular and multiple grids, https://doi.org/10.1109/visual.1996.567606, fetched 2026-10-24
