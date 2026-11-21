# MultiFAB Convexification and Parallel Volume Rendering

Also called: MultiFAB Convexification

## Overview
A parallel volume rendering strategy for Adaptive Mesh Refinement (AMR) meshes where MultiFABs are processed independently by a GPU renderer and results are combined via MPI reduction. It differs from standard brick renderers by explicitly handling the topological dependencies and non-convexities inherent in AMR grids through a "convexification" preprocessing step, ensuring correct alpha blending for color and emissivity cases [S1, unofficial].

## How it works
**Inputs:**
- A MultiFAB dataset representing an AMR mesh, consisting of a hierarchy of boxes (grids) at various refinement levels.
- Optical properties: absorption coefficient $\kappa$, emission coefficient $j$, and/or color/opacity data associated with the cells.
- Camera parameters: viewing direction and projection type (assumed orthogonal for tie-breaking logic).

**Outputs:**
- A final composite image (e.g., PNG) representing the volume rendering.

**Data Structures:**
- **MultiFAB:** A collection of `Box` objects. Each `Box` defines a contiguous chunk of 3D grid data.
- **Image Buffer:** A 2D array of pixels, typically RGBA, initialized to zero (or background color) for accumulation.

**Algorithm Steps:**

1.  **Convexify the MultiFABs:**
    The raw AMR boxes may overlap or be non-convex due to the hierarchy. To enable independent processing and correct merging:
    -   **Remove Covered Parts:** For any box, identify regions covered by coarser parent cells. These regions are excluded from the current box's data to prevent double-counting.
    -   **Split Disconnected Pieces:** If removing covered parts leaves a box topologically disconnected, split it into multiple separate boxes.
    -   **Break Dependency Loops:** In 3D AMR meshes, dependency loops can form where grid A depends on B, B on C, and C on A. Break these loops by splitting grids using cutting planes perpendicular to the z-axis. This reduces the layout to 2D layers where no loops are possible [S1, unofficial].
    -   **Split Size-Mismatched Boxes:** If boxes of different sizes prevent a valid topological ordering, split the problematic boxes along a Cartesian axis. The preferred axis is the one most nearly orthogonal to the viewing plane. For BoxLib grids, the z-axis is preferred because it leaves two contiguous chunks of data, which is more efficient for underlying numerical routines [S1, unofficial].

2.  **Render Each FAB Independently:**
    -   Call the GPU volume renderer (e.g., Viskores) separately for each resulting FAB (Family of Arrays).
    -   Each renderer produces a partial image containing the accumulated color and opacity for the rays passing through that FAB's boxes.

3.  **Merge Partial Images (Local):**
    -   **Attenuation Only:** If the volume only has absorption (no color/emissivity), simply add the images from all local FABs together.
    -   **Color/Emissivity:** If color or emissivity is present, simple addition is insufficient. The boxes must be topologically sorted and merged in back-to-front order using alpha blending.
        -   Use a binary tree merge structure since the merge operator is associative.
        -   **Topological Sort:** Sort boxes based on their depth relative to the viewer.
        -   **Tie-Breaking:** For geometrically degenerate cases where boxes are at the same depth, sort by maximum distance from the viewer (assuming orthogonal projection). Boxes with the same maximum distance are tied; one is picked arbitrarily to go first. For tied boxes A and C, the order can be either $(A \text{ merge } C) \text{ merge } B$ or $(C \text{ merge } A) \text{ merge } B$ [S1, unofficial].

4.  **MPI Reduction:**
    -   Perform an MPI reduction operation to sum all images from all MPI ranks. This combines the contributions from different nodes in the distributed system.

