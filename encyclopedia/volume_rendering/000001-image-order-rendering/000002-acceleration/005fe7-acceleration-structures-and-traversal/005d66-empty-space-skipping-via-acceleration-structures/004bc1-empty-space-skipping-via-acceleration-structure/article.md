# Empty-space skipping via Acceleration Structure

Also called: Empty space skipping via the acceleration structure

## Overview
This method accelerates direct volume rendering by using a hierarchical min/max acceleration structure to skip transparent regions of the volume data [S1][S2, unofficial]. It reduces the number of samples taken per ray by jumping over blocks that contain no visible data, distinct from sibling methods that may use sparse voxel grids or hierarchical bounding box trees [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A source volumetric dataset (e.g., $512 \times 512 \times 512$) and a pre-computed acceleration structure [S2, unofficial].
*   **Output:** A rendered image with reduced computational cost [S2, unofficial].

**Data Structures:**
The acceleration structure is a small two-channel volumetric dataset [S2, unofficial].
*   **Dimensions:** $16 \times 16 \times 16$ cells [S2, unofficial].
*   **Layout:** Each cell corresponds to a $32 \times 32 \times 32$ block in the source dataset [S2, unofficial].
*   **Content:** Each cell stores the minimum and maximum values found within its corresponding source block [S2, unofficial].
*   **Alternative Layout:** Some implementations use a per-field min/max structure organized in $32 \times 32 \times 32$ bricks [S1].

**Algorithm Steps:**
1.  For each ray, determine the intersection with the acceleration structure grid.
2.  Traverse the acceleration structure cells along the ray path.
3.  For each cell, retrieve the stored minimum and maximum values [S2, unofficial].
4.  Check if the space corresponding to the cell is transparent. This is determined by comparing the min/max values against the transfer function's opacity threshold or visibility criteria [S2, unofficial].
5.  If the cell is transparent (i.e., contains no visible data), skip the entire $32 \times 32 \times 32$ block and advance the ray to the next cell [S2, unofficial].
6.  If the cell is not transparent, sample the source data within that block and perform standard front-to-back compositing [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray(ray_origin, ray_direction, volume_data, accel_structure):
    # Determine initial entry point into the acceleration structure grid
    t_min, t_max = intersect_accel_grid(ray_origin, ray_direction)
    
    # Traverse acceleration structure cells
    while t_min < t_max:
        # Get current cell coordinates in the 16x16x16 structure
        cell_coords = get_cell_coords(ray_origin, ray_direction, t_min)
        
        # Retrieve min/max values for this cell
        min_val, max_val = accel_structure[cell_coords]
        
        # Check if the block is transparent
        if is_transparent(min_val, max_val):
            # Skip this 32x32x32 block
            t_min += step_size_accel_cell(ray_direction)
        else:
            # Process the block in the source volume
            block_origin = get_block_origin(cell_coords)
            block_samples = sample_block(block_origin, ray_origin, ray_direction, volume_data)
            composite(block_samples)
            
            # Advance ray past this block
            t_min += step_size_accel_cell(ray_direction)
            
    return accumulated_color

def is_transparent(min_val, max_val):
    # Implementation depends on transfer function; 
    # typically true if max_val < opacity_threshold
    return max_val < OPACITY_THRESHOLD
```

**Parameters:**
*   **Block Size:** $32 \times 32 \times 32$ voxels [S2, unofficial].
*   **Acceleration Structure Resolution:** $16 \times 16 \times 16$ cells [S2, unofficial].
*   **Source Resolution Example:** $512 \times 512 \times 512$ [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Min/max values must accurately represent the range of the source block to avoid false negatives (skipping visible data) or false positives (skipping empty data) [S2, unofficial].
*   **Boundary Conditions:** Ray traversal must correctly handle entry and exit from the acceleration structure grid [S2, unofficial].

## Subtopics

### [SparseLeap](5152-sparseleap.md)
Uses sparse voxel octrees for skipping; chosen for highly sparse data.

### [Hierarchical Bounding Box Tree for Empty-Space Skipping](8182-hierarchical-bounding-box-tree-for-empty-space-skipping.md)
Uses AABB trees; chosen for complex geometry integration.

### [SparseLeap Ray-Casting with Occupancy Misses](9818-sparseleap-ray-casting-with-occupancy-misses.md)
Extends SparseLeap with occupancy misses; chosen for dynamic scenes.

### [Space Skipping and Empty Volume Culling](10063-space-skipping-and-empty-volume-culling.md)
Uses bounding volumes; chosen for static volumes with large empty regions.

### [Progressive Ray Casting for Mobile](16346-progressive-ray-casting-for-mobile-devices.md)
Adaptive sampling; chosen for mobile performance.

## Operating characteristics
*   **Performance:** Accelerates visualization by 1.5-2 times compared to non-skipping methods [S2, unofficial].
*   **Complexity:** Time complexity is reduced proportional to the fraction of empty space skipped [S2, unofficial]. Memory complexity is dominated by the acceleration structure, which is significantly smaller than the source volume ($16^3$ vs $512^3$) [S2, unofficial].
*   **Strengths:** Simple to implement on GPU; effective for volumes with large transparent regions [S2, unofficial].
*   **Limitations:** Requires pre-computation of the acceleration structure; less effective for dense volumes [S2, unofficial].

## Background
This technique is an extension of CPU-based empty space skipping methods to GPU implementations [S2, unofficial]. It is used in direct volume rendering to optimize ray casting by avoiding unnecessary sampling of transparent regions [S1][S2, unofficial].

## Sources
[S1] Direct-volume raycasting, https://met3d.readthedocs.io/en/latest/05_developer_manual/03_actors/algorithms/direct_volume_rendering.html, fetched 2026-10-27
[S2] GENERAL IMPLEMENTATION ASPECTS OF THE GPU- ..., http://sv-journal.org/2011-1/02/, fetched 2026-10-30 (unofficial)
