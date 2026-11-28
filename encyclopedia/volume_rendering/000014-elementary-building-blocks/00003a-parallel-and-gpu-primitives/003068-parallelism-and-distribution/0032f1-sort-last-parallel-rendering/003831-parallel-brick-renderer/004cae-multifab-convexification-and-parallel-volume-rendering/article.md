# MultiFAB Convexification and Parallel Volume Rendering


## Overview
A parallel volume rendering strategy for Adaptive Mesh Refinement (AMR) meshes where MultiFABs are processed independently by a GPU renderer (Viskores) and results are combined via MPI reduction [S1, unofficial]. It differs from standard brick renderers by explicitly handling topological dependencies and non-convexities inherent in AMR grids through a convexification preprocessing step, ensuring correct alpha blending for color and emissivity cases [S1, unofficial].

## How it works
**Inputs:**
- A MultiFAB dataset representing an AMR mesh, consisting of a hierarchy of boxes (grids) at various refinement levels [S1, unofficial].
- Optical properties: absorption coefficient, emission coefficient, and/or color/opacity data associated with the cells [G].
- Camera parameters: viewing direction and projection type (assumed orthogonal for tie-breaking logic) [G].

**Outputs:**
- A final composite image (e.g., PNG) representing the volume rendering [S1, unofficial].

**Data Structures:**
- **MultiFAB:** A collection of `Box` objects. Each `Box` defines a contiguous chunk of 3D grid data [G].
- **Image Buffer:** A 2D array of pixels, typically RGBA, initialized to zero (or background color) for accumulation [G].

**Algorithm Steps:**

1.  **Convexify the MultiFABs:**
    The raw AMR boxes may overlap or be non-convex due to the hierarchy [G]. To enable independent processing and correct merging:
    -   **Remove Covered Parts:** For any box, identify regions covered by coarser parent cells. These regions are excluded from the current box's data to prevent double-counting [S1, unofficial].
    -   **Split Disconnected Pieces:** If removing covered parts leaves a box topologically disconnected, split it into multiple separate boxes [S1, unofficial].
    -   **Break Dependency Loops:** In 3D AMR meshes, dependency loops can form where grid A depends on B, B on C, and C on A. Break these loops by splitting grids using cutting planes perpendicular to the z-axis. This reduces the layout to 2D layers where no loops are possible [S1, unofficial].
    -   **Split Size-Mismatched Boxes:** If boxes of different sizes prevent a valid topological ordering, split the problematic boxes along a Cartesian axis. The preferred axis is the one most nearly orthogonal to the viewing plane. For BoxLib grids, the z-axis is preferred because it leaves two contiguous chunks of data, which is more efficient for underlying numerical routines [S1, unofficial].

2.  **Render Each FAB Independently:**
    -   Call the GPU volume renderer (e.g., Viskores) separately for each resulting FAB (Family of Arrays) [S1, unofficial].
    -   Each renderer produces a partial image containing the accumulated color and opacity for the rays passing through that FAB's boxes [G].

3.  **Merge Partial Images (Local):**
    -   **Attenuation Only:** If the volume only has absorption (no color/emissivity), simply add the images from all local FABs together [S1, unofficial].
    -   **Color/Emissivity:** If color or emissivity is present, simple addition is insufficient. The boxes must be topologically sorted and merged in back-to-front order using alpha blending [S1, unofficial].
        -   Use a binary tree merge structure since the merge operator is associative [S1, unofficial].
        -   **Topological Sort:** Sort boxes based on their depth relative to the viewer.
        -   **Tie-Breaking:** For geometrically degenerate cases where boxes are at the same depth, sort by maximum distance from the viewer (assuming orthogonal projection). Boxes with the same maximum distance are tied; one is picked arbitrarily to go first. For tied boxes A and C, the order can be either $(A \text{ merge } C) \text{ merge } B$ or $(C \text{ merge } A) \text{ merge } B$ [S1, unofficial].

