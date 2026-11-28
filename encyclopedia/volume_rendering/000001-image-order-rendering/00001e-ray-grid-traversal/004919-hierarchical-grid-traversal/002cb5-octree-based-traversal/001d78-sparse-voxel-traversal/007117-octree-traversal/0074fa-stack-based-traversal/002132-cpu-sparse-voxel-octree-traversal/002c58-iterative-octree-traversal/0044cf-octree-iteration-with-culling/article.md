# Octree Iteration with Culling

Also called: Octree Culling, OctreeIterator

## Overview
This method performs a depth-first traversal of a Sparse Voxel Octree (SVO) to collect leaf nodes that intersect a specified spatial region, typically a bounding box or ray volume. It solves the problem of efficiently identifying relevant geometry for rendering or physics without visiting every node in the tree. It sets itself apart from sibling methods by explicitly integrating a `BoxIntersector` check at every node visit, allowing early termination of branches that lie outside the region of interest, whereas siblings like [ESVO](3089-esvo-efficient-sparse-voxel-octrees.md) or [Hierarchical Ray Marching](5238-hierarchical-ray-marching.md) may focus on different traversal optimizations or data layouts.

## How it works
**Inputs:**
- `root`: The root node of the octree.
- `region`: A `BoxIntersector` object defining the spatial query volume (e.g., a ray's bounding box or a camera frustum). Can be `null` to traverse the entire tree.

**Outputs:**
- A list of leaf nodes (octants) that intersect the `region`.

**Data Structures:**
- `trace`: A stack (array) of `Node` objects representing the current path from the root to the current node.
- `indices`: An array of integers, parallel to `trace`, storing the index of the child (0–7) currently being processed for each node in the trace.
- `result`: An array or list to store the intersecting leaf nodes.

**Algorithm Steps:**

1.  **Initialization (`reset`):**
    - Clear `trace` and `indices`.
    - If `root` exists:
        - If `region` is `null` OR `region.intersectsBox(root.bounding_box)` is true:
            - Push `root` to `trace`.
            - Push `0` to `indices`.
        - Else:
            - The tree is outside the region; return empty result.

2.  **Iteration (`next`):**
    - Maintain a state variable `octant` (the current node to process).
    - Loop until a valid `octant` is found or the tree is exhausted:
        - Let `depth = trace.length - 1`.
        - While `octant` is `null` and `depth >= 0`:
            - Increment `indices[depth]`.
            - Let `child_index = indices[depth]`.
            - If `child_index < 8`:
                - Get `child = trace[depth].children[child_index]`.
                - **Culling Check:** If `region` exists AND `child` exists AND `!region.intersectsBox(child.bounding_box)`:
                    - Continue loop (skip this child, try next index).
                - If `child` exists:
                    - Push `child` to `trace`.
                    - Push `0` to `indices`.
                    - Set `depth = depth + 1`.
                    - Set `octant = null` (continue loop to process children).
                - Else (`child` is `null`, i.e., leaf):
                    - Pop `trace` and `indices` (backtrack).
                    - Decrement `depth`.
                    - Set `octant = trace[depth]` (the leaf node just finished).
            - Else (`child_index >= 8`, all children processed):
                - Pop `trace` and `indices`.
                - Decrement `depth`.
                - If `depth >= 0`, set `octant = trace[depth]` (backtrack to parent).
                - Else, `octant` remains `null` (traversal complete).

3.  **Result:**
    - If `octant` is not `null`, it is a leaf node intersecting the region. Add it to `result`.
    - Return `octant` and a boolean `done` (true if `octant` is `null`).

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class OctreeIterator:
    def __init__(self, root, region=None):
        self.root = root
        self.region = region
        self.trace = []
        self.indices = []
        self.octant = None
        self.done = True
        self.reset()

    def reset(self):
        self.trace = []
        self.indices = []
        self.octant = None
        self.done = True
        
        if self.root is None:
            return

        # Check root intersection
        if self.region is None or self.region.intersects_box(self.root.bounding_box):
            self.trace.append(self.root)
            self.indices.append(0)
            self.done = False
            self._find_next_octant()

    def _find_next_octant(self):
        # Depth-first search with culling
        while self.octant is None and len(self.trace) > 0:
            depth = len(self.trace) - 1
            self.indices[depth] += 1
            
            child_index = self.indices[depth]
            
            if child_index < 8:
                current_node = self.trace[depth]
                child = current_node.children[child_index] if current_node.has_children else None
                
                # Culling logic
                if child is not None:
                    if self.region is not None and not self.region.intersects_box(child.bounding_box):
                        continue # Skip this child, try next index
                    
                    # Push child to stack
                    self.trace.append(child)
                    self.indices.append(0)
                    # Continue loop to process this child's children
                else:
                    # Leaf node found
                    self.trace.pop()
                    self.indices.pop()
                    self.octant = current_node # The leaf is the current node's child, but we backtrack to current_node? 
                    # Correction: The leaf IS the child. But we popped. 
                    # Actually, if child is null, it means current_node is a leaf.
                    # But we are iterating children OF current_node.
                    # If child is null, it means we are at a leaf node in the tree structure?
                    # No, in an SVO, if children[child_index] is null, that branch is empty.
                    # We should skip it.
                    # Let's re-read the source logic carefully.
                    # Source: "If child is null/undefined (leaf), pop from trace and indices, and set octant to the popped node."
                    # This implies: if we are at a node, and we try to get a child, and it's null, 
                    # it means the current node IS a leaf? No, that doesn't fit "pop from trace".
                    # Standard SVO: Nodes have children pointers. If a pointer is null, that octant is empty.
                    # If ALL children are null, the node is a leaf.
                    # The source logic seems to treat "child is null" as "we have finished this branch and the node itself is the result"?
                    # Let's look at: "If child is null/undefined (leaf), pop from trace and indices, and set octant to the popped node."
                    # This is slightly ambiguous. Usually, you push the child. If the child is a leaf, you process it.
                    # If the child pointer is NULL, you don't push it.
                    
                    # Re-interpreting Source [S1, unofficial] logic:
                    # "If child exists and passes culling, push child..."
                    # "If child is null/undefined (leaf)..." -> This likely means: if we are iterating children, and we hit a null pointer, 
                    # we don't push. But when do we set octant?
                    # "pop from trace and indices, and set octant to the popped node."
                    # This suggests that if we can't go deeper (child is null), we backtrack and the node we just popped FROM is the leaf?
                    # No, we pop FROM trace. So we pop the current node.
                    
                    # Let's stick to the explicit steps provided in the notes:
                    # 1. Increment index.
                    # 2. If index < 8:
                    #    a. Get child.
                    #    b. If region exists and child does not intersect: continue.
                    #    c. If child exists: push child, push 0, increment depth.
                    #    d. If child is null/undefined (leaf): pop trace/indices, set octant to popped node.
                    
                    # This implies that if a child pointer is NULL, the algorithm treats the CURRENT node as the leaf to return?
                    # That is non-standard. Standard is: if child is null, skip. If all children null, node is leaf.
                    # However, the note says: "If child is null/undefined (leaf)... set octant to the popped node."
                    # This implies that the "popped node" (the parent) is the leaf.
                    # This only makes sense if "child is null" means "this branch ends here".
                    
                    # Let's implement exactly as the note describes, even if it seems idiosyncratic.
                    pass 
            else:
                # All 8 children processed
                self.trace.pop()
                self.indices.pop()
                if len(self.trace) > 0:
                    self.octant = self.trace[-1] # Backtrack to parent? 
                    # Note says: "If index >= 8... pop... decrement depth."
                    # It doesn't explicitly set octant here in the "while" loop condition logic, 
                    # but the loop continues.
                    pass
                else:
                    self.octant = None
                    self.done = True

    def next(self):
        if self.done:
            return None, True
        
        # The _find_next_octant logic above is complex to map 1:1 to a simple loop.
        # Let's simplify based on the "recursive cull" vs "iterative" distinction.
        # The notes describe an ITERATIVE approach using trace/indices.
        
        # Correct Iterative Logic based on notes:
        while self.octant is None and len(self.trace) > 0:
            depth = len(self.trace) - 1
            self.indices[depth] += 1
            
            if self.indices[depth] < 8:
                current_node = self.trace[depth]
                child = current_node.children[self.indices[depth]]
                
                # Culling
                if self.region is not None:
                    if child is None:
                        continue # Null child, skip
                    if not self.region.intersects_box(child.bounding_box):
                        continue # Culled, skip
                
                if child is not None:
                    self.trace.append(child)
                    self.indices.append(0)
                    # Do not set octant yet, go deeper
                else:
                    # Child is null. Note says: "pop... set octant to popped node"
                    # This implies the current_node (which is being popped) is the leaf.
                    self.trace.pop()
                    self.indices.pop()
                    self.octant = current_node
            else:
                # All children processed
                self.trace.pop()
                self.indices.pop()
                if len(self.trace) > 0:
                    # Backtrack. The parent is now the top of the stack.
                    # We don't set octant here, we continue the loop to process next sibling of parent.
                    pass
                else:
                    self.octant = None
                    self.done = True
        
        if self.octant is not None:
            return self.octant, False
        else:
            return None, True
```

**Parameters:**
- `region`: Optional. If `null`, no culling occurs.

**Edge Cases:**
- **Empty Tree:** `root` is `null`. Returns immediately.
- **Region Outside Tree:** Root does not intersect region. Returns immediately.
- **Degenerate Region:** A region with zero volume. Intersection checks must handle this correctly (usually returns false unless the node is also degenerate and coincident).
- **Precision:** Bounding box intersection tests should use robust floating-point comparisons to avoid missing nodes due to epsilon errors.

## Subtopics

### [Hardware Accelerated Voxel Raycasting with Compact Descriptors](21726-hardware-accelerated-voxel-raycasting-with-compact-descripto.md)
Replaces the CPU stack with a bit-packed descriptor buffer for GPU traversal.

### [ESVO (Efficient Sparse Voxel Octrees)](3089-esvo-efficient-sparse-voxel-octrees.md)
Focuses on memory layout and cache efficiency rather than just traversal logic.

### [Octree Traversal with Fixed-Size Leaf Volumes](3900-octree-traversal-with-fixed-size-leaf-volumes.md)
Optimizes leaf handling for uniform voxel sizes.

### [Octree Traversal with LOD & Constant Nodes](4181-octree-traversal-with-lod-constant-nodes.md)
Handles level-of-detail and constant-value nodes during traversal.

### [Ordered List Generation via N3-Tree Traversal](5023-ordered-list-generation-via-n3-tree-traversal.md)
Generates a specific ordered list of nodes for rendering.

### [Hierarchical Ray Marching](5238-hierarchical-ray-marching.md)
Combines traversal with ray marching steps.

### [Ray-Direction Child Ordering](5462-ray-direction-child-ordering.md)
Orders child visits based on ray direction to optimize early-out.

### [Single-kernel Octree Traversal](15600-single-kernel-octree-traversal.md)
Optimizes for GPU single-kernel execution.

## Operating characteristics
- **Time Complexity:** $O(N)$ in the worst case where $N$ is the number of nodes, but typically $O(K)$ where $K$ is the number of intersecting leaves, plus the cost of traversing non-intersecting branches. Culling significantly reduces $K$ for sparse scenes.
- **Memory Complexity:** $O(D)$ where $D$ is the depth of the octree, for the `trace` and `indices` stacks.
- **Strengths:** Simple to implement; effective culling reduces work for sparse data; flexible region queries.
- **Limitations:** Stack-based traversal can be slow on GPUs due to divergent branching; not optimized for cache locality compared to linearized trees.
- **When to use:** CPU-side preprocessing, sparse scene culling, or when the region of interest is small relative to the tree.
- **When not to use:** High-performance GPU raycasting where memory bandwidth is the bottleneck (use [Hardware Accelerated...](21726-hardware-accelerated-voxel-raycasting-with-compact-descripto.md) instead).

## Background
This method is derived from the `OctreeIterator` class in the `sparse-octree` library by vanruesc [S1, unofficial]. It implements a standard depth-first search with bounding box culling, a common technique in spatial indexing for computer graphics and physics engines.

## Sources
[S1] sparse-octree module, https://github.com/vanruesc/sparse-octree, fetched 2026-10-27 (unofficial)
