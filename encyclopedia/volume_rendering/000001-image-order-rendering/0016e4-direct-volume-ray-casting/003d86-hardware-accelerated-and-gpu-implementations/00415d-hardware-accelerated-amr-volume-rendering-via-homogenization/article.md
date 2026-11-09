# Hardware-Accelerated AMR Volume Rendering via Homogenization

Also called: AMR Partition Tree Rendering, Tiling AMR domain with grid patches

## Overview
This method renders Adaptive Mesh Refinement (AMR) data by preprocessing the hierarchy into a set of non-overlapping Cartesian grid patches, allowing standard Direct Volume Ray Casting (DVR) algorithms to be used without resampling [S1, S2, unofficial]. It differs from methods that resample AMR data to a uniform grid or render individual grids with complex boundary handling by explicitly tiling the domain with homogeneous resolution regions [S1, S2, unofficial].

## How it works
**Inputs:** An AMR dataset satisfying the Berger–Colella constraints: integer refinement ratios (typically 2 or 4) between adjacent levels, no partial refinement of cells, and no overlapping grids within the same hierarchy level [S1, unofficial].
**Outputs:** A view-dependent back-to-front ordered list of grid patches, each rendered as an RGB image with an alpha channel, composited into the final image [S1, unofficial].

**Data Structures:**
1.  **AMR Partition Tree:** A generalized kD-tree where each leaf represents a single grid patch. A grid patch is a rectangular subgrid of a single AMR grid, containing cells of identical size and data stored in a contiguous memory region [S1, unofficial].
2.  **Grid Patch:** Defined by its parent grid ID, local coordinates, and resolution.

**Algorithm Steps:**

1.  **Initialize Tree:** Create a kD-tree with a single root leaf representing the entire AMR domain [S1, unofficial].
2.  **Insert Grids:** Iterate through AMR grids in order of increasing resolution (finest last) [S1, unofficial].
    *   For each grid, traverse the kD-tree downwards.
    *   Existing interior kD-tree nodes implicitly split the incoming grid into candidate patches [S1, unofficial].
    *   **Case A (Complete Containment):** If an interior node is completely contained inside one of the candidate patches created by the incoming grid, replace that node and its entire subtree with a single leaf representing that new higher-resolution patch [S1, unofficial].
    *   **Case B (Partial Overlay):** If an existing kD-tree leaf is partially overlaid by the incoming grid patch, recursively split that leaf into a set of non-overlapped patches (retaining the coarse resolution) and a single overlaid patch [S1, unofficial]. Replace the overlaid patch with the new higher-resolution grid patch [S1, unofficial].
