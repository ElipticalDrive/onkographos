# Inter-Ray Coherency

Also called: Reference Ray Caster

## Overview
Inter-Ray Coherency accelerates volume ray casting by caching traversal results from a previously processed ray to skip redundant ray-surface intersection tests for subsequent rays [S1, unofficial]. It operates by storing cells encountered along the last ray and a flag indicating whether an iso-surface portion is contained within them, allowing rays visiting the same cells to bypass unnecessary intersection checks [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset, an iso-surface threshold, and rays originating from the eye point [S1, unofficial].
*   **Output:** A rendered image where rays terminate at the first iso-surface intersection or volume exit [S1, unofficial].

**Data Structures:**
*   **Reference Buffer:** Stores the state of the last processed ray [S1, unofficial].
    *   `cells`: A list of voxel coordinates encountered along the last ray's path [S1, unofficial].
    *   `has_iso`: A boolean flag for each cell indicating whether an iso-surface portion is contained within it [S1, unofficial].

**Algorithm Steps:**
1. **Initialize:** For the first ray, perform standard fast voxel traversal to find visited cells, then perform fast ray-surface intersection tests on those cells [S1, unofficial].
2. **Store State:** After processing a ray, save its visited cells and intersection flags into the Reference Buffer [S1, unofficial].
3. **Reuse for Next Ray:** For subsequent rays, traverse the volume to find visited cells [S1, unofficial].
4. **Check Buffer:** For each cell visited by the current ray that matches a cell in the Reference Buffer, check the corresponding `has_iso` flag [S1, unofficial].
5. **Skip or Test:** If the flag indicates no iso-surface portion, skip the ray-surface intersection test for that cell [S1, unofficial]. If the flag indicates an iso-surface portion, perform the fast ray-surface intersection test only for that cell [S1, unofficial].
6. **Early Termination:** If an intersection is found, terminate the ray and update the Reference Buffer with the new path and flags [S1, unofficial].
7. **Iterate:** Repeat for all rays [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inter_ray_coherency_render(rays, volume, iso_threshold):
    # Reference buffer state
    last_ray_cells = []
    last_ray_has_iso = []
    
    for ray in rays:
        # Step 1: Fast voxel traversal to get cells along the ray path
        current_cells = fast_voxel_traversal(ray, volume.bounds)
        
        # Step 2: Process cells using reference buffer
        for i, cell in enumerate(current_cells):
            # Check if cell was visited by the last ray
            if cell in last_ray_cells:
                idx = last_ray_cells.index(cell)
                # Step 3: Use flag to decide whether to run intersection test
                if not last_ray_has_iso[idx]:
                    continue  # Skip intersection test
                # Step 4: Run intersection test only if flag indicates iso-surface
                if ray_surface_intersection_test(ray, cell, iso_threshold):
                    render_output(ray, cell)
                    # Update buffer for next ray
                    last_ray_cells = current_cells
                    last_ray_has_iso = [False] * len(current_cells)
                    last_ray_has_iso[idx] = True
                    break
            else:
                # Cell not in buffer, must test
                if ray_surface_intersection_test(ray, cell, iso_threshold):
                    render_output(ray, cell)
                    last_ray_cells = current_cells
                    last_ray_has_iso = [False] * len(current_cells)
                    last_ray_has_iso[i] = True
                    break
        else:
            # No intersection found
            last_ray_cells = current_cells
            last_ray_has_iso = [False] * len(current_cells)
```

**Parameters and Edge Cases:**
*   **Distance Coding:** Not used, as it would require recomputation for each new iso-value [S1, unofficial].
*   **Test Conditions:** Evaluated on a 2266 dataset of a chest at 512x512 resolution for virtual angioscopy inside a stented aorta and stent visualization from outside the chest [S1, unofficial].
*   **Pitfall:** Effectiveness relies on spatial coherence between adjacent rays; deviations from the cached path require fallback to standard traversal [S1, unofficial].

## Subtopics

### [Marching Caches](10066-marching-caches.md)
An explicit data caching strategy that maintains a shared cache around a bundle of rays, incrementally updated as the ray front advances. It reduces global memory bandwidth and allows for higher-order reconstruction filters, operating on GPUs by synchronizing ray bundles to share fast memory.

### [Enforced Coherence](19842-enforced-coherence.md)
A tracing method that brings coherence to secondary rays by rearranging the ray-to-voxel mapping using sorting, enabling simultaneous intersection checks within a cell.

## Operating characteristics
*   **Performance:** Accelerates rendering by about 60 percent compared to a reference cell-based first-hit ray caster [S1, unofficial].
*   **Complexity:** Memory complexity is low, requiring only a buffer proportional to the ray path length [S1, unofficial]. Time complexity is reduced by skipping intersection tests for coherent cells [S1, unofficial].
*   **Strengths:** Significant speedup for scenes with smooth iso-surfaces and coherent ray paths by eliminating redundant computation [S1, unofficial].
*   **Limitations:** Requires recomputation if distance coding were used for new iso-values, which is why it is avoided [S1, unofficial].

## Background
The technique was introduced by Neubauer, Mroz, Hauser, and Wegenkittl in the paper "Cell-Based First-Hit Ray Casting" [S1, unofficial]. It was implemented and tested on a chest dataset for virtual angioscopy and stent visualization, using fast voxel traversal and fast ray-surface intersection tests [S1, unofficial].

## Sources
[S1] (PDF) Cell-Based First-Hit Ray Casting - ResearchGate, https://www.researchgate.net/publication/2521319_Cell-Based_First-Hit_Ray_Casting, fetched 2026-10-27 (unofficial)
