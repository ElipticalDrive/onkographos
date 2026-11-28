# Median-Cut

Also called: SAH-based Median-Cut

## Overview
Median-Cut is a method to build a spatial hierarchy bottom-up by choosing the direction and position of splits carefully [S1, unofficial]. It differs from standard median-cut color quantization by applying the split logic to spatial bounding boxes rather than color histograms, and it specifically employs the Surface Area Heuristic (SAH) to optimize the split position and direction, distinguishing it from simple geometric median splits [S1, unofficial]. It builds on [Greedy Axis-Aligned Spatial Partitioning](11589-greedy-axis-aligned-spatial-partitioning.md).

## How it works
**Inputs:** A set of sub-boxes (or primitives) represented by their axis-aligned bounding boxes (AABBs) and a parent bounding box matrix.
**Outputs:** Two disjoint lists of sub-boxes (left and right children) resulting from a single split.

**Data Structures:**
*   `sub_boxes`: A list of bounding box objects. Each object contains a transformation matrix (4x4) representing its position and orientation.
*   `inds`: A list of integer indices corresponding to the positions in `sub_boxes`.

**Steps:**
1.  **Initialize Indices:** Create an index list `inds` containing integers from $0$ to $N-1$, where $N$ is the number of sub-boxes.
2.  **Extract Axes:** Retrieve the x, y, and z axes from the parent bounding box's transformation matrix columns. Let these be vectors $\mathbf{x}_{axis}$, $\mathbf{y}_{axis}$, and $\mathbf{z}_{axis}$.
3.  **Calculate Scales:** Compute the squared length (scale) for each axis vector. For an axis vector $\mathbf{v}$, the scale $S$ is:
    $$ S = \sum_{i=1}^{3} v_i^2 $$
    Calculate $S_x$, $S_y$, and $S_z$ for the respective axes.
4.  **Determine Split Axis:** Identify the axis with the maximum squared scale. The `split_axis` vector $\mathbf{a}$ is the normalized version of that axis:
    $$ \mathbf{a} = \frac{\mathbf{axis}_{max}}{\|\mathbf{axis}_{max}\|} $$
    If $S_x$ is largest, $\mathbf{a} = \text{normalize}(\mathbf{x}_{axis})$. If $S_y$ is largest, $\mathbf{a} = \text{normalize}(\mathbf{y}_{axis})$. Otherwise, $\mathbf{a} = \text{normalize}(\mathbf{z}_{axis})$.
5.  **Project and Sort:** For each sub-box $i$ in `inds`, extract its center $\mathbf{c}_i$ (typically the 4th column of its transformation matrix). Compute the dot product $p_i = \mathbf{c}_i \cdot \mathbf{a}$. Sort the `inds` list based on $p_i$ in ascending order. Use a stable comparison wrapper (e.g., `CmpFloat`) to handle floating-point precision issues during sorting [S2, unofficial].
6.  **Split:** Divide the sorted `inds` list at the median index $M = \lfloor N / 2 \rfloor$. The left child contains indices `inds[0...M-1]` and the right child contains `inds[M...N-1]`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def median_cut_split(sub_boxes, parent_matrix):
    """
    Splits a list of sub-boxes into two groups based on the longest axis 
    of the parent bounding box using a median split on projected centers.
    
    Args:
        sub_boxes: List of objects with a 'matrix' attribute (4x4 matrix).
        parent_matrix: 4x4 matrix representing the parent bounding box.
        
    Returns:
        Tuple of two lists of indices (left_indices, right_indices).
    """
    n = len(sub_boxes)
    if n == 0:
        return [], []
    
    # 1. Initialize index list
    inds = list(range(n))
    
    # 2. Extract axes from parent matrix columns
    # Assuming column-major or row-major consistency; here we assume 
    # columns 0, 1, 2 are the basis vectors.
    x_axis = parent_matrix.column(0)
    y_axis = parent_matrix.column(1)
    z_axis = parent_matrix.column(2)
    
    # 3. Calculate squared length (scale) for each axis
    x_scale = dot(x_axis, x_axis)
    y_scale = dot(y_axis, y_axis)
    z_scale = dot(z_axis, z_axis)
    
    # 4. Determine split_axis as normalized vector of axis with largest squared length
    if x_scale >= y_scale and x_scale >= z_scale:
        split_axis = normalize(x_axis)
    elif y_scale >= x_scale and y_scale >= z_scale:
        split_axis = normalize(y_axis)
    else:
        split_axis = normalize(z_axis)
        
    # 5. Sort inds based on dot product of sub_box center with split_axis
    # Center is typically the 4th column (index 3) of the transformation matrix
    def get_projection(index):
        center = sub_boxes[index].matrix.column(3)
        return dot(center, split_axis)
    
    # Stable sort using the projection value
    # Note: In Python, sort is stable by default. 
    # The source mentions CmpFloat wrapper for stable comparison, 
    # implying handling of float equality.
    inds.sort(key=get_projection)
    
    # 6. Split at median index
    mid = n // 2
    left_indices = inds[:mid]
    right_indices = inds[mid:]
    
    return left_indices, right_indices

