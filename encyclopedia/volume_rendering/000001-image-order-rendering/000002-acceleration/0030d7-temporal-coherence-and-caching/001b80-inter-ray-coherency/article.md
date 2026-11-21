# Inter-Ray Coherency

Also called: Reference Ray Caster

## Overview
Inter-Ray Coherency is an acceleration technique for volume ray casting that exploits spatial correlation between adjacent rays to reduce redundant computation. It solves the problem of repeated, expensive ray-surface intersection tests by caching traversal results from a previously processed ray and reusing them for subsequent rays in a bundle. Unlike sibling methods that rely on temporal accumulation or image-based reprojection, this method operates within a single frame by leveraging the high coherence of ray paths as they traverse the volume grid [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset, an iso-surface threshold, and a set of rays (typically scanlines or pixel rays) originating from the eye point.
*   **Output:** A rendered image where rays terminate at the first iso-surface intersection or volume exit.

**Data Structures:**
*   **Reference Buffer:** A data structure storing the state of the "last ray" processed. It contains:
    *   A list of cells (voxels) encountered along the last ray's path.
    *   A flag (or bit) for each cell indicating whether an iso-surface portion was found within it.
*   **Current Ray State:** Tracks the current traversal position and intersection status.

**Algorithm Steps:**
1.  **Initialize:** For the first ray in a bundle (or frame), perform standard fast voxel traversal and ray-surface intersection tests.
2.  **Cache Update:** After processing a ray, store its visited cells and intersection flags in the Reference Buffer.
3.  **Reuse for Next Ray:** For the subsequent ray:
    *   Check the Reference Buffer for cells previously visited by the last ray.
    *   If the current ray passes through a cell marked in the buffer:
        *   If the buffer flag indicates an intersection exists, perform the ray-surface intersection test only for that specific cell.
        *   If the buffer flag indicates no intersection, skip the intersection test for that cell, assuming coherence holds.
    *   If the current ray deviates significantly from the last ray's path (e.g., enters a new region not in the buffer), fall back to standard traversal.
4.  **Early Termination:** If an intersection is found, terminate the ray. The buffer is updated with the path up to the intersection.
5.  **Iterate:** Repeat for all rays in the bundle.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global state for inter-ray coherency
reference_buffer = {
    'cells': [],       # List of voxel coordinates visited by last ray
    'has_intersection': [] # Boolean flags corresponding to cells
}

def process_ray_bundle(rays, volume, iso_threshold):
    for ray in rays:
        # Step 1: Check if we can reuse reference buffer
        if reference_buffer is not empty:
            # Identify cells in the current ray's path that are in the reference buffer
            common_cells = intersect_path_with_buffer(ray.path, reference_buffer['cells'])
            
            # Step 2: Skip intersection tests for cells known to be empty
            for cell in common_cells:
                if not reference_buffer['has_intersection'][cell]:
                    continue # Skip expensive intersection test
                
                # Step 3: Perform intersection test only if flag indicates potential hit
                if check_ray_surface_intersection(ray, cell, iso_threshold):
                    render_intersection(ray, cell)
                    # Update buffer with new path for next ray
                    update_reference_buffer(ray.path, cell)
                    break
            else:
                # If no intersection found in cached cells, do full traversal
                full_traversal_and_render(ray, volume, iso_threshold)
                update_reference_buffer(ray.path, None) # No intersection found
        else:
            # First ray: full traversal
            full_traversal_and_render(ray, volume, iso_threshold)
            update_reference_buffer(ray.path, None)

def update_reference_buffer(path, last_intersection_cell):
    reference_buffer['cells'] = path
    # Mark cells: True if last_intersection_cell is in this cell's range, else False
    # Simplified: flag is set for the specific cell where intersection occurred
    reference_buffer['has_intersection'] = {cell: (cell == last_intersection_cell) for cell in path}
```

**Parameters and Edge Cases:**
*   **Bundle Size:** The technique is most effective when rays are processed in spatially coherent bundles (e.g., scanlines). [S1, unofficial]
*   **Precision:** Fast voxel traversal algorithms are used to determine cell boundaries. [S1, unofficial]
*   **Pitfall:** If the scene changes significantly between rays (e.g., high-frequency details), the buffer may cause missed intersections if not updated correctly. Distance coding is not used as it requires recomputation for new iso-values. [S1, unofficial]

## Subtopics

### [Marching Caches](10066-marching-caches.md)
An explicit data caching strategy that maintains a shared cache around a bundle of rays, incrementally updated as the ray front advances. It reduces global memory bandwidth and allows for higher-order reconstruction filters, operating on GPUs by synchronizing ray bundles to share fast memory.

## Operating characteristics
*   **Performance:** Accelerates rendering by approximately 60 percent compared to reference cell-based first-hit ray casting. [S1, unofficial]
*   **Complexity:** Time complexity is reduced by skipping intersection tests for coherent cells. Memory complexity is low, requiring only a buffer proportional to the ray path length.
*   **Strengths:** Significant speedup for scenes with smooth iso-surfaces and coherent ray paths. Reduces redundant computation.
*   **Limitations:** Effectiveness depends on the coherence between adjacent rays. May not benefit highly fragmented or noisy volumes. Not suitable for dynamic iso-values without recomputation. [S1, unofficial]

## Background
The technique was introduced by Neubauer, Mroz, Hauser, and Wegenkittl in the paper "Cell-Based First-Hit Ray Casting." It was implemented and tested on a chest dataset (2266 data set) for virtual angioscopy and stent visualization. The reference implementation used fast voxel traversal and ray-surface intersection tests. [S1, unofficial]

## Sources
[S1] (PDF) Cell-Based First-Hit Ray Casting - ResearchGate, https://www.researchgate.net/publication/2521319_Cell-Based_First-Hit_Ray_Casting, fetched 2026-10-27 (unofficial)