5.  **Output:**
    -   Write the final composite image (e.g., PNG file) from rank 0 [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_fab_convexify_render(multifab, camera, renderer):
    """
    Render a MultiFAB using convexification and parallel reduction.
    """
    # Step 1: Convexify
    # Remove parts covered by coarse cells and split disconnected/loop-causing boxes
    convexified_boxes = convexify(multifab)
    
    # Step 2: Render each FAB independently
    local_images = []
    for fab in convexified_boxes:
        # Render the specific FAB using the GPU renderer
        img = renderer.render(fab, camera)
        local_images.append((fab, img))
        
    # Step 3: Merge locally if color/emissivity is present
    # If only attenuation, simple addition is sufficient (handled in reduction)
    if has_color_or_emissivity():
        # Sort boxes topologically for correct alpha blending
        sorted_boxes = topological_sort(local_images, camera)
        
        # Merge in back-to-front order using a binary tree
        merged_image = merge_binary_tree(sorted_boxes)
    else:
        # Sum images for attenuation-only case
        merged_image = sum_images([img for _, img in local_images])
        
    # Step 4: MPI Reduction
    # Sum all images from all ranks
    global_image = mpi_reduce_sum(merged_image)
    
    # Step 5: Output
    if rank == 0:
        write_png(global_image, "output.png")
        
    return global_image

def convexify(multifab):
    """
    Remove covered parts and split boxes to ensure convexity and no dependency loops.
    """
    boxes = []
    for box in multifab.boxes:
        # Remove regions covered by coarser parents
        clean_box = remove_covered_regions(box, multifab)
        
        # Split if topologically disconnected
        pieces = split_disconnected(clean_box)
        
        # Break dependency loops by cutting along z-axis if necessary
        # This reduces 3D layout to 2D layers
        final_pieces = break_dependency_loops(pieces, axis='z')
        
        boxes.extend(final_pieces)
    return boxes

def topological_sort(boxes, camera):
    """
    Sort boxes in back-to-front order.
    Tie-break by max distance from viewer (orthogonal projection).
    """
    def sort_key(box):
        # Calculate max distance from viewer for the box
        max_dist = box.max_distance_from(camera.origin, camera.direction)
        return -max_dist  # Negative for back-to-front (far to near)
    
    sorted_boxes = sorted(boxes, key=sort_key)
    return sorted_boxes

def merge_binary_tree(sorted_boxes):
    """
    Merge images in back-to-front order using a binary tree.
    """
    if not sorted_boxes:
        return empty_image()
    
    # Simple linear merge for demonstration, binary tree is more efficient
    result = sorted_boxes[0].image
    for i in range(1, len(sorted_boxes)):
        # Alpha blend current result with next box's image
        result = alpha_blend(result, sorted_boxes[i].image)
    return result
```

## Operating characteristics
-   **Time Complexity:** Dominated by the GPU rendering of each FAB and the MPI reduction. The convexification step adds CPU overhead proportional to the number of boxes and their complexity.
-   **Memory Complexity:** Requires storing the MultiFAB data and intermediate images. The convexification step may increase the number of boxes, potentially increasing memory overhead slightly.
-   **Strengths:** Handles complex AMR topologies correctly, including dependency loops and non-convex shapes. Efficient for large-scale parallel rendering using MPI and GPUs.
-   **Limitations:** The convexification step can be complex to implement correctly. Splitting boxes may lead to a large number of small boxes, potentially reducing rendering efficiency if not managed well. Dependency loop breaking requires careful handling of cutting planes.
-   **When to Use:** Suitable for AMR meshes where topological correctness is crucial, especially for color and emissivity rendering. Not ideal for simple, uniform grids where standard brick rendering suffices.
-   **When Not to Use:** For octree meshes, which do not have the dependency loop problem, simpler methods may be preferred. If a physically-accurate solution is not required, alternative tie-breaking or pixel-independent merging might be considered, though this increases MPI communication [S1, unofficial].

## Background
This method was proposed in the context of the Quokka astro codebase for outputting volume renderings of AMR data [S1, unofficial]. It builds on the Parallel Brick Renderer concept but adapts it for the specific challenges of AMR grids, such as dependency loops and non-convexity. The approach leverages the Viskores GPU renderer and MPI for parallelism. The idea of breaking dependency loops by cutting along the z-axis to reduce to 2D layers is a key insight for handling 3D AMR meshes [S1, unofficial].

## Sources
[S1] output volume renderings, https://github.com/quokka-astro/quokka/issues/1078, fetched 2026-10-28 (unofficial)
