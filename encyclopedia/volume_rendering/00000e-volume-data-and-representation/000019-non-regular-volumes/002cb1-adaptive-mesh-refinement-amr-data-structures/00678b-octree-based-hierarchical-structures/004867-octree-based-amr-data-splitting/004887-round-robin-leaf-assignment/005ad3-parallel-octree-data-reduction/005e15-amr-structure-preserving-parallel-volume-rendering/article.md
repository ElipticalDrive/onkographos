# AMR Structure-Preserving Parallel Volume Rendering


## Overview
A parallel volume rendering method that preserves the Adaptive Mesh Refinement (AMR) block structure, distributing leaves of the octree among processors and using raycasting with adaptive sampling [S1, unofficial]. It avoids resampling to a uniform grid, instead rendering individual blocks directly while buffering small ray-segments to reduce communication overhead [S1, unofficial].

## How it works
**Inputs:** AMR dataset organized as a hierarchy of grids; number of processors $P$ [S1, unofficial].
**Outputs:** Composite volume-rendered image [G].

**Data Structures:**
The AMR hierarchy is organized as a quadtree (2D) or octree (3D) [S1, unofficial].
*   **Inner Nodes:** Indicate regions requiring further refinement [S1, unofficial].
*   **Leaf Nodes:** Specify a grid block with resolution determined by the current hierarchy level [S1, unofficial].

**Steps:**
1.  **Tree Construction:** Organize the input AMR grids into the quadtree/octree structure [S1, unofficial].
2.  **Load Balancing:** Distribute individual blocks (leaf nodes of the octree) among the $P$ processors in a round-robin fashion to achieve static load balancing [S1, unofficial].
3.  **Communication Buffering:** Buffer small ray-segments resulting from the block structure into larger messages to decrease communication overhead [S1, unofficial].
4.  **Rendering:** Render individual blocks using raycasting [S1, unofficial]. Two sampling schemes are used: a simple approach with a fixed constant sample distance, and an adaptive approach that decreases sample distance in finer resolution blocks [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def amr_structure_preserving_rendering(amr_hierarchy, num_processors, adaptive=False, base_distance=1.0):
    # Step 1: Identify leaf blocks
    leaves = [node for node in amr_hierarchy.traverse() if node.is_leaf()]
    
    # Step 2: Round-robin distribution for static load balancing
    processor_blocks = [[] for _ in range(num_processors)]
    for i, leaf in enumerate(leaves):
        processor_blocks[i % num_processors].append(leaf)
    
    # Step 3 & 4: Render blocks, buffer segments, compose
    processor_images = []
    for blocks in processor_blocks:
        # Initialize local image buffer [G]
        width, height = 100, 100
        local_image = [[0.0, 0.0, 0.0, 0.0] for _ in range(width * height)]
        
        for block in blocks:
            # Determine sampling distance based on scheme
            sample_distance = base_distance / block.resolution_level if adaptive else base_distance
            
            # Raycast block
            block_image = raycast_block(block, sample_distance)
            
            # Compose block into local image
            for y in range(height):
                for x in range(width):
                    idx = y * width + x
                    c_b, a_b = block_image[idx][0:3], block_image[idx][3]
                    c_l, a_l = local_image[idx][0:3], local_image[idx][3]
                    contrib = (1.0 - a_l) * a_b
                    local_image[idx] = [
                        c_l[0] + contrib * c_b[0],
                        c_l[1] + contrib * c_b[1],
                        c_l[2] + contrib * c_b[2],
                        a_l + contrib
                    ]
        processor_images.append(local_image)
    
    # Composite processor images (e.g., binary-swap or direct accumulation)
    final_image = processor_images[0]
    for img in processor_images[1:]:
        final_image = compose_images(final_image, img)
    return final_image

def raycast_block(block, sample_distance):
    # Standard front-to-back volume raycasting [G]
    width, height = block.image_dimensions
    image = [[0.0, 0.0, 0.0, 0.0] for _ in range(width * height)]
    for y in range(height):
        for x in range(width):
            o = block.camera_origin(x, y)
            d = block.camera_direction(x, y)
            # Compute ray-box intersection [G]
            t_min, t_max = compute_ray_box_intersection(o, d, block.bounds)
            if t_max < t_min or t_max < 0:
                continue
            alpha = 0.0
            color = [0.0, 0.0, 0.0]
            t = t_min
            while t <= t_max:
                c, a = block.sample(t)
                contrib = (1.0 - alpha) * a
                color[0] += contrib * c[0]
                color[1] += contrib * c[1]
                color[2] += contrib * c[2]
                alpha += contrib
                if alpha >= 1.0:
                    break
                t += sample_distance
            image[y * width + x] = [color[0], color[1], color[2], alpha]
    return image

def compose_images(img1, img2):
    # Standard alpha compositing [G]
    width, height = len(img1[0]), len(img1)
    result = [[0.0, 0.0, 0.0, 0.0] for _ in range(width * height)]
    for y in range(height):
        for x in range(width):
            idx = y * width + x
            c1, a1 = img1[idx][0:3], img1[idx][3]
            c2, a2 = img2[idx][0:3], img2[idx][3]
            contrib = (1.0 - a1) * a2
            result[idx] = [
                c1[0] + contrib * c2[0],
                c1[1] + contrib * c2[1],
                c1[2] + contrib * c2[2],
                a1 + contrib
            ]
    return result
```

**Parameters:**
*   **Sampling Scheme:** 'simple' or 'adaptive' [S1, unofficial].
*   **Sample Distance:** Constant for simple; variable based on resolution for adaptive [S1, unofficial].

**Edge Cases:**
*   **Communication Overhead:** The block structure can lead to many small ray-segments. Buffering is required to mitigate overhead [S1, unofficial].

## Subtopics

### [VisIt Data Parallel Volume Rendering](17553-visit-data-parallel-volume-rendering.md)
A scheme in VisIt that treats AMR patches as atomic units and ignores ghost cells for correct sampling at boundaries. Choose this when using VisIt and needing explicit ghost-cell handling; otherwise, use the base method for general octree leaf distribution.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Preserves AMR structure; static load balancing via round-robin; reduced communication overhead via buffering [S1, unofficial].
*   **Limitations:** Static load balancing does not adapt to dynamic scene complexity [G].
*   **When to Use:** Interactive visualization of large AMR datasets where preserving block structure is beneficial and static load balancing is sufficient [S1, unofficial].
*   **When Not to Use:** Not given in the sources.

## Background
Described by Ma in 1999 using the PARAMESH framework [S1, unofficial]. Ma compared two approaches: resampling to a uniform grid (binary-swap composition) and preserving the AMR structure (the method described here) [S1, unofficial]. The structure-preserving method was developed to avoid the cost of resampling and to leverage the hierarchical nature of AMR data directly [S1, unofficial].

## Sources
[S1] Visualization Tools for Adaptive Mesh Refinement Data, https://digital.library.unt.edu/ark:/67531/metadc894061/, fetched 2026-10-26 (unofficial)
