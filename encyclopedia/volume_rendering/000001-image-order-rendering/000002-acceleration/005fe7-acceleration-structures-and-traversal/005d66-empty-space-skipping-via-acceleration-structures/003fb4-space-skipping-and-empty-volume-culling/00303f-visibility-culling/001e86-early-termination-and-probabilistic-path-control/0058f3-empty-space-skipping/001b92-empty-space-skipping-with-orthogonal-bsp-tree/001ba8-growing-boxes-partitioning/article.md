# Growing Boxes Partitioning

Also called: Gradient boxes partitioning, Growing boxes

## Overview
Growing Boxes Partitioning is an adaptive volume partitioning method that divides a volume into sub-volumes (boxes) by clustering voxels based on their properties in both the volume domain and the transfer function domain [S1]. It aims to ensure that the visible box subset encloses few invisible voxels, thereby accelerating texture-based volume rendering by minimizing unnecessary lighting computations and texture switches [S1]. Unlike uniform partitioning or octrees, it separates surface voxels (non-zero gradient) from interior voxels (zero gradient) to optimize lit rendering, using a direct one-stage goal-oriented growth process rather than region growing followed by division [S1].

## How it works
**Inputs:**
- A 3D volume dataset (density values) [S1].
- A 1D transfer function mapping density to color and opacity [S1].
- A predetermined cost threshold for box growth [S1].
- A minimal size constraint for uniform boxes [S1].

**Outputs:**
- A set of non-overlapping boxes covering the volume, categorized as:
  - **Gradient boxes:** Enclose voxels with non-zero gradient magnitude [S1].
  - **Uniform boxes:** Enclose voxels with zero gradient magnitude and similar densities [S1].
  - **Other boxes:** Fill remaining spaces left by minimal size restrictions [S1].

**Data Structures:**
- Each box is defined by its axis-aligned bounding box (AABB) coordinates $(x_{min}, y_{min}, z_{min})$ to $(x_{max}, y_{max}, z_{max})$ and its type (gradient, uniform, other) [S1].
- Voxels are assigned to exactly one box [S1].
- Border voxels are replicated to ensure proper interpolation across box boundaries [S1].

**Algorithm Steps:**
1. **Gradient Box Creation:** Compute the gradient magnitude for each voxel [S1]. Identify voxels with non-zero gradient magnitude [S1]. Cluster these voxels into gradient boxes [S1].
2. **Uniform Box Creation:** Identify voxels with zero gradient magnitude [S1]. Group neighboring texels whose densities vary within a small range (e.g., 32 density units) [S1]. Grow boxes to enclose these uniform regions [S1]. Apply a minimal size restriction to uniform boxes [S1].
3. **Other Box Filling:** Identify any remaining spaces in the volume not covered by gradient or uniform boxes (typically due to the minimal size restriction on uniform boxes) [S1]. Fill these spaces with "other" boxes [S1].

**Cost Function and Growth Rule:**
- The partitioning balances the number of boxes and the total cost [S1].
- Boxes grow to enclose voxels of similar properties [S1].
- The general rule is to let boxes grow as large as possible while keeping the accumulated cost function smaller than a predetermined threshold [S1].
- The exact formula for the cost function is not provided in the sources [S1].

**Parameters:**
- **Minimal size of a uniform box:** A restriction on the minimum dimensions of a uniform box. This may leave spaces filled by other boxes [S1].
- **Density range for uniformity:** The maximum allowed density variation within a uniform box (e.g., 32 units) [S1].
- **Cost threshold:** The maximum accumulated cost for a box before growth stops [S1].

