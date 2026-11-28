# Pointer Chasing BVH Traversal


## Overview
Pointer Chasing BVH Traversal is a method for traversing Bounding Volume Hierarchies (BVHs) that eliminates index arithmetic operations in favor of explicit pointer dereferences [S1]. It solves the problem of computational overhead in the traversal hot loop by replacing index-based child selection with direct memory access [S1]. It sets itself apart from its siblings, such as [PBRTv4 BVH Layout](22864-pbrtv4-bvh-layout.md), by using explicit child pointers rather than implicit indexing, which reduces memory-dependent operations on latency-bound architectures but severely underutilizes memory bandwidth on GPUs [S1].

## How it works
**Inputs and Outputs:**
- Input: A BVH structure where nodes contain explicit pointers to their children, and a ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Output: The intersection point or the leaf node containing the hit primitive.

**Data Structures:**
The BVH is stored as a tree of nodes. Each node contains:
- `bbox`: Bounding volume (e.g., AABB) for the subtree.
- `left_child`: Pointer to the left child node.
- `right_child`: Pointer to the right child node.
- `is_leaf`: Boolean flag indicating if the node is a leaf.
- `data`: Primitive indices or volume data (if leaf).

Unlike compact layouts that store nodes in contiguous arrays with implicit indexing (e.g., $2i+1, 2i+2$), this layout relies on the hardware's ability to fetch memory locations pointed to by `left_child` and `right_child` [S1].

**Steps:**
1. **Initialize:** Start at the root node. Initialize ray parameter $t_{min} = 0$ and $t_{max} = \infty$.
2. **Intersection Test:** Check if the ray intersects the current node's bounding box. If not, terminate traversal for this ray [S1].
3. **Child Selection:**
   - If the node is a leaf, process the contained primitives/volume data and terminate [S1].
   - If the node is internal, determine which child(ren) the ray intersects.
   - Unlike index-based methods that compute child indices via arithmetic, this method follows the explicit pointers `left_child` and `right_child` [S1].
   - Typically, the ray is tested against both children's bounding boxes. If both are intersected, both are pushed to the traversal stack or processed recursively. If only one is intersected, only that child is followed [S1].
4. **Traversal:**
   - Update the current node pointer to the selected child.
   - Repeat from Step 2.
5. **Backtracking:** If a path is exhausted (no more children or no intersection), backtrack to the parent node using the traversal stack or recursion [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_bvh_ptr_chasing(ray, root):
    stack = [root]
    closest_t = infinity
    
    while stack is not empty:
        node = stack.pop()
        
        # Check if node is within ray's t_max
        if node.bbox.min_t > ray.t_max or node.bbox.max_t < ray.t_min:
            continue
            
        if node.is_leaf:
            # Process leaf primitives/volume
            hit = intersect_leaf_primitives(ray, node.data)
            if hit.t < closest_t:
                closest_t = hit.t
        else:
            # Explicit pointer chasing
            # Test left child
            if ray.intersects(node.left_child.bbox):
                stack.push(node.left_child)
            # Test right child
            if ray.intersects(node.right_child.bbox):
                stack.push(node.right_child)
                
    return closest_t
```

**Parameters:**
- Stack size: Depends on tree depth. Not given in the sources.
- Memory alignment: Not specified, but pointer chasing performance is sensitive to cache behavior [S1].

**Edge Cases and Pitfalls:**
- **Performance Degradation:** On GPU architectures, this method exhibits catastrophic performance degradation (over 100x slowdown) compared to compact layouts due to severe underutilization of available memory bandwidth [S1].
- **Memory Latency:** On ARM architectures, it may marginally outperform index-based layouts (like `pbrt-align16`) when the workload is memory latency bound, as it reduces memory-dependent operations in the hot loop [S1].

## Subtopics

### [PBRTv4 BVH Layout](22864-pbrtv4-bvh-layout.md)
A memory-efficient, cache-friendly representation that replaces explicit child pointers with implicit indexing and unified node structures, enabling more predictable memory access patterns during traversal. Choose this for GPU performance where memory bandwidth is the bottleneck.

## Operating characteristics
- **Time Complexity:** $O(\log N)$ on average for balanced trees, but dependent on memory access patterns.
- **Memory Complexity:** $O(N)$ nodes, but with higher overhead due to pointer storage and potential fragmentation.
- **Measured Performance:**
  - **ARM:** Marginally outperforms `pbrt-align16` when the workload is memory latency bound [S1].
  - **GPU:** Over 100x slower than compact layouts like `sg-eq` and `pbrt-q16` due to underutilized memory bandwidth [S1].
- **Strengths:** Reduces computational overhead (index arithmetic) in the traversal hot loop, beneficial on latency-bound CPUs [S1].
- **Limitations:** Poor memory bandwidth utilization on GPUs, leading to significant performance penalties [S1].
- **When to Use:** On CPU architectures where memory latency is the primary bottleneck and cache behavior favors pointer chasing.
- **When Not to Use:** On GPU architectures or any platform where memory bandwidth is the limiting factor.

## Background
The method is discussed in the context of comparing BVH layouts for volume rendering and ray tracing [S1]. It is presented as a baseline or alternative to compact, index-based layouts like `pbrt-align16`, `sg-eq`, and `pbrt-q16` [S1]. The performance characteristics are evaluated across ARM and GPU architectures, highlighting the architectural divergence in layout preferences [S1].

## Sources
[S1] Decoupling Data Layouts from Bounding Volume Hierarchies, https://arxiv.org/pdf/2511.15028, fetched 2026-10-29
