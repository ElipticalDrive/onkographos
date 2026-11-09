# Distributed Data Volume Rendering via Shear-Warp Factorisation

Also called: S-W factorisation, distributed shear-warp

## Overview
Distributed Data Volume Rendering via Shear-Warp Factorisation is a parallel rendering technique that divides a large 3D volume into sub-volumes, renders them independently using the shear-warp algorithm, and composites the results using the associative 'over' operator to produce an image identical to a single-pass renderer [S1]. It distinguishes itself from its sibling, the Spatially Partitioned Shear-Warp Method, by explicitly defining a hierarchical rendering tree with configurable branching factors and specific optimizations for network transfer (window-encoding, RLE) and memory storage (multi-order copies) to handle large datasets on distributed clusters [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume data set, a camera configuration (viewpoint, projection type, image plane), and a rendering tree configuration (branching factor $b$, depth $n$) [S1].
*   **Output:** A 2D composited image on the final image plane [S1].

**Data Structures:**
*   **Rendering Tree:** A hierarchical structure of nodes containing compositors (branch nodes) and renderers (leaf nodes) [S1].
*   **Renderer Nodes (Leaves):** Apply shear, perspective scaling, the 'over' operator, and warping to generate a projected image [S1].
*   **Compositor Nodes (Branches):** Combine $b$ images from children using the 'over' operator with back-to-front ordering [S1].
*   **Head Compositor:** The root node that produces the final image and sends it to the user interface [S1].
*   **Sub-volumes:** Convex, adjacent partitions of the original volume, always the same size, sharing at least one plane of voxels to ensure correct edge reproduction [S1].
*   **Data Storage:** Each rendering node stores three alternately-ordered copies of the sub-volume data to avoid expensive real-time re-ordering [S1].

**Parameters:**
*   **Branching Factor ($b$):** The number of children for each compositor node, tuned based on processor speed and network bandwidth [S1].
*   **Tree Depth ($n$):** The number of levels in the tree. An $n$-level tree contains $b^{n-1}$ renderers and $\sum_{i=0}^{n-2} b^i$ compositors [S1].
*   **Dynamic Range:** Input floating-point data is mapped to 16-bit (65,536 levels) or 8-bit (256 levels) integers to save memory and cycles, as 32-bit float range ($10^{38}$) is rarely required for 24-bit displays [S1].

**Algorithm Steps:**

1.  **Volume Division:** The head compositor divides the entire data volume into $b$ pieces. If a child is a compositor, it further divides its sub-volume along its longest axis into $b$ pieces for its children. This iterative division produces convex, adjacent sub-volumes ensuring a balanced tree and correct back-to-front ordering [S1].
2.  **Parameter Propagation:** The viewing angle, image plane, and camera parameters are passed down the tree to all renderer nodes [S1].
3.  **Rendering (Leaf Nodes):** Each renderer node applies the shear-warp factorisation to its assigned sub-volume: apply appropriate shear to the sub-volume data, apply perspective scaling if required, use the 'over' operator to traverse the sheared volume and generate a projected volume-rendered image, and warp the projected image into the required image plane [S1]. Optimization: Compute the bounding box of non-blank pixels by projecting the sub-volume corners into the image plane to improve transfer speed [S1].
4.  **Compositing (Branch Nodes):** Rendered images are sent up the tree. Each compositor node receives $b$ images from its children. Using the ordering information derived from the sub-volume positions, composite the $b$ images using the 'over' operator from back to front [S1]. Optimization (Minimal Compositing): Use the window-encoding information to composite only the non-blank image sections, avoiding decoding full-size images [S1].
5.  **Final Output:** The head compositor produces the final composited image. Optimization (RLE): Run-Length Encode the final image before sending it to the user interface to save network bandwidth on slow connections [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distributed_volume_render(volume, camera, tree_config):
    """
    volume: 3D list of voxels (list of list of list)
    camera: dict with 'viewpoint', 'projection', 'image_plane', 'width', 'height'
    tree_config: dict with 'b' (branching factor) and 'n' (depth)
    """
    b = tree_config['b']
    n = tree_config['n']
    
    # 1. Volume Division
    sub_volumes = divide_volume(volume, b, depth=1, max_depth=n)
    
    # 2. Rendering Phase (Parallel on Leaf Nodes)
    rendered_images = []
    windows = []
    
    for sub_vol in sub_volumes:
        # Store three alternately-ordered copies to avoid real-time re-ordering
        copy1 = sub_vol
        copy2 = transpose_axes(sub_vol, axes=(1, 0, 2))
        copy3 = transpose_axes(sub_vol, axes=(2, 1, 0))
        
        # Select optimal storage order based on camera viewing angle [G]
        optimal_data = select_optimal_storage(copy1, copy2, copy3, camera)
        
        # Apply Shear-Warp: shear, project with 'over', warp
        sheared_data = shear_data(optimal_data, camera)
        projected_image = shear_warp_project(sheared_data, camera)
        final_image = warp_image(projected_image, camera)
        
        # Window-Encoding: Compute bounding box of non-blank pixels
        window = compute_bounding_box(final_image)
        
        rendered_images.append(final_image)
        windows.append(window)
        
    # 3. Compositing Phase (Up the Tree)
    # Composite back-to-front using ordering information
    composited_image = rendered_images[0]
    for i in range(1, len(rendered_images)):
        composited_image = over_operator(rendered_images[i], composited_image)
        
    # 4. Output
    output_image = run_length_encode(composited_image)
    return output_image

def divide_volume(vol, b, depth, max_depth):
    """Divides volume into b pieces along the longest axis.
    Sub-volumes share at least one plane of voxels [S1]."""
    if depth >= max_depth:
        return [vol]
    
    # Find longest axis using len() to avoid AttributeError on lists
    dims = [len(vol), len(vol[0]), len(vol[0][0])]
    axis = dims.index(max(dims))
    
    # Split into b pieces, sharing one plane
    step = len(vol) // b
    pieces = []
    for i in range(b):
        start = i * step
        end = (i + 1) * step + 1 if i < b - 1 else (i + 1) * step
        pieces.append(slice_volume(vol, axis, start, end))
        
    # Recursively divide if children are compositors
    result = []
    for piece in pieces:
        result.extend(divide_volume(piece, b, depth + 1, max_depth))
    return result

def slice_volume(vol, axis, start, end):
    if axis == 0: return vol[start:end]
    elif axis == 1: return [row[start:end] for row in vol]
    else: return [[[voxel[start:end] for voxel in row] for row in plane] for plane in vol]

def transpose_axes(vol, axes):
    # Standard 3D list transpose
    return [[[vol[i][j][k] for k in range(len(vol[0][0]))] for j in range(len(vol[0]))] for i in range(len(vol))]

def select_optimal_storage(c1, c2, c3, camera):
    # [G] Selects copy based on viewing direction to align with shear plane
    return c1

def shear_data(data, camera):
    # [G] Reorders data so viewing rays are parallel to a fundamental axis
    return data

def shear_warp_project(data, camera):
    # [G] Traverses sheared volume and applies 'over' operator
    return [[0.0 for _ in range(camera['width'])] for _ in range(camera['height'])]

def warp_image(image, camera):
    # [G] Maps projected image to final image plane coordinates
    return image

def compute_bounding_box(image):
    # [G] Standard bounding box calculation for non-blank pixels
    return (0, 0, len(image[0]), len(image))

def over_operator(front, back):
    # Standard alpha compositing operator
    return [[front[i][j] + back[i][j] * (1 - front[i][j]) for j in range(len(front[0]))] for i in range(len(front))]

def run_length_encode(image):
    # Replaces runs of repeated values with value and count [S1]
    encoded = []
    for row in image:
        if not row: continue
        current = row[0]
        count = 1
        for val in row[1:]:
            if val == current:
                count += 1
            else:
                encoded.append((current, count))
                current = val
                count = 1
        encoded.append((current, count))
    return encoded
```

## Operating characteristics
*   **Time Complexity:** Depends on the number of voxels, image resolution, and tree depth. The shear-warp algorithm reduces transcendental calculations by aligning rays with data axes [S1].
*   **Memory Complexity:** Triples memory requirements on rendering nodes due to storing three alternately-ordered copies of the sub-volume data [S1]. Dynamic range compression reduces per-voxel memory from 4 bytes (32-bit float) to 2 bytes (16-bit int) or 1 byte (8-bit int) [S1].
*   **Network Bandwidth:** Window-encoding and minimal compositing significantly reduce data transfer between nodes by only sending and compositing non-blank sub-images [S1]. RLE reduces bandwidth between the head compositor and the user interface [S1].
*   **Strengths:** Produces an image identical to a single-pass renderer due to the associativity of the 'over' operator [S1]. Avoids expensive real-time data re-ordering by pre-storing multiple data orders [S1]. Scalable via configurable branching factor and tree depth [S1].
*   **Limitations:** Requires sufficient memory to store three copies of sub-volumes [S1]. Requires careful tuning of $b$ and $n$ for specific hardware and network configurations [S1]. Edge values require shared voxel planes, increasing memory slightly [S1].
*   **When to Use:** Large volumes that cannot fit in a single node's memory or require interactive frame rates on a cluster. Suitable when network bandwidth between cluster nodes is fast, but bandwidth to the user workstation may be slower [S1].
*   **When Not to Use:** Not given in the sources.

## Background
The shear-warp factorisation was first applied to volume rendering by Lacroute & Levoy (1994) [S1]. The distributed implementation described here is christened 'dvr' (distributed volume renderer) [S1]. It builds on the concept of the rendering tree, similar to VFleet, but differs in that VFleet requires all nodes to have access to all data, whereas this method partitions the data [S1]. The system was designed for visualisation of large astronomical cubes [S1].

## Sources
[S1] "A Distributed-Data Implementation of the Perspective Shear-Warp Volume Rendering Algorithm for Visualisation of Large Astronomical Cubes", http://web.eng.ucsd.edu/~jschulze/projects/Beeson2003.pdf, fetched 2026-10-29
