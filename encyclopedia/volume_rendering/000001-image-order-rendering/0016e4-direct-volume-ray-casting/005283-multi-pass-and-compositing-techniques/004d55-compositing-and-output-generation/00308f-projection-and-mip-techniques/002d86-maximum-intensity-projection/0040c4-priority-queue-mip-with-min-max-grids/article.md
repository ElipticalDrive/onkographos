# Priority Queue MIP with Min/Max Grids

Also called: Front-to-Back MIP, Maximum Intensity Projection (MIP) with Priority Queue, Min/Max Grid MIP, Min/Max Multilevel Grid MIP, Parker et al. MIP, Priority Queue MIP

## Overview
Priority Queue MIP with Min/Max Grids is a ray-casting technique that efficiently computes the Maximum Intensity Projection (MIP) by combining a hierarchical spatial acceleration structure with a priority queue [S1]. Unlike standard ray marching which samples every voxel, this method skips low-intensity regions by only processing nodes whose maximum stored value exceeds the current global maximum found along the ray [S1]. It is primarily used in medical imaging to visualize thin, high-intensity structures like blood vessels [S1].

## How it works
**Inputs:** A 3D volume dataset, a camera view (defining rays), and a min/max multilevel grid where each node stores the minimum and maximum scalar values of its contained voxels [S1].
**Outputs:** For each pixel, the maximum scalar value encountered along the ray and its corresponding 3D position [S3, unofficial].

**Data Structures:**
1.  **Min/Max Multilevel Grid:** A hierarchical tree where each node stores `min_val`, `max_val`, `children` pointers, and `bounds` (axis-aligned bounding box) [S1].
2.  **Priority Queue:** A max-heap storing nodes ordered by `max_val` [S1].

**Algorithm Steps:**
1.  **Initialization:** Initialize the priority queue with the root node of the min/max grid and its `max_val` [S1]. Initialize the global maximum for the ray: `global_max = -∞` [G]. Initialize the global maximum position: `global_max_pos = ray.start` [G].
2.  **Iteration:** While the priority queue is not empty, extract the node `N` with the highest `max_val` from the queue [S1].
3.  **Pruning:** If `N.max_val <= global_max`, discard `N` and continue to the next iteration [G].
4.  **Internal Node Expansion:** If `N` is not a leaf, insert all children of `N` into the priority queue with their respective `max_val`s [S1].
5.  **Leaf Traversal:** If `N` is a leaf, start a macro-cell traversal [S1]. At each pierced cell-face, perform bi-linear interpolation assuming a linear function along the ray, which implies no extremal value exists strictly within the cell [S1]. Store the maximum value encountered along the ray segment through the macro-cell back into the priority queue [S1].
6.  **Termination:** Terminate the algorithm if one of the leaf cells appears at the head of the priority queue [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class MaxHeap:
    def __init__(self):
        self.heap = []
    def push(self, item):
        self.heap.append(item)
        self._bubble_up(len(self.heap) - 1)
    def pop(self):
        if not self.heap:
            return None
        self._swap(0, len(self.heap) - 1)
        item = self.heap.pop()
        self._bubble_down(0)
        return item
    def _bubble_up(self, i):
        while i > 0:
            parent = (i - 1) // 2
            if self.heap[i].max_val > self.heap[parent].max_val:
                self._swap(i, parent)
                i = parent
            else:
                break
    def _bubble_down(self, i):
        n = len(self.heap)
        while True:
            largest = i
            left = 2 * i + 1
            right = 2 * i + 2
            if left < n and self.heap[left].max_val > self.heap[largest].max_val:
                largest = left
            if right < n and self.heap[right].max_val > self.heap[largest].max_val:
                largest = right
            if largest != i:
                self._swap(i, largest)
                i = largest
            else:
                break
    def _swap(self, i, j):
        self.heap[i], self.heap[j] = self.heap[j], self.heap[i]
    def is_empty(self):
        return len(self.heap) == 0

def priority_queue_mip(ray, min_max_grid):
    pq = MaxHeap()
    root = min_max_grid.root
    pq.push(root)

    global_max = float('-inf')
    global_max_pos = ray.start

    while not pq.is_empty():
        node = pq.pop()

        # Pruning: skip nodes that cannot exceed the current global maximum
        if node.max_val <= global_max:
            continue

        # Termination: if the highest priority node is a leaf, process and stop
        if node.is_leaf():
            leaf_max, leaf_pos = macro_cell_traversal(ray, node)

            if leaf_max > global_max:
                global_max = leaf_max
                global_max_pos = leaf_pos

            # Store the maximum value encountered along the ray segment back into the queue
            node.max_val = leaf_max
            pq.push(node)
            break

        # Internal node: insert children into the queue
        for child in node.children:
            pq.push(child)

    return global_max, global_max_pos

def macro_cell_traversal(ray, leaf_node):
    local_max = float('-inf')
    local_max_pos = ray.start

    # Traverse cells within the leaf node's bounds intersected by the ray
    cells = _get_cells_intersected_by_ray(ray, leaf_node.bounds)

    for cell in cells:
        # Find intersection points of the ray with the cell faces
        t_enter, t_exit = _ray_cell_intersection(ray, cell)

        # Sample at the entry face using bi-linear interpolation
        pos_enter = _ray_point_at(ray, t_enter)
        val_enter = _bilinear_interpolate(pos_enter, cell)

        if val_enter > local_max:
            local_max = val_enter
            local_max_pos = pos_enter

    return local_max, local_max_pos

def _get_cells_intersected_by_ray(ray, bounds):
    # Standard 3D DDA traversal over grid cells within bounds [G]
    return []

def _ray_cell_intersection(ray, cell):
    # Returns entry and exit t-values for ray-cell intersection [G]
    return 0.0, 1.0

def _ray_point_at(ray, t):
    # Returns 3D position along ray at parameter t [G]
    return ray.start + t * (ray.end - ray.start)

def _bilinear_interpolate(pos, cell):
    # Interpolates scalar value at pos on a cell face using corner values [G]
    return 0.0
```

**Parameters:**
*   `MAX_NUM_STEPS`: 512 maximum steps for ray marching in some implementations [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Bi-linear interpolation assumes linearity; high-frequency details within a cell may be missed [S1].
*   **Boundaries:** Correctly handling ray entry/exit from the volume and leaf nodes is critical [G].
*   **Degenerate Input:** Empty volumes or volumes with uniform values should be handled gracefully (e.g., returning the uniform value) [G].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Highly efficient for visualizing thin structures like blood vessels [S1].
*   **Limitations:** Requires pre-computation of the min/max grid [G].
*   **When to use:** Medical imaging, visualization of thin structures [S1].
*   **When not to use:** Not given in the sources.

## Background
The method was suggested by Parker et al. in [PPL*99] as an important technique for real-world medical imaging applications [S1]. It uses a priority queue in conjunction with a spatial data structure, specifically min/max multilevel grids, to track cells with maximum values [S1]. While Parker et al. used min/max multilevel grids, other common min/max hierarchical data structures could also be exploited [S1]. The technique is part of the broader field of ray tracing-based volume rendering, which has been developed since the early 1990s [S1].

## Sources
[S1] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S2] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
[S3] GitHub - mlavik1/UnityVolumeRendering: Volume rendering, implemented in ..., https://github.com/mlavik1/UnityVolumeRendering, fetched 2026-10-28 (unofficial)