4.  **MPI Reduction:**
    -   Perform an MPI reduction operation to sum all images from all MPI ranks. This combines the contributions from different nodes in the distributed system [S1, unofficial].

5.  **Output:**
    -   Write the final composite image (e.g., PNG file) from rank 0 [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_fab_convexify_render(multifab, camera, renderer, mpi_comm, rank, has_color_or_emissivity):
    # Step 1: Convexify MultiFABs
    # Remove covered parts, split disconnected pieces, break dependency loops (z-axis),
    # split size-mismatched boxes (axis most orthogonal to view).
    convexified_fabs = [] # [G] Standard AMR box decomposition not detailed in sources

    # Step 2: Render each FAB independently
    local_images = []
    for fab in convexified_fabs:
        img = renderer.render(fab, camera) # [G] GPU volume rendering call
        local_images.append((fab, img))

    # Step 3: Merge partial images locally
    if has_color_or_emissivity:
        # Topological sort with tie-breaking
        # Sort by maximum distance from viewer (orthogonal projection)
        sorted_fabs = sorted(local_images, key=lambda x: -x[0].max_distance_from(camera))
        # Merge in back-to-front order using a binary tree
        merged_image = merge_binary_tree(sorted_fabs)
    else:
        # Attenuation only: simple addition
        merged_image = sum_images([img for _, img in local_images])

    # Step 4: MPI reduction
    global_image = mpi_comm.allreduce(merged_image, op=mpi_comm.SUM) # [G] Standard MPI reduction

    # Step 5: Output
    if rank == 0:
        write_png(global_image, "output.png") # [G] Standard image encoding

    return global_image

def merge_binary_tree(sorted_fabs):
    if len(sorted_fabs) == 1:
        return sorted_fabs[0].image
    mid = len(sorted_fabs) // 2
    left = merge_binary_tree(sorted_fabs[:mid])
    right = merge_binary_tree(sorted_fabs[mid:])
    return alpha_blend(left, right) # [G] Standard back-to-front alpha blending

def sum_images(images):
    result = zero_image()
    for img in images:
        result += img
    return result

def zero_image():
    return [[0.0, 0.0, 0.0, 0.0] for _ in range(width * height)] # [G] Standard initialization
```

## Operating characteristics
-   **Time and Memory Complexity:** Not given in the sources [S1, unofficial].
-   **Measured Performance:** Not given in the sources [S1, unofficial].
-   **Scaling:** Not given in the sources [S1, unofficial].
-   **Strengths:** Handles complex AMR topologies correctly, including dependency loops and non-convex shapes [S1, unofficial]. Efficient for large-scale parallel rendering using MPI and GPUs [S1, unofficial].
-   **Limitations:** The convexification step adds CPU overhead. Splitting boxes may increase the number of boxes. Dependency loop breaking requires cutting planes [S1, unofficial].
-   **When to Use:** Suitable for AMR meshes where topological correctness is crucial, especially for color and emissivity rendering [S1, unofficial].
-   **When Not to Use:** For octree meshes, which do not have the dependency loop problem [S1, unofficial]. If a physically-accurate solution is not required, alternative tie-breaking or pixel-independent merging might be considered, though this increases MPI communication [S1, unofficial].

## Background
This method was proposed in the context of the Quokka astro codebase for outputting volume renderings of AMR data [S1, unofficial]. It builds on the Parallel Brick Renderer concept but adapts it for the specific challenges of AMR grids, such as dependency loops and non-convexity. The approach leverages the Viskores GPU renderer and MPI for parallelism. The idea of breaking dependency loops by cutting along the z-axis to reduce to 2D layers is a key insight for handling 3D AMR meshes [S1, unofficial].

## Sources
[S1] output volume renderings, https://github.com/quokka-astro/quokka/issues/1078, fetched 2026-10-28 (unofficial)
