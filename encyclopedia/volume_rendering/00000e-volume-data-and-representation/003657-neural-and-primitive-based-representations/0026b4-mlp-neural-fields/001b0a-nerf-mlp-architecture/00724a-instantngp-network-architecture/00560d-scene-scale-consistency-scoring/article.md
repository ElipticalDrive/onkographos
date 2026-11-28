# Scene Scale Consistency Scoring

Also called: SSCS

## Overview
Scene Scale Consistency Scoring (SSCS) is a post-processing heuristic used to detect and remove floating artifacts in neural radiance fields by comparing occupancy grids from models trained with different bounding box scales. It distinguishes itself from sibling methods like Occupancy Clustering by exploiting the geometric invariance of real scene structures against the scale-dependent manifestation of artifacts, thereby filtering out inconsistent density before clustering is applied.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of trained Instant-NGP models, each associated with a specific `aabb_scale` value $s \in S$. The standard set is $S = [8, 16, 32]$ [S1]. An occupancy grid representation for each model.
*   **Outputs:** A filtered occupancy grid where cells identified as inconsistent (likely artifacts) are set to 0.

**Data Structures:**
The method operates on a 3D occupancy grid. Each cell $C_k$ is indexed by integer coordinates $(x_I, y_I, z_I)$. The grid stores binary occupancy values: 1 if the cell is occupied (density > 0), and 0 otherwise [S1].

**Algorithm Steps:**
1.  **Training:** Train separate Instant-NGP models using different `aabb_scale` values from the set $S$. The `aabb_scale` defines the scaling factor of the unit cube in which ray marching is performed. Powers of 2 are recommended to align grid borders [S1].
2.  **Iteration:** For each cell $C_k(x_I, y_I, z_I)$ in the grid:
    a.  Retrieve the occupancy status $C_{k,s}(x_I, y_I, z_I)$ from each trained model corresponding to scale $s \in S$. This value is 1 if the cell is occupied in that model, and 0 otherwise.
    b.  Calculate $N(S, C_{k,s})$, the number of scales $s \in S$ for which the cell $C_k$ lies within the training volume extent defined by scale $s$. This represents the number of scaled scenes in which the cell is theoretically settable [S1].
    c.  Compute the SSCS score using the formula below.
3.  **Filtering:** If $SSCS(C_k) < 1$, the cell is considered inconsistently set (meaning at least one scaled model does not carry density there). Set the occupancy of this cell to 0 [S1].
4.  **Execution Order:** Perform SSCS filtering **before** Occupancy Clustering. This step breaks the coherence between relevant geometry and floaters, enabling subsequent clustering to detect remaining artifacts more reliably [S1].

**Formula:**
The SSCS score for a cell is the fraction of scaled scenes in which the occupancy of the cell is set, normalized by the number of scenes where the cell is theoretically valid:

$$
SSCS(C_k(x_I, y_I, z_I)) = \frac{\sum_{s \in S} C_{k,s}(x_I, y_I, z_I)}{N(S, C_{k,s}(x_I, y_I, z_I))}
$$

