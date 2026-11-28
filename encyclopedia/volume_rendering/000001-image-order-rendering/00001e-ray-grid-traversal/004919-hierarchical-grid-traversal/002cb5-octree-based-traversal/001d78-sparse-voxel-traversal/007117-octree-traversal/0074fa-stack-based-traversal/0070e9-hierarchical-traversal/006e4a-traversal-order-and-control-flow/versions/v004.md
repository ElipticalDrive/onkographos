# Traversal Order and Control Flow


## Overview
Traversal Order and Control Flow defines the algorithmic strategies for navigating spatial hierarchies (such as octrees or BVHs) during volume rendering, determining the sequence in which nodes are visited and how transitions between cells are managed. It solves the problem of computational overhead in deep hierarchies by optimizing the descent and ascent paths, reducing redundant intersection tests, and enabling parallel optimizations like tile-sharing. It sets itself apart from sibling methods by focusing on the *control logic* of the traversal (e.g., two-pass decoupling, iterative up-down transitions) rather than the specific spatial data structure or the shading model itself.

## Shared principles
The family operates on a rooted tree $\mathcal{H}$ where each node $N$ has a bounding volume (typically an AABB), pointers to children (if internal), and potentially a pointer to its parent. The core mechanism involves determining the next node to visit based on the ray $R(t) = \mathbf{o} + t\mathbf{d}$ and the current node's position in the hierarchy.

Key shared concepts include:
1.  **Intersection Testing:** Standard ray-AABB intersection tests are used to determine if a ray enters a node's volume.
2.  **Transition Classification:** Transitions between cells are classified based on whether they remain within the bounds of a common ancestor (internal) or leave it (external).
3.  **Depth Awareness:** Many methods require knowledge of the tree's depth or the depth of the current node relative to the leaves to make traversal decisions.
4.  **State Maintenance:** The traversal maintains state such as the current node, entry/exit distances, and potentially a stack or buffer of nodes to process.

## Subtopics

### [Two-Pass Hierarchy Traversal](3339-two-pass-hierarchy-traversal.md)
Two-Pass Hierarchy Traversal is an optimization that reduces computational cost by splitting the descent into a coarse first pass and a detailed second pass. It solves the problem of redundant intersection tests in deep hierarchies by extracting a coarse subtree in the first step and restarting traversal from that subtree in the second step. It sets itself apart from siblings like [Iterative Up-Down Tree Traversal](21243-iterative-up-down-tree-traversal.md) by explicitly decoupling the coarse culling phase from the fine traversal phase to enable tile-sharing optimizations for single scattering. See [Two-Pass Hierarchy Traversal](3339-two-pass-hierarchy-traversal.md) for algorithm details.

### [Iterative Up-Down Tree Traversal](21243-iterative-up-down-tree-traversal.md)
Iterative Up-Down Tree Traversal is a hierarchical grid traversal algorithm for octrees that avoids restarting the search from the root node when transitioning between cells. It leverages the current node's position awareness to determine if the next cell is a sibling or child, and if not, it ascends the tree until an internal transition is found, then descends to the leaf. Unlike the standard top-down method which starts the search from the root node for every transition, this method uses local tree structure knowledge to minimize traversal distance, avoiding the overhead of root-to-leaf searches for non-internal transitions. See [Iterative Up-Down Tree Traversal](21243-iterative-up-down-tree-traversal.md) for algorithm details.

### BFS Hierarchy Cut Algorithm
An algorithm to cut a binary tree hierarchy along a certain condition using a breadth-first search (BFS) queue [S1, unofficial]. The procedure initializes a queue with the root of the hierarchy $\mathcal{H}$. It dequeues nodes one by one; if a node satisfies the condition, it is added to a cut set. Otherwise, its children are enqueued. This allows for efficient pruning or modification of the hierarchy based on specific criteria during training or rendering setup [S1, unofficial].

## Background
The specific variants cited here, such as Two-Pass Hierarchy Traversal and Iterative Up-Down Tree Traversal, are discussed in the context of large-scale reconstruction and rendering, particularly in relation to 3D Gaussian Splatting and hierarchical memory management [S1, unofficial]. The BFS Hierarchy Cut algorithm is presented in the context of unified training and rendering for ultra-large scale reconstruction with external memory [S1, unofficial]. These methods address the scalability challenges of rendering massive scenes by optimizing the traversal overhead associated with spatial hierarchies.

## Sources
[S1] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
