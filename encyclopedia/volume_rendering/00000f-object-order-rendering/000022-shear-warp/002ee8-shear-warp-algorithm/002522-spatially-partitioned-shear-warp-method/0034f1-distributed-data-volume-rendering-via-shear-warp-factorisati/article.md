# Distributed Data Volume Rendering via Shear-Warp Factorisation

Also called: S-W factorisation, distributed shear-warp

## Overview
Distributed Data Volume Rendering via Shear-Warp Factorisation is a parallel rendering technique that divides a large 3D volume into sub-volumes, renders them independently using the shear-warp algorithm, and composites the results using the associative 'over' operator to produce an image identical to a single-pass renderer [S1]. It distinguishes itself from its sibling, the Spatially Partitioned Shear-Warp Method, by explicitly defining a hierarchical rendering tree with configurable branching factors and specific optimizations for network transfer (window-encoding, RLE) and memory storage (multi-order copies) to handle large datasets on distributed clusters [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume data set (array of voxels), a camera configuration (viewpoint, projection type, image plane), and a rendering tree configuration (branching factor $b$, depth $n$).
*   **Output:** A 2D composited image on the final image plane.

**Data Structures:**
*   **Rendering Tree:** A hierarchical structure of nodes.
    *   **Renderer Nodes (Leaves):** Perform shear-warp projection on a sub-volume.
    *   **Compositor Nodes (Branches):** Composite images from child nodes using the 'over' operator.
    *   **Head Compositor:** The root node of the tree, which sends the final image to the user interface.
*   **Sub-volumes:** Convex, adjacent partitions of the original volume. To ensure correct edge reproduction, sub-volumes are always the same size and share at least one plane of voxels [S1].
*   **Data Storage:** To avoid real-time re-ordering, each rendering node stores three alternately-ordered copies of its sub-volume data (optimized for the three orthogonal shear directions) [S1].

**Parameters:**
*   **Branching Factor ($b$):** The number of children for each compositor node. Tuned based on processor speed and network bandwidth [S1].
*   **Tree Depth ($n$):** The number of levels in the tree. An $n$-level tree contains $b^{n-1}$ renderers and $\sum_{i=0}^{n-2} b^i$ compositors [S1].
*   **Dynamic Range:** Input floating-point data is mapped to 16-bit (65,536 levels) or 8-bit (256 levels) integers to save memory and cycles, as 32-bit float range ($10^{38}$) is rarely required for 24-bit displays [S1].

**Algorithm Steps:**

1.  **Volume Division:**
    The head compositor divides the entire data volume into $b$ pieces. If a child is a compositor, it further divides its sub-volume along its longest axis into $b$ pieces for its children. This iterative division produces convex, adjacent sub-volumes ensuring a balanced tree and correct back-to-front ordering [S1].

2.  **Parameter Propagation:**
    The viewing angle, image plane, and camera parameters are passed down the tree to all renderer nodes [S1].

3.  **Rendering (Leaf Nodes):**
    Each renderer node applies the shear-warp factorisation to its assigned sub-volume:
    *   Apply appropriate shear to the sub-volume data.
    *   Apply perspective scaling if required.
    *   Use the 'over' operator to traverse the sheared volume and generate a projected volume-rendered image.
    *   Warp the projected image into the required image plane [S1].
    *   *Optimization:* Compute the bounding box of non-blank pixels by projecting the sub-volume corners into the image plane. This defines the "window" for the image [S1].

4.  **Compositing (Branch Nodes):**
    Rendered images are sent up the tree. Each compositor node receives $b$ images from its children.
    *   Using the ordering information derived from the sub-volume positions, composite the $b$ images using the 'over' operator from back to front [S1].
    *   *Optimization (Minimal Compositing):* Use the window-encoding information to composite only the non-blank image sections, avoiding decoding full-size images [S1].

5.  **Final Output:**
    The head compositor produces the final composited image.
    *   *Optimization (RLE):* Run-Length Encode the final image before sending it to the user interface to save network bandwidth on slow connections (e.g., standard Ethernet) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distributed_volume_render(volume, camera, tree_config):
    """
    volume: 3D array of voxels
    camera: object containing viewpoint, projection, image_plane
    tree_config: dict with 'b' (branching factor) and 'n' (depth)
    """
    b = tree_config['b']
    n = tree_config['n']
    
    # 1. Volume Division
    # The head compositor divides the volume into b pieces.
    # Recursive division continues down the tree.
    sub_volumes = divide_volume(volume, b)
    
    # 2. Parameter Propagation
    # Camera parameters are passed to all leaf nodes.
    
    # 3. Rendering Phase (Parallel on Leaf Nodes)
    rendered_images = []
    windows = []
    
    for sub_vol in sub_volumes:
        # Store three alternately-ordered copies of sub_vol to avoid re-ordering
        data_copy_1 = reorder(sub_vol, order=1)
        data_copy_2 = reorder(sub_vol, order=2)
        data_copy_3 = reorder(sub_vol, order=3)
        
        # Select optimal storage order based on camera viewing angle
        optimal_data = select_optimal_storage(data_copy_1, data_copy_2, data_copy_3, camera)
        
        # Apply Shear-Warp
        # Shear the volume so viewing rays are parallel to a fundamental axis
        sheared_data = shear(optimal_data, camera)
        
        # Project using 'over' operator
        projected_image = shear_warp_project(sheared_data, camera)
        
        # Warp to image plane
        final_image = warp(projected_image, camera)
        
        # Window-Encoding: Compute bounding box of non-blank pixels
        window = compute_bounding_box(final_image)
        
        rendered_images.append(final_image)
        windows.append(window)
        
    # 4. Compositing Phase (Up the Tree)
    # Images are composited back-to-front based on sub-volume position
    # For a simple 2-level tree (b renderers, 1 head compositor):
    composited_image = rendered_images[0]
    for i in range(1, len(rendered_images)):
        # Ensure correct back-to-front order
        if is_behind(rendered_images[i], composited_image):
            composited_image = over_operator(rendered_images[i], composited_image)
        else:
            composited_image = over_operator(composited_image, rendered_images[i])
            
    # 5. Output
    # Run-Length Encode for network transfer
    output_image = run_length_encode(composited_image)
    return output_image

def divide_volume(vol, b):
    """
    Iteratively divide volume into b pieces.
    If children are compositors, they divide along the longest axis.
    Sub-volumes share at least one plane of voxels.
    """
    # Implementation details for iterative division along longest axis
    # Not given in the sources.
    pass

def select_optimal_storage(copy1, copy2, copy3, camera):
    """
    Selects the pre-ordered copy that aligns with the shear direction
    required by the camera angle.
    """
    # Implementation details for angle-to-storage mapping
    # Not given in the sources.
    pass

def shear_warp_project(data, camera):
    """
    Traverses sheared volume and applies 'over' operator.
    """
    # Standard Shear-Warp projection logic
    # Not given in the sources.
    pass

def over_operator(front, back):
    """
    Standard alpha compositing operator.
    """
    # Standard 'over' operator logic
    # Not given in the sources.
    pass

def compute_bounding_box(image):
    """
    Projects volume corners to find min/max x,y of non-blank pixels.
    """
    # Implementation details
    # Not given in the sources.
    pass

def run_length_encode(image):
    """
    Replaces runs of repeated values with value and count.
    Example: 'kavababababyt' -> 'kavab+3yt'
    """
    # Standard RLE logic
    # Not given in the sources.
    pass
```

## Operating characteristics
*   **Time Complexity:** Depends on the number of voxels, image resolution, and tree depth. The shear-warp algorithm reduces transcendental calculations by aligning rays with data axes [S1].
*   **Memory Complexity:** Triples memory requirements on rendering nodes due to storing three alternately-ordered copies of the sub-volume data [S1]. Dynamic range compression reduces per-voxel memory from 4 bytes (32-bit float) to 2 bytes (16-bit int) or 1 byte (8-bit int) [S1].
*   **Network Bandwidth:** Window-encoding and minimal compositing significantly reduce data transfer between nodes by only sending/compositing non-blank sub-images [S1]. RLE reduces bandwidth between the head compositor and the user interface [S1].
*   **Strengths:**
    *   Produces an image identical to a single-pass renderer due to the associativity of the 'over' operator [S1].
    *   Avoids expensive real-time data re-ordering by pre-storing multiple data orders [S1].
    *   Scalable via configurable branching factor and tree depth [S1].
*   **Limitations:**
    *   Requires sufficient memory to store three copies of sub-volumes [S1].
    *   Requires careful tuning of $b$ and $n$ for specific hardware/network configurations [S1].
    *   Edge values require shared voxel planes, increasing memory slightly [S1].
*   **When to Use:** Large volumes that cannot fit in a single node's memory or require interactive frame rates on a cluster. Suitable when network bandwidth between cluster nodes is fast, but bandwidth to the user workstation may be slower [S1].
*   **When Not to Use:** Small volumes where single-pass rendering is sufficient, or when memory is extremely constrained (triple storage is prohibitive) [S1].

## Background
The shear-warp factorisation was first applied to volume rendering by Lacroute & Levoy (1994) [S1]. The distributed implementation described here is christened 'dvr' (distributed volume renderer) [S1]. It builds on the concept of the rendering tree, similar to VFleet, but differs in that VFleet requires all nodes to have access to all data, whereas this method partitions the data [S1]. The system was designed for visualisation of large astronomical cubes [S1].

## Sources
[S1] "A Distributed-Data Implementation of the Perspective Shear-Warp Volume Rendering Algorithm for Visualisation of Large Astronomical Cubes", http://web.eng.ucsd.edu/~jschulze/projects/Beeson2003.pdf, fetched 2026-10-29
