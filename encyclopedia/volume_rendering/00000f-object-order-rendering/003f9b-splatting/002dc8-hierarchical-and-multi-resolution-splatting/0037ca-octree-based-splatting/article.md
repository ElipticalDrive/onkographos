# Octree-based Splatting

Also called: Laur and Hanrahan Splatting

## Overview
Octree-based Splatting is an object-order interactive volume rendering technique that accelerates rendering by approximating volume contributions using an octree hierarchy [S2, book]. It solves the performance bottleneck of dense voxel splatting by culling large empty or smooth regions and rendering coarse approximations for dense areas [S1, unofficial]. Unlike standard splatting which processes every voxel, this method selects the appropriate level of detail based on a stored error metric, allowing for variable resolution rendering [S2, book].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume dataset; an error tolerance threshold $\epsilon$.
*   **Output:** A 2D image with accumulated color and opacity.

**Data Structures:**
The volume is organized into an octree. Each node in the octree stores:
1.  **Approximate Scalar Value ($v_{approx}$):** A representative scalar value for the region covered by the node [S2, book].
2.  **Error Value ($e$):** A metric indicating the deviation of the child nodes' scalar values from $v_{approx}$ [S2, book].
    *   *Note:* The exact formula for $e$ is not specified in the sources, but it represents the magnitude of variation within the subtree [S2, book].

**Algorithm Steps:**
1.  **Preprocessing (Octree Construction):**
    *   Construct the octree from the input volume.
    *   For each internal node, compute $v_{approx}$ and $e$ based on its children [S2, book].
2.  **Rendering (Traversal and Splatting):**
    *   Initialize the image buffer.
    *   Start traversal from the root node.
    *   For each node $N$:
        1.  Check if $N$ is a leaf or if its error value $e < \epsilon$ [S2, book].
        2.  **If condition met:**
            *   Render the region covered by $N$ as a single geometric primitive (splat) [S2, book].
            *   The size of the splat is determined by the spatial extent of the cell represented by $N$ in the octree [S1, unofficial].
            *   The color/opacity of the splat is derived from $v_{approx}$ [S2, book].
            *   Composite this splat into the image [S1, unofficial].
        3.  **If condition not met:**
            *   Recursively descend to the children of $N$ [S2, book].
    *   Ensure splats are composited in depth order (back-to-front) to handle transparency correctly [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_octree_splat(root_node, error_tolerance, image_buffer):
    """
    Recursively traverse the octree and splat nodes that meet the error tolerance.
    
    Args:
        root_node: The root of the octree.
        error_tolerance: The threshold epsilon for stopping descent.
        image_buffer: The target 2D image buffer for compositing.
    """
    traverse(root_node, error_tolerance, image_buffer)

def traverse(node, error_tolerance, image_buffer):
    """
    Traverse a single node of the octree.
    """
    # Check if we should stop descending
    # Sources state: "descend the octree only until a node with less than a given error tolerance is encountered"
    # This implies if node.error < tolerance, we render this node.
    if node.error < error_tolerance:
        # Render the region covered by this node as a splat
        # The size of the splat is based on the cell it represents in the octree
        splat_size = get_splat_size_for_node(node)
        splat_color = map_scalar_to_color(node.approximate_scalar_value)
        
        # Composite the splat into the image
        # Splatting requires depth-ordered compositing
        composite_splat(splat_size, splat_color, node.position, image_buffer)
    else:
        # Descend to children
        for child in node.children:
            traverse(child, error_tolerance, image_buffer)
```

**Parameters:**
*   **Error Tolerance ($\epsilon$):** A threshold value. Increasing $\epsilon$ decreases rendering time by allowing larger regions to be approximated at higher levels of the octree [S2, book].

**Edge Cases and Pitfalls:**
*   **Precision:** The error metric must be computed accurately during construction to ensure visual consistency [S2, book].
*   **Boundaries:** The transition between different levels of detail must be handled to avoid visible seams, though the sources do not specify a specific blending technique [S2, book].

## Subtopics

### Hierarchical Splatting
A broader category including octree-based methods. Octree-based splatting is a specific implementation using 3D spatial partitioning [1168-hierarchical-splatting.md].

### Multi-Resolution Points
Uses point clouds with LOD. Octree-based splatting uses voxel-derived primitives [4784-multi-resolution-points.md].

### 3D Gaussian Splatting LOD methods
Uses Gaussian primitives with covariance-based LOD. Octree-based splatting uses scalar-value-based error metrics [5439-3d-gaussian-splatting-lod-methods.md].

### Splat List (SVL) Generation with Multi-Level Downsampling
Precomputes splat lists. Octree-based splatting traverses the tree at render time [5767-splat-list-svl-generation-with-multi-level-downsampling.md].

## Operating characteristics
*   **Time Complexity:** Depends on the number of nodes that do not meet the error tolerance. In the worst case (high error tolerance or high detail), it approaches $O(N)$ where $N$ is the number of voxels. In the best case (smooth data, high tolerance), it approaches $O(1)$ per frame for large regions [S2, book].
*   **Memory Complexity:** $O(N)$ to store the octree, where each node stores an approximate value and an error value [S2, book].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Interactive rendering speeds; adaptive resolution based on data smoothness [S2, book].
*   **Limitations:** Quality depends on the error metric; potential artifacts at level-of-detail boundaries [S2, book].
*   **When to use:** Interactive exploration of large volume datasets where frame rate is critical [S2, book].
*   **When not to use:** When high-fidelity, error-free rendering is required for all regions [S2, book].

## Background
Octree-based Splatting was introduced by Laur and Hanrahan in 1991 [S2, book]. It is a foundational technique in hierarchical volume rendering, building on the concept of splatting [Shirley90, Wilhelms91] and combining it with octree data structures for efficient culling and approximation [S1, unofficial]. It is used in interactive visualization systems where real-time performance is necessary [S2, book].

## Sources
[S1] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
[S2] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