Where:
*   $S$ is the set of scales, typically $[8, 16, 32]$ [S1].
*   $C_{k,s}(x_I, y_I, z_I)$ is the occupancy of cell $k$ in the model trained with scale $s$ (1 if occupied, 0 otherwise) [S1].
*   $N(S, C_{k,s}(x_I, y_I, z_I))$ is the count of scales $s \in S$ whose corresponding extents contain the cell [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sscs_filtering(occupancy_grids, scales):
    """
    Filters occupancy grids using Scene Scale Consistency Scoring.
    
    Args:
        occupancy_grids: A dictionary mapping scale s to a 3D numpy array 
                         representing the occupancy grid (1 for occupied, 0 for empty).
        scales: A list of aabb_scale values used for training (e.g., [8, 16, 32]).
        
    Returns:
        A filtered 3D numpy array representing the combined occupancy grid.
    """
    # Assume all grids share the same resolution and coordinate system
    # The grid dimensions must be large enough to cover the largest scale
    grid_shape = occupancy_grids[scales[0]].shape
    filtered_grid = np.zeros(grid_shape, dtype=np.uint8)
    
    for x in range(grid_shape[0]):
        for y in range(grid_shape[1]):
            for z in range(grid_shape[2]):
                occupied_count = 0
                valid_scale_count = 0
                
                for s in scales:
                    grid = occupancy_grids[s]
                    
                    # Check if the cell is within the theoretical extent of this scale
                    # The exact boundary check depends on how the grid maps to the 
                    # aabb_scale. Generally, if the grid covers the volume defined 
                    # by the scale, the cell is valid.
                    # Note: The source implies N(S, Ck,s) counts scales where the 
                    # cell is inside the training volume.
                    if is_cell_in_extent(x, y, z, s, grid_shape):
                        valid_scale_count += 1
                        
                        # Check occupancy in this specific model's grid
                        if grid[x, y, z] == 1:
                            occupied_count += 1
                
                # Calculate SSCS score
                if valid_scale_count > 0:
                    sscs_score = occupied_count / valid_scale_count
                else:
                    sscs_score = 0.0
                
                # Filter: Keep cell only if it is occupied in ALL valid scales
                # SSCS < 1 means at least one valid scale has 0 occupancy
                if sscs_score == 1.0:
                    filtered_grid[x, y, z] = 1
                else:
                    filtered_grid[x, y, z] = 0
                    
    return filtered_grid

def is_cell_in_extent(x, y, z, scale, grid_shape):
    """
    Determines if a grid cell is within the theoretical extent of a given aabb_scale.
    This is a placeholder for the specific mapping logic between grid indices 
    and the scaled bounding box. In practice, this depends on the grid resolution 
    and how the aabb_scale maps to world coordinates.
    [G] Implementation detail: Typically, the grid covers a fixed world volume. 
    The 'extent' refers to whether the cell's world position falls within the 
    bounding box defined by the aabb_scale. If the grid is fixed and covers 
    the maximum possible scene size (e.g., scale 32), then all cells are 
    theoretically settable for all scales if the grid resolution is sufficient.
    However, if the grid is cropped or scaled, this check is non-trivial.
    The source states N(S, Ck,s) is the number of scales whose extents contain the cell.
    For simplicity, if the grid covers the union of all scales, valid_scale_count 
    might equal len(scales) for all cells. If the grid is fixed to the largest scale,
    then all cells are valid for all scales.
    """
    # [G] Assuming the grid resolution is fixed to cover the largest scale (32).
    # If the grid covers the volume of scale 32, then cells outside the volume 
    # of scale 8, 16, etc. would be 0 or undefined. 
    # The source implies we only count scales where the cell is "theoretically setable".
    # If the grid is static and covers the max extent, we check if the cell's 
    # world coordinate is within the bounding box of the specific scale.
    # This requires knowing the world coordinate of (x,y,z).
    # For the purpose of this pseudocode, we assume a helper exists or the 
    # grid is dense enough that all cells are valid for all scales in S.
    # If the grid is sparse or cropped, this function must implement the 
    # bounding box check.
    return True # Placeholder: Requires specific coordinate mapping logic
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** Cells near the edge of the training volume may have different $N(S, C_{k,s})$ values depending on how the `aabb_scale` affects the grid's effective coverage.
*   **Precision:** The score is a fraction. Floating-point comparison should be used carefully, though the threshold is strictly $< 1$ [S1].
*   **Degenerate Input:** If a cell is outside the extent of all scales, $N(S, C_{k,s}) = 0$, leading to division by zero. Such cells should be treated as empty or excluded [S1].

## Subtopics

### Occupancy Clustering
A subsequent step that groups connected occupied cells to identify and remove small, isolated clusters (floaters) that survived SSCS. It relies on SSCS to break coherence between geometry and artifacts. [Link](5049-instantngp-network-architecture.md)

### Instant-NGP Network Architecture
The base method that SSCS post-processes. It uses hash grids and a tiny MLP. SSCS operates on the output occupancy grids of this architecture. [Link](5049-instantngp-network-architecture.md)

## Operating characteristics
*   **Time Complexity:** $O(|S| \cdot N)$, where $|S|$ is the number of scales (typically 3) and $N$ is the number of cells in the occupancy grid. The filtering step is linear in the number of cells.
*   **Memory Complexity:** $O(|S| \cdot N)$ to store the occupancy grids for all scales.
*   **Performance:** Effectively removes floaters that are coherent with scene geometry, which clustering alone cannot remove. It is most effective on scenes with sparse input data where artifacts manifest differently across scales (e.g., the "garbage" dataset) [S1].
*   **Strengths:** Simple to implement; does not require additional training; improves the effectiveness of subsequent clustering.
*   **Limitations:** Requires training multiple models (increasing training time and memory); relies on the assumption that real geometry is invariant to scale, which may not hold for all scene types or if the scale changes significantly distort the representation.
*   **When to Use:** When floaters are present and appear coherent with the scene geometry, making them resistant to simple clustering.
*   **When Not to Use:** When computational resources are limited and multiple training runs are prohibitive.

## Background
SSCS was introduced in "A Post Processing Technique to Automatically Remove Floater Artifacts in Neural Radiance Fields" (2023) as a method to improve the quality of NeRF reconstructions by leveraging the properties of Instant-NGP's scale-dependent training. It addresses the limitation of occupancy clustering, which fails when artifacts are connected to the main geometry [S1].

## Sources
[S1] A Post Processing Technique to Automatically Remove Floater Artifacts in Neural Radiance Fields, https://doi.org/10.1111/cgf.14977, fetched 2026-11-03