**Edge Cases and Pitfalls:**
- **Minimal Size Restrictions:** If the minimal size for uniform boxes is too large, significant portions of the volume may be left uncovered by uniform boxes, requiring "other" boxes, which may not offer the same optimization benefits [S1].
- **Overhead:** Increasing the number of boxes increases overhead for setting up and switching textures and increases replicated storage at box borders for proper interpolation [S1].
- **Lit vs. Unlit Rendering:** Uniform boxes alone are inefficient for lit rendering because they include many zero-gradient voxels for lighting computation. Gradient boxes solve this by isolating surface voxels [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def growing_boxes_partitioning(volume, transfer_function, cost_threshold, min_uniform_size):
    """
    Partitions a volume into gradient, uniform, and other boxes.
    Args:
        volume: 3D array of density values.
        transfer_function: 1D lookup table mapping density to color/opacity.
        cost_threshold: Maximum accumulated cost for a box.
        min_uniform_size: Minimum size for a uniform box.
    Returns:
        A list of Box objects.
    """
    boxes = []
    visited = set()

    # Step 1: Create Gradient Boxes
    # Compute gradient magnitude for each voxel and identify non-zero values [S1]
    gradient_voxels = []
    for z in range(volume.shape[0]):
        for y in range(volume.shape[1]):
            for x in range(volume.shape[2]):
                if compute_gradient_magnitude(volume, x, y, z) > 0:
                    gradient_voxels.append((x, y, z))
                    visited.add((x, y, z))

    # Cluster gradient voxels into boxes based on cost threshold [S1]
    # Note: Specific clustering algorithm not detailed in sources [S1]
    gradient_boxes = []
    boxes.extend(gradient_boxes)

    # Step 2: Create Uniform Boxes
    # Identify voxels with zero gradient magnitude [S1]
    uniform_voxels = []
    for z in range(volume.shape[0]):
        for y in range(volume.shape[1]):
            for x in range(volume.shape[2]):
                if (x, y, z) not in visited and compute_gradient_magnitude(volume, x, y, z) == 0:
                    uniform_voxels.append((x, y, z))
                    visited.add((x, y, z))

    # Group uniform voxels by density similarity (range of 32) [S1]
    uniform_regions = []
    uniform_boxes = []
    for region in uniform_regions:
        # Grow box as large as possible while cost < threshold [S1]
        box = grow_box(region, cost_threshold)

        # Check minimal size restriction [S1]
        if box.size() >= min_uniform_size:
            uniform_boxes.append(box)
        else:
            # If too small, it is not added as a uniform box, leaving a gap [S1]
            pass

    boxes.extend(uniform_boxes)

    # Step 3: Fill Remaining Spaces with Other Boxes
    # Identify uncovered spaces [S1]
    remaining_spaces = []
    # Fill with other boxes [S1]
    other_boxes = []
    boxes.extend(other_boxes)

    return boxes

def compute_gradient_magnitude(volume, x, y, z):
    # Standard gradient computation (e.g., Sobel or finite differences) [G]
    pass

def grow_box(region, cost_threshold):
    # Grow box to enclose region while keeping cost < threshold [S1]
    pass
```

## Subtopics

### [Orthogonal BSP Tree Construction](1134-orthogonal-bsp-tree-construction.md)
Converts the set of growing boxes into an orthogonal BSP tree for visibility sorting. It sets itself apart by imposing a strict hierarchical structure on the adaptive boxes, ensuring trivial visibility sorting for 3D texture rendering.

## Operating characteristics
- **Time Complexity:** Not explicitly given. The one-stage growth process is described as more direct than region growing followed by division [S1].
- **Memory Complexity:** Increases with the number of boxes due to replicated storage at box borders for interpolation [S1].
- **Performance:** Improves rendering speed by skipping invisible sub-volumes and reducing unnecessary lighting computations on zero-gradient voxels [S1].
- **Strengths:** Effective for lit rendering by isolating surface voxels; adaptive to volume properties [S1].
- **Limitations:** Overhead increases with the number of boxes; minimal size restrictions may lead to inefficient "other" boxes [S1].
- **When to Use:** When lit rendering performance is critical and the volume has distinct surface and interior regions [S1].
- **When Not to Use:** When the overhead of managing many small boxes outweighs the benefits, or for unilluminated rendering where uniform boxes alone may suffice [S1].

## Background
- **Origin:** Introduced by Li and Kaufman in 2003 [S1].
- **History:** Initially experimented with 2D textures, later extended to 3D textures using an orthogonal BSP tree [S1].
- **Usage:** Used in texture-based volume rendering for early termination and empty space skipping [S1].

## Sources
[S1] Empty Space Skipping and Occlusion Clipping for Texture-based Volume ..., https://www3.cs.stonybrook.edu/~mueller/papers/OcclusionClipping.pdf, fetched 2026-10-26
