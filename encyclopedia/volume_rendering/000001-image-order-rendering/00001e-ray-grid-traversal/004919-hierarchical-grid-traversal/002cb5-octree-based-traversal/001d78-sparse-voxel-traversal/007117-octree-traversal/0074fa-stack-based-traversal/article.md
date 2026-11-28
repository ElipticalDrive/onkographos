# Stack-based traversal

Also called: Iterative octree traversal, Explicit stack traversal

## Overview
Stack-based traversal is a method for navigating hierarchical spatial structures, such as Sparse Voxel Octrees (SVO) or Bounding Volume Hierarchies (BVH), by replacing recursive function calls with an explicit data structure (a stack) to manage the traversal state . It solves the problem of stack overflow in deep trees and improves cache locality by allowing linear memory access patterns for node fetching . Unlike recursive implementations, it explicitly controls visitation order by pushing children onto the stack based on their distance from the ray origin, ensuring closer nodes are processed first .

## Shared principles
The core mechanism involves maintaining a stack of traversal states, where each state typically contains the node index or pointer, and intersection parameters ($t_{min}, t_{max}$) relative to the ray .

1.  **Initialization**: The stack is initialized with the root node and the ray's entry/exit distances for the root bounding volume .
2.  **Loop**: While the stack is not empty:
    *   Pop the top state .
    *   Perform a ray-node intersection test (e.g., slab method for AABBs) .
    *   If the node is a leaf, process it (e.g., sample volume) and terminate or continue based on requirements .
    *   If the node is internal, compute intersection distances for its children .
    *   Push children onto the stack in reverse order of their distance from the ray origin (farthest first) so that the closest child is popped next .
3.  **Backtracking**: Implicitly handled by the stack; when a subtree is exhausted, the algorithm naturally returns to the parent state stored in the stack .

Key mathematical operations include the slab intersection test for axis-aligned bounding boxes:
$$ t_{min} = \max_i \left( \frac{boxMin_i - o_i}{d_i} \right), \quad t_{max} = \min_i \left( \frac{boxMax_i - o_i}{d_i} \right) $$
where $o$ is ray origin and $d$ is ray direction . Zero direction components must be handled by clamping $d_i$ to $\epsilon$ or using infinity for $t$ calculations .

## Subtopics

### [Hierarchical Traversal](896-hierarchical-traversal.md)
Uses a stack to step through a grid hierarchy, prioritizing closer nodes to skip empty regions.

### [Sparse Voxel Octree Ray Traversal](2451-sparse-voxel-octree-ray-traversal.md)
Explicitly manages a stack to control visitation order based on eye position, differing from state-machine approaches.

### [CPU Sparse Voxel Octree Traversal](2455-cpu-sparse-voxel-octree-traversal.md)
Uses an explicit stack on CPU to avoid recursion overhead and leverage cache-friendly memory access.

## Background
Stack-based traversal is a standard technique in ray tracing and volume rendering, originating from the need to handle deep recursion limits in hardware or constrained environments . It is widely used in CPU-based SVO renderers and GPU implementations where explicit stack management is required for dynamic branching .

## Sources
