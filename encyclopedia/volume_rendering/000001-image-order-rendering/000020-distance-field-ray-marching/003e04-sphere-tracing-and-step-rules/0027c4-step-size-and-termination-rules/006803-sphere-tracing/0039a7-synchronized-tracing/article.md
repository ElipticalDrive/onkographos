# Synchronized-tracing

Also called: Synchronized Tracing of Primitive-based Implicit Volumes

## Overview
Synchronized-tracing is a tile-based rendering pipeline for implicit volumes (blobtrees) that synchronizes threads within a subfrustum to allow coherent field evaluation, using sparse bottom-up tree traversal to prune the blobtree on-the-fly [S1, unofficial][S2, abstract only, unofficial]. It is designed for interactive modeling scenarios where no preprocessing is required when primitive parameters are updated [S1, unofficial][S2, abstract only, unofficial]. It sets itself apart from standard sphere tracing by introducing a synchronization mechanism via a low-resolution A-buffer to manage thread coherence within workgroups, and by using compact, smooth CSG operators to compute tight bounding volumes for approximate signed distance fields [S1, unofficial][S2, abstract only, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A blobtree representing implicit surfaces/volumes, consisting of thousands of primitives [S1, unofficial][S2, abstract only, unofficial]. The blobtree is defined by primitive parameters that may be updated dynamically [S1, unofficial][S2, abstract only, unofficial].
*   **Output:** Rendered image of the implicit surface/volume.

**Data Structures:**
*   **Blobtree:** A hierarchical structure of implicit primitives combined using CSG operators [S1, unofficial][S2, abstract only, unofficial].
*   **Low-resolution A-buffer:** Stores the primitives of interest for a given screen tile [S1, unofficial][S2, abstract only, unofficial]. This structure is used to synchronize threads within a subfrustum [S1, unofficial][S2, abstract only, unofficial].
*   **Tight Bounding Volume:** Computed for all primitives of the blobtree [S1, unofficial][S2, abstract only, unofficial].

**Parameters:**
*   **Lipschitz Bound:** For approximate signed distance fields, the Lipschitz bound should be close to 1 [S1, unofficial].
*   **Tile Size:** Not given in the sources.
*   **Subfrustum Size:** Not given in the sources.

**Steps:**
1.  **Compute Tight Bounding Volume:** For all primitives of the blobtree, compute a tight augmented bounding volume (or volume of interest) using compact, smooth CSG operators extended from standard bounded operators [S1, unofficial][S2, abstract only, unofficial]. This is particularly relevant when using approximate signed distance fields where the Lipschitz bound is close to 1 [S1, unofficial].
2.  **Tile-based Processing:** Divide the screen into tiles. For each tile, identify the primitives of interest and store them in the low-resolution A-buffer [S1, unofficial][S2, abstract only, unofficial].
3.  **Synchronization:** Use the low-resolution A-buffer during ray processing to synchronize threads within a subfrustum [S1, unofficial][S2, abstract only, unofficial]. This synchronization allows for coherent field evaluation within workgroups [S1, unofficial][S2, abstract only, unofficial].
4.  **Sparse Bottom-up Tree Traversal:** Perform a sparse bottom-up tree traversal on the blobtree to prune it on-the-fly [S1, unofficial][S2, abstract only, unofficial]. This step decorrelates field evaluation complexity from the full blobtree size [S1, unofficial][S2, abstract only, unofficial].
5.  **Ray Processing:** Execute ray processing using the sphere tracing algorithm [S1, unofficial][S2, abstract only, unofficial]. The sphere tracing algorithm uses the signed distance field values to determine step sizes [Parent].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def synchronized_tracing(blobtree, screen_width, screen_height, tile_size):
    # Initialize low-resolution A-buffer for each tile
    a_buffers = [[None] * (screen_width // tile_size) for _ in range(screen_height // tile_size)]
    
    # For each tile in the screen
    for ty in range(screen_height // tile_size):
        for tx in range(screen_width // tile_size):
            tile_bbox = [tx * tile_size, ty * tile_size, (tx + 1) * tile_size, (ty + 1) * tile_size]
            
            # Step 1: Compute tight bounding volume for all primitives
            # Using compact, smooth CSG operators extended from standard bounded operators
            # [S1, unofficial] Sources do not specify exact CSG operator formulas
            tight_bounding_volume = compute_csg_bounding_volume(blobtree)
            
            # Step 2: Populate A-buffer with primitives of interest
            primitives_of_interest = []
            for primitive in blobtree.primitives:
                if intersects(primitive, tight_bounding_volume) and intersects(primitive, tile_bbox):
                    primitives_of_interest.append(primitive)
            a_buffers[ty][tx] = primitives_of_interest
            
            # Step 3: Synchronize threads within the subfrustum using the A-buffer
            # [G] Standard GPU barrier synchronization for workgroup coherence
            synchronize_threads()
            
            # Step 4: Sparse bottom-up tree traversal to prune the blobtree
            pruned_blobtree = sparse_bottom_up_traversal(blobtree, primitives_of_interest)
            
            # Step 5: Execute ray processing using sphere tracing
            render_tile(tile_bbox, pruned_blobtree)

def compute_csg_bounding_volume(blobtree):
    # [S1, unofficial] Compact, smooth CSG operators extended from standard bounded operators
    # [G] Standard bounding volume computation for implicit primitives
    return blobtree.compute_global_bounding_box()

def intersects(primitive, bbox):
    # [G] Standard axis-aligned bounding box intersection test
    return primitive.min_x <= bbox[2] and primitive.max_x >= bbox[0] and \
           primitive.min_y <= bbox[3] and primitive.max_y >= bbox[1]

def synchronize_threads():
    # [G] GPU thread barrier (e.g., __syncthreads() in CUDA)
    pass

def sparse_bottom_up_traversal(blobtree, primitives):
    # [S1, unofficial] Prunes the blobtree on-the-fly to decorrelate complexity
    # [G] Standard tree traversal filtering out nodes outside the primitive set
    return blobtree.filter(primitives)

def render_tile(tile_bbox, pruned_blobtree):
    # Standard sphere tracing algorithm [Parent]
    # [G] Implementation details for ray generation and SDF evaluation
    for y in range(int(tile_bbox[1]), int(tile_bbox[3])):
        for x in range(int(tile_bbox[0]), int(tile_bbox[2])):
            origin = [float(x), float(y), -5.0]
            direction = [0.0, 0.0, 1.0]
            t = 0.0
            max_steps = 1000
            eps = 1e-4
            for _ in range(max_steps):
                p = [origin[i] + t * direction[i] for i in range(3)]
                dist = pruned_blobtree.sdf(p)
                if dist < eps:
                    return (x, y, t)
                t += max(eps, dist)
    return None
```

**Edge Cases and Pitfalls:**
*   **Approximate SDFs:** When using approximate signed distance fields, the Lipschitz bound should be close to 1 to ensure the tight bounding volume computation is accurate [S1, unofficial].
*   **Dynamic Updates:** The pipeline is designed to handle updates to primitive parameters without preprocessing, so dynamic updates must be handled efficiently [S1, unofficial][S2, abstract only, unofficial].
*   **Scalability:** The pipeline scales well to surfaces consisting of thousands of primitives, but performance scaling beyond this range is not specified [S1, unofficial][S2, abstract only, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The pipeline scales well to surfaces consisting of thousands of primitives [S1, unofficial][S2, abstract only, unofficial].
*   **Strengths:**
    *   No preprocessing required when primitive parameters are updated, making it suitable for interactive modeling [S1, unofficial][S2, abstract only, unofficial].
    *   Synchronization via A-buffer allows coherent field evaluation within workgroups [S1, unofficial][S2, abstract only, unofficial].
    *   Sparse bottom-up tree traversal decorrelates field evaluation complexity from the full blobtree size [S1, unofficial][S2, abstract only, unofficial].
*   **Limitations:**
    *   Relies on approximate signed distance fields with Lipschitz bound close to 1 to compute tight augmented bounding volumes using compact, smooth CSG operators [S1, unofficial].
    *   Performance scaling beyond thousands of primitives is not specified [S1, unofficial][S2, abstract only, unofficial].
*   **When to Use:** Interactive modeling scenarios involving implicit surfaces represented by blobtrees with thousands of primitives [S1, unofficial][S2, abstract only, unofficial].
*   **When Not to Use:** Not given in the sources.

## Background
Synchronized-tracing is a tile-based rendering pipeline for implicit surfaces represented by blobtrees [S2, abstract only, unofficial]. It was developed to address the need for efficient rendering in interactive modeling scenarios where no preprocessing is required when primitive parameters are updated [S1, unofficial][S2, abstract only, unofficial]. The approach builds on the sphere tracing algorithm [Parent] and enhances it with tile-based processing, synchronization mechanisms, and sparse tree traversal [S1, unofficial][S2, abstract only, unofficial].

## Sources
[S1] Accelerating Sphere Tracing, https://www.researchgate.net/publication/329152815_Accelerating_Sphere_Tracing, fetched 2026-11-04 (unofficial)
[S2] Enhanced Sphere Tracing, https://www.researchgate.net/publication/265020476_Enhanced_Sphere_Tracing, fetched 2026-11-03 (abstract only, unofficial)
