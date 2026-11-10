# Adaptive Volume Ray Casting

Also called: volume ray casting technique

## Overview
Adaptive Volume Ray Casting is a volume rendering technique where one ray is shot through the volume for each pixel, traversing an octree to accumulate physical values along the ray path [S1, unofficial]. It builds on [Hierarchical Trilinear Ray Casting](851-hierarchical-trilinear-ray-casting.md) by using the octree structure to determine traversal depth based on camera-defined resolution limits. It produces better image quality than splatting by avoiding splatting artifacts, but is more computationally intensive [S1, unofficial].

## How it works
**Inputs:**
*   Ray origin and direction for each pixel.
*   Octree data structure representing the volume.
*   Camera parameters defining the maximum resolution level.

**Outputs:**
*   Accumulated pixel value for each ray.

**Data Structures:**
*   **Octree:** Hierarchical structure where nodes represent spatial subdivisions.
*   **Leaf Definition:** Every cell ranked with the maximum level of resolution defined by the camera is considered as a leaf [S1, unofficial]. Highest level of resolution cells are discarded when these cells are too small compared to the image pixel size. The maximum octree depth level read is therefore defined by the camera.

**Steps:**
1.  **Find Intersections:** Determine the intersections between the grid and the ray. Using basic 3D geometry calculation, compute the coordinates $(x,y,z)$ of the entry and exit points of the ray in the cell faces [S1, unofficial].
2.  **Traverse Octree:** Start from the root of the octree and traverse down the tree. This top-down traversal of the tree stops when a leaf cell is found [S1, unofficial].
3.  **Accumulate Contribution:** Multiply the cell's observed physical value by the length between the entry and exit points into the cell faces. Add every contribution along the ray to the resulting pixel value, without taking into account any ordering [S1, unofficial].
4.  **Post-Processing (Optional):** Apply an adaptive Gaussian blur filter as an image post-processing option to avoid aliasing in coarse mesh regions by adjusting filter size to local image resolution [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_volume_ray_casting(ray_origin, ray_direction, octree_root, camera_max_level):
    """
    Renders a single pixel by traversing an octree.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray.
        octree_root: Node, root of the octree.
        camera_max_level: int, maximum octree depth level defined by camera.
        
    Returns:
        float, accumulated pixel value.
    """
    # Step 1: Find intersections with the grid
    # Compute coordinates (x,y,z) of entry and exit points of the ray in the cell faces
    # using basic 3D geometry calculation.
    entry_point, exit_point = compute_ray_grid_intersections(ray_origin, ray_direction)
    
    # Initialize accumulator
    accumulated_value = 0.0
    
    # Step 2: Traverse Octree
    # Start from root and go down until a leaf cell is found.
    # Note: The sources describe a simple traversal stopping at the first leaf.
    # In a full implementation, this would involve checking if the current node
    # is a leaf or if it needs refinement based on camera_max_level.
    current_node = octree_root
    while not is_leaf(current_node, camera_max_level):
        # Determine which child to visit based on ray position
        # This step is implied by "traverse down the tree" but details of
        # child selection are not given in the sources.
        current_node = get_child_node(current_node, ray_origin, ray_direction)
        
    # Step 3: Accumulate Contribution
    # Multiply the cell's observed physical value by the length between
    # the entry and exit points into this cell faces.
    # Note: The sources state "without taking into account any ordering".
    # This implies simple addition, potentially ignoring depth sorting.
    cell_value = get_physical_value(current_node)
    cell_length = compute_distance(entry_point, exit_point) # Simplified: assumes single cell traversal for this step
    accumulated_value += cell_value * cell_length
    
    # Step 4: Post-Processing (Optional)
    # Apply adaptive Gaussian blur filter to avoid aliasing in coarse mesh regions.
    # This is typically done after all rays are processed.
    # accumulated_value = apply_adaptive_gaussian_blur(accumulated_value, local_resolution)
    
    return accumulated_value

def is_leaf(node, max_level):
    """
    Check if a node is considered a leaf based on camera max level.
    Every cell ranked with the maximum level of resolution defined by the camera
    is considered as a leaf.
    """
    return node.level >= max_level

def compute_ray_grid_intersections(origin, direction):
    """
    Compute entry and exit points of the ray in the cell faces.
    Uses basic 3D geometry calculation.
    """
    # Implementation details for basic 3D geometry are not provided in sources.
    # Placeholder for actual intersection logic.
    return origin, origin + direction

def get_child_node(node, origin, direction):
    """
    Determine which child node to visit next.
    """
    # Implementation details for child selection are not provided in sources.
    return node.children[0] # Placeholder

def get_physical_value(node):
    """
    Get the physical value observed in the cell.
    """
    return node.value

def compute_distance(p1, p2):
    """
    Compute Euclidean distance between two points.
    """
    return sqrt((p2.x - p1.x)**2 + (p2.y - p1.y)**2 + (p2.z - p1.z)**2)
```

**Parameters:**
*   **Camera Max Level:** The maximum octree depth level read is defined by the camera. Cells smaller than the image pixel size at this level are discarded [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Aliasing:** Aliasing can be seen on coarse mesh resolution areas because low simulation resolution leads to big pixels [S1, unofficial]. An adaptive Gaussian blur filter can be used to mitigate this [S1, unofficial].
*   **Ordering:** Contributions are added without taking into account any ordering, which may affect transparency/opacity calculations if not handled by the physical value definition [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not explicitly given, but described as "more computationally intensive" than splatting [S1, unofficial].
*   **Memory Complexity:** Not explicitly given.
*   **Performance:** The software implementation is not interactive on large datasets (e.g., Milky Way simulation with ~200 million AMR cells) [S1, unofficial].
*   **Strengths:** Produces better image quality than splatting (no splatting artifacts) [S1, unofficial]. Convenient to process large datasets as the algorithm is embarrassingly parallel [S1, unofficial].
*   **Limitations:** More computationally intensive than splatting [S1, unofficial]. Aliasing in coarse mesh regions [S1, unofficial].
*   **When to use:** When image quality is prioritized over interactive performance and splatting artifacts are unacceptable [S1, unofficial].
*   **When not to use:** When interactive performance on large datasets is required [S1, unofficial].

## Background
*   **Origin:** Based on volume ray casting explained by Levoy (1990) [S1, unofficial].
*   **History:** Described in the context of AMR (Adaptive Mesh Refinement) simulations, specifically for visualizing large datasets like Milky Way simulations [S1, unofficial].
*   **Usage:** Used in PyMSES software for volume rendering of AMR simulations [S1, unofficial].

## Sources
[S1] Volume Rendering of AMR Simulations, https://doi.org/10.48550/arxiv.1210.8014, fetched 2026-10-26 (unofficial)
