# Sparse Voxel Octree Memory Management

Also called: Flat Array SVO Memory Management, Free List Octree Allocation

## Overview
This method manages Sparse Voxel Octree (SVO) memory using a flat array and free lists for dynamic allocation and deallocation of branch nodes and attributes [S1, unofficial]. It enables efficient memory reuse during frequent node creation and destruction in dynamic octrees [S1, unofficial]. The flat array reuses contiguous memory blocks, while the CPU Memory Layout variant uses pointer-free linear memory pools indexed by unique cell indices [S1, unofficial][S2, unofficial].

## How it works
The method operates on two primary data structures: a contiguous data array for node storage and free lists for tracking available memory slots [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A request to allocate or free a branch node or attribute [S1, unofficial].
*   **Output:** A valid memory pointer (or index) to the allocated resource, or a deallocated state [S1, unofficial].

**Data Structures:**
1.  **Data Array:** A contiguous block of memory storing branch nodes and attributes [S1, unofficial].
2.  **Free Structure Memory List:** A list of available pointers/indices for branch nodes [S1, unofficial].
3.  **Free Attribute Memory List:** A list of available pointers/indices for attribute data [S1, unofficial].

**Steps:**

1.  **Allocate Branch Node:** Check the Free Structure Memory List [S1, unofficial]. If the list is empty, append a new block to the end of the Data Array [S1, unofficial]. Otherwise, pop and reuse the last free pointer [S1, unofficial]. Record updates for the branch pointer and the last element of the branch (typically `ptr + 7` for an 8-child node) to maintain structural integrity [S1, unofficial].
2.  **Free Branch Node:** Add the node's pointer to the Free Structure Memory List [S1, unofficial]. Recursively free all children of this branch node [S1, unofficial].
3.  **Free Attribute:** Add the attribute's pointer to the Free Attribute Memory List [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class SVOAllocator:
    def __init__(self):
        self.data_array = []  # Contiguous memory for nodes/attributes
        self.free_branches = []  # Stack of available branch pointers
        self.free_attributes = []  # Stack of available attribute pointers

    def allocate_branch(self) -> int:
        if self.free_branches:
            ptr = self.free_branches.pop()
        else:
            ptr = len(self.data_array)
            self.data_array.append({'children': [None] * 8, 'attributes': None})
        # Record updates for branch pointer and last element (ptr + 7)
        self._update_branch_structure(ptr)
        return ptr

    def free_branch(self, ptr: int):
        node = self.data_array[ptr]
        for child_ptr in node['children']:
            if child_ptr is not None:
                self.free_branch(child_ptr)
        self.free_branches.append(ptr)

    def allocate_attribute(self) -> int:
        if self.free_attributes:
            return self.free_attributes.pop()
        ptr = len(self.data_array)
        self.data_array.append({'data': None})
        return ptr

    def free_attribute(self, ptr: int):
        self.free_attributes.append(ptr)

    def _update_branch_structure(self, ptr: int):
        # Record updates for the branch pointer and the last element of the branch (ptr + 7)
        if ptr + 7 < len(self.data_array):
            self.data_array[ptr + 7]['parent'] = ptr
```

**Parameters and Edge Cases:**
*   **Branch Size:** Typically 8 children per node, implied by the `ptr + 7` offset [S1, unofficial].
*   **Precision:** Not given in the sources.
*   **Boundaries:** The data array handles dynamic resizing; if the free list is a stack, allocation is $O(1)$ [G].
*   **Degenerate Input:** Not given in the sources.

## Subtopics

### CPU Memory Layout
A pointer-free variant using linear memory pools indexed by unique cell indices $I$, supporting dynamic refinement and coarsening via linked-list based allocation [S2, unofficial]. When a cell is refined, eight children are always created [S2, unofficial]. Field pools store properties like color, density, and temporary variables separately, packing frequently accessed pairs (e.g., color and alpha) into a single pool per cell [S2, unofficial]. The allocation unit is fixed at 8 cells, and freed units return to the beginning of the pool [S2, unofficial]. A uniform root array at the pool's beginning cannot be freed [S2, unofficial]. A separate flag pool stores cell depth and bit field flags [S2, unofficial]. The tree is defined by parents and children pools storing indices of parents and the first child; the remaining seven children are consecutively numbered and need not be stored explicitly [S2, unofficial]. Use this variant when dynamic refinement/coarsening is frequent and cache locality for attribute access is critical [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales well with dynamic scenes due to efficient memory reuse [S1, unofficial]. The CPU Memory Layout variant may suffer from poor ray-casting performance as tree depth increases, since neighbor traversal in the primal tree graph $G_p$ can become distant [S2, unofficial].
*   **Strengths:** Low memory overhead, no pointer chasing, efficient reuse of memory blocks [S1, unofficial].
*   **Limitations:** Potential fragmentation in flat arrays [G]. Recursive freeing can be costly for deep subtrees [S1, unofficial]. The CPU Memory Layout variant requires careful pool management and may have poor performance for deep trees without additional neighbor graphs [S2, unofficial].
*   **When to Use:** Suitable for applications requiring frequent dynamic updates to the SVO structure, such as real-time volumetric painting or adaptive mesh refinement [S2, unofficial].
*   **When Not to Use:** Not ideal for static scenes where a simpler, pre-allocated structure would suffice [G]. Avoid if deep tree traversal performance is critical without additional optimization (e.g., neighbor graphs) [S2, unofficial].

## Background
The flat array and free list approach is described in the context of Unity-based implementations [S1, unofficial]. The CPU Memory Layout variant originates from the CanvoX paper for high-resolution VR painting in large volumetric canvases [S2, unofficial].

## Sources
[S1] xandergos/unity-sparse-voxel-octrees, https://github.com/xandergos/unity-sparse-voxel-octrees, fetched 2026-10-26 (unofficial)
[S2] CanvoX: High-resolution VR Painting in Large Volumetric Canvas, https://archive.org/details/arxiv-1704.02724, fetched 2026-10-29 (unofficial)
