# Octree-based Raycasting

Also called: Raycasting mit Octree-Beschleunigungsstruktur, Voxel-Raycasting mit Octree

## Overview
Octree-based Raycasting is a volume rendering technique that accelerates ray traversal by skipping empty space regions using an octree data structure [S1, unofficial]. It solves the performance bottleneck of naive voxel raycasting, which has $O(n)$ complexity relative to the number of voxels, by reducing traversal cost to depend only on occupied space [S1, unofficial]. It differs from its parent [Ray-Occlusion Traversal in Octree](5121-ray-occlusion-traversal-in-octree.md) by explicitly managing dynamic voxel updates and concurrency, and from [Bottom-up Neighbor-Finding Ray Traversal](5610-bottom-up-neighbor-finding-ray-traversal.md) by relying on hierarchical empty-space skipping rather than neighbor-based stepping.

## How it works
**Inputs:** Ray origin, ray direction, and an octree structure representing the voxel volume.
**Outputs:** Color and opacity values for the ray's path through the volume.

**Data Structures:**
The octree node structure must store:
1.  **Existence Flag:** Boolean indicating if the node is active.
2.  **Voxel Count:** An integer field storing the exact number of active voxels contained within this node and its descendants [S1, unofficial]. This count is critical for determining if a node is empty (count == 0) and should be skipped [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Construct the octree. For each node, calculate the total number of voxels in its subtree. Initialize all nodes as active [S1, unofficial].
2.  **Ray Traversal:**
    a. Intersect the ray with the root bounding box to find the entry point.
    b. Traverse the octree hierarchy. If a node's voxel count is 0, skip the entire subtree (empty space optimization) [S1, unofficial].
    c. If the node is a leaf, perform raycasting against the contained voxels (e.g., volume integration or surface intersection).
    d. If the node is an internal node, recursively traverse its children.
3.  **Dynamic Update (Voxel Modification):**
    a. When a voxel is added or removed, update the octree structure.
    b. Decrement the voxel count for all ancestor nodes of the modified voxel [S1, unofficial].
    c. If a node's voxel count reaches 0, deactivate the node (mark as empty) [S1, unofficial].
4.  **Concurrency Handling:**
    a. If multiple voxels are modified simultaneously, use atomic operations to prevent data races on the voxel count variables [S1, unofficial].
    b. Use `InterlockedAdd` for atomic increment/decrement of voxel counts [S1, unofficial].
    c. Declare the structured buffer containing node data as `globallycoherent` to ensure visibility across warps [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Node structure
class OctreeNode:
    is_active: bool
    voxel_count: int
    children: list[OctreeNode] # 8 children for octree
    voxels: list[Voxel] # Only for leaf nodes

# Atomic update for concurrent voxel modification
def atomic_voxel_update(node: OctreeNode, delta: int):
    # delta is +1 for add, -1 for remove
    # Requires globallycoherent buffer and InterlockedAdd intrinsic
    node.voxel_count = InterlockedAdd(node.voxel_count, delta)
    if node.voxel_count == 0:
        node.is_active = False

# Ray traversal
def raycast_octree(ray: Ray, node: OctreeNode):
    if not node.is_active:
        return EmptyResult # Skip empty subtree
    
    if node.is_leaf():
        return raycast_voxels(ray, node.voxels)
    
    # Traverse children
    results = []
    for child in node.children:
        if child.is_active:
            result = raycast_octree(ray, child)
            if result.hit:
                return result # Early exit on first hit or accumulate
            results.append(result)
    
    return combine_results(results)
```

**Parameters:**
*   **Voxel Count Threshold:** 0. Nodes with count 0 are skipped [S1, unofficial].
*   **Memory Qualifier:** `globallycoherent` for structured buffers in GPU shaders [S1, unofficial].

**Edge Cases:**
*   **Data Races:** Concurrent writes to the same node's voxel count without atomic operations lead to inconsistent data [S1, unofficial].
*   **Degenerate Input:** If a node becomes empty, it must be deactivated immediately to prevent wasted traversal [S1, unofficial].

## Subtopics

### Atomic Octree Update
A technique to avoid data races during concurrent voxel modifications by using atomic operations on node voxel counts [S1, unofficial]. It sets itself apart by enabling real-time updates without locking mechanisms, using `InterlockedAdd` and `globallycoherent` buffers [S1, unofficial]. Choose this when dynamic, high-frequency voxel updates are required on the GPU [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(n)$ for naive raycasting, reduced to $O(k)$ where $k$ is the number of occupied voxels/leaves traversed, due to empty-space skipping [S1, unofficial].
*   **Memory Complexity:** $O(N)$ where $N$ is the number of octree nodes, plus storage for voxel counts [S1, unofficial].
*   **Performance:** Significantly faster than naive raycasting for sparse volumes [S1, unofficial].
*   **Strengths:** Efficient empty-space skipping, supports dynamic updates [S1, unofficial].
*   **Limitations:** Requires maintaining voxel counts, which adds update overhead and complexity for concurrency [S1, unofficial].
*   **When to Use:** Sparse volumes with dynamic changes [S1, unofficial].
*   **When Not to Use:** Dense volumes where traversal overhead dominates, or static volumes where simpler structures suffice [S1, unofficial].

## Background
Originated in the context of raycasting-based voxel rendering, specifically addressing the need for acceleration structures to handle large voxel datasets [S1, unofficial]. Developed to overcome the $O(n)$ complexity of naive raycasting [S1, unofficial]. Used in real-time voxel rendering applications where dynamic updates are necessary [S1, unofficial].

## Sources
[S1] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