3.  **Enumerate Order:** Perform a directed depth-first traversal of the resulting kD-tree to determine rendering order [S1, unofficial].
    *   At each internal node, compare the camera viewpoint with the node's dividing plane [S1, unofficial].
    *   Traverse the subtree corresponding to the region "behind" the dividing plane (from the viewpoint's perspective) first [S1, unofficial].
    *   Traverse the other region second [S1, unofficial].
    *   This ensures that no patch rendered in the first traversal can occlude any patch rendered in the second, guaranteeing correct front-to-back compositing even if the viewpoint is inside the domain [S1, unofficial].
4.  **Render:** For each patch in the enumerated order, upload the patch data to 3D texture memory and execute standard hardware-accelerated DVR (ray generation, sampling, compositing) [S1, S2, unofficial]. Patches are composited into the frame buffer, typically using hardware alpha blending [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_amr_homogenized(amr_grids, camera_viewpoint):
    # Step 1: Initialize kD-tree
    # Root leaf covers the entire domain bounding box
    kdtree_root = KDTreeNode(domain=amr_domain, is_leaf=True, patch=None)
    
    # Step 2: Insert grids by increasing resolution
    sorted_grids = sort_by_resolution(amr_grids) # Coarsest to finest
    
    for grid in sorted_grids:
        insert_grid_into_kdtree(kdtree_root, grid)
        
    # Step 3: Enumerate patches in back-to-front order
    ordered_patches = []
    traverse_kdtree_for_order(kdtree_root, camera_viewpoint, ordered_patches)
    
    # Step 4: Render patches
    for patch in ordered_patches:
        # Upload patch data to GPU texture
        upload_texture(patch.data)
        # Execute standard hardware-accelerated DVR for this patch
        # The hardware compositing (alpha blending) handles the over operator
        render_patch_hardware(patch)

def insert_grid_into_kdtree(node, grid):
    if node.is_leaf:
        # Check if grid completely covers this leaf's domain
        if grid.contains(node.domain):
            # Case A: Replace leaf with new higher-res patch
            node.is_leaf = False
            node.patch = Patch.from_grid(grid, node.domain)
            node.children = []
        else:
            # Case B: Partial overlap. Split leaf.
            # Note: Implementation detail [G] - split logic depends on 
            # intersection of grid bounds and leaf bounds.
            split_leaf_into_patches(node, grid)
    else:
        # Interior node: Recurse into children
        # The grid is implicitly split by the node's dividing plane
        for child in node.children:
            if grid.intersects(child.domain):
                insert_grid_into_kdtree(child, grid)

def split_leaf_into_patches(node, grid):
    # Split the current leaf domain into:
    # 1. Regions not covered by the new grid (coarse patches)
    # 2. Region covered by the new grid (fine patch)
    # Replace the leaf with an interior node representing these splits
    # [G] Specific splitting geometry depends on AMR grid boundaries
    # and the kD-tree plane.
    pass

def traverse_kdtree_for_order(node, viewpoint, result_list):
    if node.is_leaf:
        if node.patch is not None:
            result_list.append(node.patch)
        return
    
    # Compare viewpoint with node's dividing plane
    behind_plane = is_behind_plane(viewpoint, node.dividing_plane)
    
    if behind_plane:
        # Traverse behind child first
        traverse_kdtree_for_order(node.behind_child, viewpoint, result_list)
        traverse_kdtree_for_order(node.front_child, viewpoint, result_list)
    else:
        # Traverse front child first
        traverse_kdtree_for_order(node.front_child, viewpoint, result_list)
        traverse_kdtree_for_order(node.behind_child, viewpoint, result_list)
```

**Parameters:**
*   **Refinement Ratios:** Integers, typically 2 or 4 [S1, unofficial].
*   **Transfer Function Scaling:** Opacity and saturation scaling may be applied to handle varying resolution levels [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Viewpoint Inside Domain:** The kD-tree traversal order handles this correctly by using the viewpoint to determine "behind" vs "front" relative to dividing planes [S1, unofficial].
*   **Partial Refinement:** The method assumes no partial refinement; if present, the algorithm is undefined [S1, unofficial].
*   **Precision:** Using hardware-accelerated DVR avoids varying sample distances associated with texture slicing, offering higher precision [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Preprocessing (homogenization) is computationally efficient, relying on kD-tree insertions [S2, unofficial]. Rendering time scales with the number of grid patches and the complexity of the transfer function [S2, unofficial].
*   **Memory Complexity:** Requires storing the kD-tree structure and the AMR data. Grid patches are stored in contiguous memory regions, optimizing texture cache usage [S1, unofficial].
*   **Performance:** Homogenization is cited as the most efficient way to render AMR hierarchies because it is computationally efficient and allows the use of standard, optimized hardware-accelerated methods [S2, unofficial].
*   **Strengths:** No resampling required; leverages standard DVR hardware; correct compositing order guaranteed by kD-tree traversal [S1, S2, unofficial].
*   **Limitations:** Requires AMR data to satisfy specific constraints (integer refinement ratios, no partial refinement) [S1, unofficial]. The number of patches can increase significantly with complex AMR structures, potentially increasing draw calls [S1, unofficial].

## Background
The method was developed to address the challenges of rendering AMR data, which does not fit standard Cartesian grid rendering algorithms. It was presented in the context of interactive direct volume rendering for AMR data, specifically tested on a star cluster formation simulation (Root level 32x32x32, data from Greg Bryan, Theoretical Astronomy Group, MIT) [S2, unofficial]. The approach builds on earlier work by Kähler & Hege (2001/2002) regarding texture-based AMR volume rendering and was further refined for GPU-assisted raycasting [S2, unofficial].

## Sources
[S1] Remote Interactive Direct Volume Rendering of AMR Data, https://www.osti.gov/servlets/purl/799586, fetched 2026-10-26 (unofficial)
[S2] Visualization Tools for Adaptive Mesh Refinement Data, https://www.vacet.org/publications/AMRVisJune2007.pdf, fetched 2026-10-27 (unofficial)
