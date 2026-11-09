# Octree-based Splatting

Also called: Laur and Hanrahan Splatting

## Overview
Octree-based Splatting is an object-order interactive volume rendering technique that approximates volume contributions using an octree representation to balance rendering speed and image quality [S2, book]. It solves the performance bottleneck of dense voxel rendering by culling voxels and aggregating data, allowing the renderer to descend the octree only until a node's error tolerance is met [S1, unofficial][S2, book]. Unlike standard splatting which processes every voxel, this method adapts the resolution of the splats based on the spatial hierarchy, rendering larger geometric primitives for coarse, uniform regions and finer primitives for detailed areas [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D scalar volume dataset; an octree data structure representing the volume; an error tolerance threshold $\epsilon$.
*   **Output:** A 2D image composed of rasterized splats.

**Data Structures**
The core data structure is an octree node. Each node contains:
1.  **Approximate Scalar Value ($v_{approx}$):** A single scalar value representing the region covered by the node and its descendants.
2.  **Error Value ($e$):** A scalar value indicating the deviation of the child nodes' scalar values from $v_{approx}$ [S2, book].
3.  **Spatial Extent:** Implicitly defined by the node's position in the octree hierarchy (level and coordinates), determining the physical size of the region it represents.

**Algorithm Steps**
1.  **Octree Construction:** Build an octree from the input volume. For each parent node, compute $v_{approx}$ (e.g., mean of children) and $e$ (e.g., RMS deviation of children from $v_{approx}$) [S2, book].
2.  **Traversal:** Start at the root node.
3.  **Error Check:** At the current node, compare the stored error value $e$ against the user-defined error tolerance $\epsilon$.
4.  **Decision:**
    *   If $e < \epsilon$: Stop descending. The node's region is approximated by a single splat.
    *   If $e \ge \epsilon$: Descend to all eight children and repeat Step 3 for each.
5.  **Splat Generation:** For each node where the traversal stopped, generate a geometric primitive (splat). The size of the splat is determined by the spatial extent of the octree cell it represents [S1, unofficial]. Larger cells (higher in the tree) result in larger splats.
6.  **Rasterization and Compositing:** Rasterize the generated splats into the image buffer. Composite them in depth order (back-to-front) using a matting operation to handle opacity and blending [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_octree_splatting(root_node, error_tolerance, camera):
    """
    Renders a volume using octree-based splatting.
    
    Args:
        root_node: The root of the precomputed octree.
        error_tolerance: The threshold epsilon for stopping traversal.
        camera: Object containing view matrix and projection parameters.
        
    Returns:
        Image buffer.
    """
    splats = []
    traverse(root_node, error_tolerance, splats)
    
    # Sort splats by depth relative to camera for correct compositing
    splats.sort(key=lambda s: s.depth_from_camera)
    
    image = initialize_image_buffer()
    for splat in splats:
        rasterize_splat(splat, camera, image)
        
    return image

def traverse(node, error_tolerance, splat_list):
    """
    Recursively traverses the octree to collect splats.
    """
    # Check error tolerance condition
    if node.error_value < error_tolerance:
        # Create a splat for this node
        # The size of the splat corresponds to the cell size at this octree level
        splat = create_splat_from_node(node)
        splat_list.append(splat)
    else:
        # Descend to children
        for child in node.children:
            traverse(child, error_tolerance, splat_list)

def create_splat_from_node(node):
    """
    Constructs a geometric primitive (splat) from an octree node.
    The size is scaled based on the node's level in the octree.
    """
    # Determine the physical size of the voxel/cell represented by this node
    cell_size = get_cell_size_for_level(node.level)
    
    # Create a polygon or Gaussian kernel with size proportional to cell_size
    # Position is the center of the cell
    return Splat(
        position=node.center,
        size=cell_size,
        value=node.approximate_scalar_value
    )
```

**Parameters**
*   **Error Tolerance ($\epsilon$):** A threshold value for the error stored in octree nodes. Increasing this value decreases rendering time by allowing larger regions to be approximated at higher levels in the octree [S2, book].

**Edge Cases and Pitfalls**
*   **Precision:** The error value must be computed consistently during octree construction. If the error metric does not accurately reflect visual significance, artifacts may appear at the boundaries of different resolution levels.
*   **Boundary Artifacts:** Discontinuities may occur where nodes of different sizes meet if the splat blending or matting operation is not handled carefully.

## Operating characteristics
*   **Time Complexity:** The rendering time depends on the number of nodes visited, which is determined by the error tolerance. In the worst case (high detail, low tolerance), it approaches $O(N)$ where $N$ is the number of voxels. In the best case (uniform data, high tolerance), it approaches $O(1)$ or $O(\sqrt[3]{N})$ depending on the hierarchy depth. Generally, it reduces the number of draw calls from $O(N)$ to a smaller number proportional to the number of leaf nodes in the truncated octree [S2, book].
*   **Memory Complexity:** Requires storing the octree structure. Each node stores an approximate value and an error value. This overhead is generally lower than storing all voxels if the volume is sparse or highly coherent.
*   **Strengths:** Significantly faster than ray-casting and standard splatting for large datasets. Interactive frame rates are achievable by adjusting the error tolerance [S2, book].
*   **Limitations:** Image quality is dependent on the error tolerance setting. High tolerance leads to blurring or loss of detail. The precomputation of the octree adds an initial cost.
*   **When to use:** Interactive visualization of large volume datasets where frame rate is critical and some loss of fine detail is acceptable.
*   **When not to use:** When maximum visual fidelity is required regardless of performance, or when the data is highly heterogeneous with no coherent regions to approximate.

## Background
Octree-based Splatting was introduced by Laur and Hanrahan in 1991 [S2, book]. It was developed to address the limitations of texture-mapping approaches and standard splatting, which were bottlenecked by the number of polygons or voxels processed. By using an octree to represent the volume and culling voxels based on error tolerance, it enabled faster rendering speeds by reducing the number of texture-mapped polygons or splats used to represent the volume [S2, book]. This technique is essentially equivalent to reducing the number of samples taken along the ray in an image-order ray casting method, but in an object-order framework [S2, book].

## Sources
[S1] DTIC ADA606342: An Anti-Aliasing Technique for Splatting, https://archive.org/details/DTIC_ADA606342, fetched 2026-10-26 (unofficial)
[S2] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
