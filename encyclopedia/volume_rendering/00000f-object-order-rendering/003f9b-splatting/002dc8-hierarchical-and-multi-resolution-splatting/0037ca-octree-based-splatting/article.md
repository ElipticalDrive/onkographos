# Octree-based Splatting

Also called: Laur and Hanrahan Splatting

## Overview
Octree-based Splatting is an object-order interactive volume rendering technique that approximates volume contributions using an octree representation to accelerate rendering based on a user-defined error tolerance [S2, book]. Introduced by Laur in 1991, it sets itself apart from standard splatting by dynamically adjusting the size of splats based on the octree cell they represent, thereby culling voxels and reducing the number of primitives rasterized [S1, unofficial]. It builds on [Hierarchical and Multi-Resolution Splatting](5618-hierarchical-and-multi-resolution-splatting.md) by specifically utilizing an octree structure for hierarchical decomposition and error-based traversal.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume dataset; an octree representation of the volume where each node stores an approximate scalar value and an error value; a global error tolerance threshold $\epsilon$.
*   **Output:** A 2D image with alpha compositing.

**Data Structures:**
The octree node structure consists of:
1.  **Approximate Scalar Value ($A$):** A single scalar value representing the region (e.g., average or mean of children).
2.  **Error Value ($E$):** A scalar value indicating the deviation of the child nodes' scalar values from the parent's approximate value [S2, book].
3.  **Children:** Pointers to up to 8 child octree nodes (or null if leaf).

**Algorithm Steps:**
1.  **Preprocessing (Octree Construction):**
    *   Construct the octree from the input volume.
    *   For each internal node, compute the approximate scalar value $A$ (e.g., mean of children).
    *   Compute the error value $E$ for the node. While the specific formula is not explicitly detailed in the provided sources, it represents the deviation of child scalar values from $A$ [S2, book].
    *   Store $A$ and $E$ in the parent node [S2, book].

2.  **Rendering (Traversal and Splatting):**
    *   Start traversal from the root of the octree.
    *   For the current node, check if its stored error value $E$ is less than the given error tolerance $\epsilon$ ($E < \epsilon$) [S2, book].
    *   **If $E < \epsilon$:**
        *   Stop descending into this node's children.
        *   Render the region represented by this node as a single geometric primitive (splat) [S2, book].
        *   The size of the splat is determined by the size of the octree cell (voxel) it represents [S1, unofficial].
        *   The color/opacity of the splat is derived from the node's approximate scalar value $A$ [S2, book].
    *   **If $E \ge \epsilon$:**
        *   Recursively traverse the children of the current node.
    *   **Rasterization:**
        *   The generated splats are rasterized using standard splatting techniques.
        *   Splatting is accelerated by using texture mapping hardware to render splats as textures mapped to polygons [S1, unofficial].
        *   Splatting involves compositing the sheets in depth order into the image with a matting operation [S1, unofficial].

**Parameters:**
*   **Error Tolerance ($\epsilon$):** A threshold value. Increasing $\epsilon$ decreases rendering time by allowing larger regions to be approximated at higher levels in the octree, thus reducing the number of splats [S2, book].

**Edge Cases and Pitfalls:**
*   **Precision:** The error value must accurately reflect the deviation to ensure visual fidelity. If the error metric is too coarse, artifacts may appear.
*   **Boundary Conditions:** The octree must cover the entire volume. Nodes at the boundary of the volume must be handled correctly during traversal.
*   **Degenerate Input:** If the volume has uniform scalar values, the error will be zero, and the entire volume may be represented by a single splat at the root, which is optimal.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_octree_splat(node, error_tolerance, camera):
    """
    Recursively traverse the octree and render splats.
    
    Args:
        node: The current octree node.
        error_tolerance: The threshold epsilon for stopping traversal.
        camera: Camera object for projection and depth sorting.
    """
    # Check if the node's error is within tolerance
    if node.error < error_tolerance:
        # Render the region as a single splat
        # The splat size is based on the cell size represented by this node
        splat = create_splat_from_node(node, camera)
        rasterize_splat(splat)
    else:
        # Descend into children
        for child in node.children:
            if child is not None:
                render_octree_splat(child, error_tolerance, camera)

def create_splat_from_node(node, camera):
    """
    Create a geometric primitive (splat) from an octree node.
    
    Args:
        node: The octree node.
        camera: Camera object for projection.
        
    Returns:
        A splat object with position, size, and color/opacity.
    """
    # Get the bounding box of the cell represented by the node
    bbox = node.get_bounding_box()
    
    # Project the center of the bbox to screen space
    center = bbox.center
    screen_pos = camera.project(center)
    
    # Determine splat size based on the cell size
    # The size is proportional to the dimensions of the octree cell
    cell_size = bbox.size
    splat_size = map_cell_size_to_splat_radius(cell_size, camera)
    
    # Use the approximate scalar value for color/opacity
    scalar_value = node.approximate_scalar_value
    color = map_scalar_to_color(scalar_value)
    opacity = map_scalar_to_opacity(scalar_value)
    
    return Splat(screen_pos, splat_size, color, opacity)
```

## Operating characteristics
*   **Time Complexity:** Rendering time is dependent on the number of voxels in the dataset and the chosen error tolerance [S2, book]. Higher error tolerances reduce the number of splats, thus decreasing rendering time [S2, book].
*   **Memory Complexity:** Requires storing the octree structure, including approximate scalar values and error values for each node [S2, book].
*   **Strengths:**
    *   Faster than ray-casting because reconstruction is performed for each sample point along the ray in ray-casting, whereas splatting uses precomputed primitives [S1, unofficial].
    *   Accelerated by texture mapping hardware [S1, unofficial].
    *   Allows for interactive rendering by adjusting error tolerance [S2, book].
*   **Limitations:**
    *   Image quality is affected by the size, shape, and type of the reconstruction kernel used [S1, unofficial].
    *   Reduced resolution versions of the data can be precomputed, but this requires additional preprocessing time and memory [S2, book].
*   **When to use:** When interactive rendering of large volume datasets is required, and visual fidelity can be traded for speed via error tolerance.
*   **When not to use:** When high-fidelity, artifact-free rendering is required without approximation, or when the volume has high-frequency details that require fine-grained representation.

## Background
Octree-based Splatting was introduced by Laur in 1991 [S2, book]. It is a method for accelerating volume rendering by using an octree representation to approximate volume contributions. The technique is part of the broader field of hierarchical and multi-resolution splatting. It builds on the concept of splatting, where volume data is projected onto the image plane as geometric primitives [S2, book]. The method was developed to address the computational cost of rendering large volume datasets interactively.

## Sources
[S1] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
[S2] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