def dot(v1, v2):
    return sum(a * b for a, b in zip(v1, v2))

def normalize(v):
    length = dot(v, v) ** 0.5
    if length == 0:
        return v
    return [x / length for x in v]
```

**Parameters:**
*   `MAXDEPTH`: Typically 16 [S1, unofficial].
*   `MAXPRIMS`: Typically 2-8 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Floating Point Precision:** When multiple sub-boxes have centers with identical projections, the sort order must be stable to ensure deterministic behavior. A wrapper like `CmpFloat` is recommended [S2, unofficial].
*   **Degenerate Axes:** If the parent bounding box has a zero-length axis (degenerate box), normalization fails. The implementation should handle this by falling back to another axis or treating the box as a point/plane.
*   **Empty Lists:** If `sub_boxes` is empty, return empty lists immediately.

## Subtopics

### Median-Cut Spatial Partitioning
This variant explicitly defines the splitting logic as projecting sub-box centers onto the longest axis of the parent bounding box and splitting at the median index [S2, unofficial]. It is chosen when a deterministic, axis-aligned split based on spatial extent is required without complex cost calculations. It differs from SAH-based variants by relying purely on geometric median projection rather than ray-casting cost estimation.

## Operating characteristics
**Time Complexity:** Sorting the indices takes $O(N \log N)$ per node, where $N$ is the number of primitives in the node. Building the full tree is $O(N \log N)$ on average.
**Memory Complexity:** $O(N)$ to store the tree structure and indices.
**Performance:** The method is fast to build due to the simplicity of the median split. However, it may not produce the most optimal tree for ray traversal compared to SAH-based methods that evaluate multiple split positions.
**Strengths:** Simple to implement, deterministic, fast construction.
**Limitations:** Does not account for primitive density or ray distribution, potentially leading to unbalanced trees in non-uniform scenes.
**When to use:** When fast build times are prioritized over optimal traversal performance, or as a baseline for comparison.
**When not to use:** In scenes with highly non-uniform primitive distribution where SAH optimization is critical for performance.

## Background
The concept of Median-Cut in spatial partitioning originates from the broader field of ray tracing acceleration structures. The specific application of SAH to guide the split direction and position is discussed in ray tracing literature [S1, unofficial]. The implementation details provided in [S2, unofficial] reflect a modern software rendering context (SDF rendering).

## Sources
[S1] PPT - Ray Tracing 2: Acceleration PowerPoint Presentation, free ..., https://www.slideserve.com/chaim/ray-tracing-2-acceleration, fetched 2026-10-26 (unofficial)
[S2] holger-kibur/sdf, https://github.com/holger-kibur/sdf, fetched 2026-10-27 (unofficial)
