# Octree Traversal with Fixed-Size Leaf Volumes

Also called: Fixed-Size Leaf Octree Traversal, 32x32x32 Leaf Traversal

## Overview
This method accelerates sparse voxel octree (SVO) raycasting by enforcing a uniform leaf node size, thereby decoupling traversal depth from data sparsity and ensuring predictable memory access patterns. Unlike [Iterative Octree Traversal](2618-iterative-octree-traversal.md), which may vary leaf granularity, this approach fixes the leaf volume to 32x32x32 voxels, optimizing for hardware cache efficiency and reducing branch divergence in GPU warps, albeit at the cost of increased false-positive checks in noisy or dense regions.

## How it works
The algorithm traverses an octree to find the first non-empty leaf node intersected by a ray. It uses a fixed leaf size to simplify the termination condition and optimize the inner loop.

**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized).
- Octree root node.
- Volume side length $s$ (assumed cubic, $s \times s \times s$).
- Fixed leaf size $L = 32$ voxels.

**Outputs:**
- Intersection point or null if no leaf is hit.

**Data Structures:**
- **Skip List (`skipListPerDepth`):** An array of bitmasks, one per depth level. Each bit $i$ indicates that the $i$-th child of a node at that depth has already been visited or determined empty for the current ray, preventing redundant checks.
- **Node State:** Each node has a state indicating if it is a leaf or internal.
- **Current Depth Variable (`currentDepthVariable`):** Tracks the depth of the current node in the traversal.

**Parameters:**
- Leaf size $L = 32$.
- Volume side $s$.

**Steps:**
1. **Initialization:**
   - Set `currentDepthVariable` to 0.
   - Initialize `skipListPerDepth` for all possible depths to 0.
   - Start traversal at the root node.

2. **Traversal Loop:**
   - While the current node is not a leaf:
     - Determine the child index $i$ (0-7) that the ray enters next. This is typically calculated using the ray's direction and the node's bounding box.
     - Check if the bit $i$ is set in `skipListPerDepth[currentDepthVariable]`.
       - If set, the child has been processed; skip to the next sibling or backtrack.
       - If not set, proceed to the child.
     - **Update Skip List:** Mark the current child as visited by setting bit $i$ in the skip list:
       $$ \text{skipListPerDepth}[currentDepthVariable] \ |= \ (1 \ll i) $$
       [S1, unofficial]
     - Save the index and position of the current parent node for potential backtracking.
     - Increment `currentDepthVariable` to move one level deeper.
     - Move to the selected child node.

3. **Leaf Processing:**
   - Once a leaf node is reached, examine the 32x32x32 voxel volume contained within it.
   - Perform ray-voxel intersection tests within this fixed-size block to find the first occupied voxel.
   - If an intersection is found, return the result.
   - If no intersection is found in this leaf, backtrack to the parent and continue searching other siblings.

**Formulas:**
- **Diagonal Leaf Count:** The number of leaf nodes along the main diagonal of the volume is derived from the Pythagorean theorem:
  $$ tcountNodes = \frac{s}{64} \cdot 3 \cdot \left(\frac{s}{32}\right)^2 $$
  [S1, unofficial]
  Note: The factor $\frac{s}{64}$ and $3$ likely relate to the specific geometric derivation of the diagonal path through the grid of leaves, where each leaf is $32 \times 32 \times 32$. The term $(\frac{s}{32})^2$ represents the number of leaves along one axis squared, scaled by the diagonal factor.

- **Worst-Case Complexity:**
  $$ tTotalCount = tcountNodes \cdot \log s + 3 \cdot 32^2 $$
  [S1, unofficial]
  Asymptotically, this simplifies to $O(s \cdot \log s)$.
  [S1, unofficial]

**Pitfalls:**
- **Noise Sensitivity:** Noise in volume data causes false positives, forcing the algorithm to examine unnecessary leaf nodes, significantly reducing efficiency.
  [S1, unofficial]
- **Precision Errors:** Unpredictable precision errors may cause rays to miss voxels or return incorrect results, often requiring post-processing (e.g., median filtering).
  [S1, unofficial]

## Subtopics

### Iterative Octree Traversal (BFS)
This variant uses Breadth-First Search (BFS) to traverse the octree, extracting plane parameters from leaf nodes into a `Planes` structure for each voxel. It initializes an empty dictionary and a queue with the root node. For each voxel, it iterates through the octree, popping nodes from the queue. If a node is a leaf (`octo_state_ == 0`), it appends center, normal, distance, and complex attributes to a temporary `Planes` object. If not a leaf, it adds its non-None children to the queue. The resulting tensors are moved to CUDA. This method is distinct in its use of BFS and explicit plane parameter extraction, suitable for Gaussian Splatting applications.
[S2, unofficial]

## Operating characteristics
- **Time Complexity:**
  - Best-case: $O(\log s)$ per voxel, where $s$ is the volume side length. This occurs when the ray hits a leaf quickly.
    [S1, unofficial]
  - Worst-case: $O(s \cdot \log s)$. This happens when every 32x32x32 leaf node along the diagonal (through the volume center) must be examined.
    [S1, unofficial]
- **Memory Complexity:** Depends on the octree structure. Fixed leaf sizes reduce memory overhead compared to variable-sized leaves for sparse data, but larger leaf sizes increase the probability of false positives.
  [S1, unofficial]
- **Performance:**
  - Warp divergence is minimal because similar rays produce similar results within a warp. Divergence is further mitigated by the relatively low number of iterations for noise-free data.
    [S1, unofficial]
  - Larger leaf sizes improve efficiency for sparse data but increase false positives. Smaller leaf sizes reduce false positives but increase octree memory usage.
    [S1, unofficial]
- **Strengths:** Predictable performance, reduced branch divergence, optimized for fixed-size hardware caches.
- **Limitations:** Sensitive to noise, potential for false positives in dense regions, fixed leaf size may not be optimal for all data distributions.

## Background
This method is described in a thesis on raycasting-based voxel rendering, focusing on recursive-free traversal of octrees. It emphasizes complexity analysis and divergence analysis for GPU implementation.
[S1, unofficial]

## Sources
[S1] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S2] Yang-SyZng/AVM-GS, https://github.com/Yang-SyZng/AVM-GS, fetched 2026-10-27 (unofficial)
