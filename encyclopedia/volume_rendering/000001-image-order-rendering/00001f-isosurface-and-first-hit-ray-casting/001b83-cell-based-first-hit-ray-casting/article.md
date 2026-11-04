# Cell-Based First-Hit Ray Casting


## Overview
Cell-based first-hit ray casting is a volume rendering technique that accelerates iso-surface visualization by dividing the volume into cubic sub-volumes called macro-cells, storing encountered cells in a buffer to exploit inter-ray coherency, and avoiding unnecessary intersection tests for rays visiting the same cells [S1, S2, unofficial]. It performs iso-surfacing with interactive threshold adjustment, reducing average ray path lengths to only a few steps per pixel through volume subdivision and early termination [S1, unofficial]. It sets itself apart from conventional ray casting by processing only macro-cells containing iso-surface portions and from Surface Fitting techniques by not requiring dedicated acceleration hardware or pre-computed surface meshes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume dataset, a set of iso-value thresholds (e.g., in Hounsfield Units), a camera position and orientation, and an image resolution (e.g., 512x512) [S1, S2, unofficial].
*   **Output:** A 2D image where each pixel contains a color value determined by the first iso-surface intersection along the corresponding view ray [S1, S2, unofficial].

**Data Structures:**
1.  **Min-Max Octree:**
    *   The volume is organized as an octree.
    *   Each node represents a cubic sub-volume.
    *   Each node stores two scalar values: `min_val` and `max_val`, representing the minimum and maximum data values within that sub-volume [S1, S2, unofficial].
    *   Leaf nodes represent **macro-cells**.
2.  **Macro-Cell:**
    *   A cubic sub-volume consisting of $n^3$ data cells (or $(n+1)^3$ voxels) [S1, S2, unofficial].
    *   $n$ is a parameter typically between 3 and 10 [S1, S2, unofficial].
3.  **Cell Hit Buffer:**
    *   A buffer storing cells encountered along the last ray cast.
    *   Each entry includes a flag indicating if a portion of an iso-surface is contained in that cell [S1, unofficial].
    *   This buffer is used to skip intersection tests for subsequent rays visiting the same cells [S1, unofficial].

**Algorithm Steps:**

1.  **Octree Construction:**
    *   Divide the volume data into macro-cells of size $n \times n \times n$ cells.
    *   Build the min-max octree. For each node, compute the minimum and maximum data values of the sub-space it represents. Leaf nodes correspond to macro-cells [S1, S2, unofficial].

2.  **Octree Traversal and Macro-Cell Selection:**
    *   Traverse the octree starting from the root.
    *   For each node, check if any of the specified iso-value thresholds lies between the node's `min_val` and `max_val`.
    *   Only process the 8 child nodes if this condition is met. This effectively skips empty space [S1, S2, unofficial].
    *   When a leaf node (macro-cell) passes the check, it contains a part of the iso-surface.

3.  **Macro-Cell Trimming:**
    *   Trim the found macro-cell to the smallest cuboid containing its portions of the iso-surfaces. This reduces the area projected to the screen [S2, unofficial].

4.  **Projection and Rasterization:**
    *   Project the trimmed macro-cell onto the image plane.
    *   Rasterize the projection to identify all pixels covered by the macro-cell [S1, S2, unofficial].

5.  **Local Ray Casting:**
    *   For each pixel covered by the projection that has not yet been assigned a color (i.e., not yet hit by an iso-surface in the current frame), cast a local ray through the macro-cell [S1, S2, unofficial].
    *   The ray starts at the entry point into the macro-cell and stops when it hits an iso-surface or leaves the macro-cell [S1, unofficial].
    *   **Viewing Order:** Macro-cells and their child nodes must be processed in correct viewing order to ensure the first intersection determines the final pixel color. This is achieved by sorting child nodes according to the viewing vector during traversal, using a representative ray direction that intersects all sub-volumes [S1, S2, unofficial].

6.  **Voxel Traversal and Boundary Detection:**
    *   Use a fast voxel traversal algorithm (e.g., Amanatides and Woo) to detect cells pierced by the local ray within the macro-cell [S1, S2, unofficial].
    *   For each intersected cell, determine if it is a **boundary cell** (containing data values both above and below one of the specified thresholds) [S1, unofficial].
    *   **Coherency Check:** Before performing expensive intersection tests, check the Cell Hit Buffer. If the cell was encountered in the last ray and the buffer indicates an iso-surface portion is contained there, perform the intersection test; otherwise, skip it if the buffer indicates no surface [S1, unofficial].

7.  **Ray-Surface Intersection:**
    *   If a boundary cell is found, execute a ray-surface intersection test [S1, S2, unofficial].
    *   **Fast Intersection Algorithm:**
        *   Divide the ray path through the surface cell into two equal parts.
        *   Calculate the density values at the endpoints of each part.
        *   If an intersection exists (indicated by the density values), approximate the intersection point using linear interpolation [S1, unofficial].
        *   *Note:* An exact intersection algorithm (inverting trilinear interpolation) exists but is computationally expensive [S1, unofficial]. The fast method yields slightly less accurate results but is significantly cheaper [S1, unofficial].

8.  **Shading and Output:**
    *   If an intersection is found, calculate the gradient vector at the intersection point using trilinear interpolation of the volume data [S1, S2, unofficial].
    *   Calculate a color value from the gradient (e.g., using lighting models) [S1, S2, unofficial].
    *   Assign the color to the corresponding pixel.
    *   If the iso-surface is opaque, mark the pixel such that no more local rays are cast at this pixel during the current frame [S2, unofficial].

9.  **Early Scanline Termination (Optional Optimization):**
    *   Eliminate local rays that are highly probable not to find any intersections, further reducing ray counts [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cell_based_first_hit_ray_casting(volume, thresholds, camera, image_width, image_height, n):
    """
    volume: 3D array of scalar values
    thresholds: list of iso-values
    camera: object with position and direction vectors
    image_width, image_height: int
    n: int, macro-cell dimension (cells per side)
    """
    # 1. Build Min-Max Octree
    octree = build_min_max_octree(volume, n)
    
    # 2. Initialize Output and Coherency Buffer
    image = zeros((image_height, image_width, 3))
    hit_buffer = {} # Maps cell_id to boolean (has_surface)
    
    # 3. Traverse Octree in Viewing Order
    # Sort traversal order based on camera direction to ensure first-hit correctness
    nodes_to_process = [octree.root]
    
    while nodes_to_process:
        node = nodes_to_process.pop(0)
        
        # Check if node contains any iso-surface
        if not any(threshold in [node.min_val, node.max_val] or 
                   (node.min_val < threshold < node.max_val) for threshold in thresholds):
            continue
            
        if node.is_leaf():
            # Leaf node: Macro-cell
            macro_cell = node.data
            
            # Trim macro-cell to smallest cuboid containing iso-surfaces
            trimmed_cuboid = trim_macro_cell(macro_cell, thresholds)
            
            # Project trimmed cuboid to screen
            covered_pixels = project_to_screen(trimmed_cuboid, camera, image_width, image_height)
            
            # Cast local rays for covered pixels
            for pixel in covered_pixels:
                if image[pixel.y, pixel.x] is not empty:
                    continue # Pixel already hit
                
                ray = get_ray_for_pixel(pixel, camera)
                
                # Traverse voxels in the macro-cell
                cells_pierced = fast_voxel_traversal(ray, macro_cell)
                
                for cell in cells_pierced:
                    # Check coherency buffer
                    if cell.id in hit_buffer:
                        if not hit_buffer[cell.id]:
                            continue # Skip if buffer says no surface
                        
                    # Check if cell is a boundary cell
                    if is_boundary_cell(cell, thresholds):
                        # Perform fast ray-surface intersection
                        intersection_point = fast_ray_surface_intersection(ray, cell, thresholds)
                        
                        if intersection_point is not None:
                            # Calculate gradient via trilinear interpolation
                            gradient = trilinear_gradient(volume, intersection_point)
                            
                            # Compute color
                            color = compute_color(gradient, camera)
                            
                            # Assign color
                            image[pixel.y, pixel.x] = color
                            
                            # Mark pixel as hit
                            mark_pixel_hit(pixel)
                            
                            # Update coherency buffer for this cell
                            hit_buffer[cell.id] = True
                            break # Stop processing this ray for this pixel
        else:
            # Internal node: Add children to queue, sorted by viewing distance
            children = get_sorted_children(node, camera.direction)
            nodes_to_process.extend(children)
            
    return image

def build_min_max_octree(volume, n):
    """
    Recursively builds a min-max octree.
    Leaf nodes represent macro-cells of size n x n x n cells.
    """
    # Implementation details omitted for brevity, but involves:
    # 1. Dividing volume into n^3 blocks.
    # 2. Computing min/max for each block.
    # 3. Recursing if block is too large or needs subdivision.
    pass

def trim_macro_cell(macro_cell, thresholds):
    """
    Returns the smallest cuboid within the macro_cell that contains
    any cell crossing a threshold.
    """
    # Find min/max x, y, z indices of boundary cells
    pass

def project_to_screen(cuboid, camera, width, height):
    """
    Rasterizes the 3D cuboid onto the 2D image plane.
    Returns list of pixel coordinates.
    """
    pass

def fast_voxel_traversal(ray, macro_cell):
    """
    Uses Amanatides and Woo algorithm to find cells pierced by ray.
    """
    pass

def is_boundary_cell(cell, thresholds):
    """
    Returns True if cell contains values both above and below any threshold.
    """
    for t in thresholds:
        if cell.min_val < t < cell.max_val:
            return True
    return False

def fast_ray_surface_intersection(ray, cell, thresholds):
    """
    Fast approximation:
    1. Split ray segment in cell into two halves.
    2. Evaluate density at endpoints.
    3. If sign change, linearly interpolate intersection.
    """
    # Implementation based on S1 description
    pass

def trilinear_gradient(volume, point):
    """
    Computes gradient using trilinear interpolation of volume data.
    """
    pass

def compute_color(gradient, camera):
    """
    Standard lighting model (e.g., Phong) using gradient as normal.
    """
    pass
```

**Parameters:**
*   **$n$ (Macro-cell size):** The number of cells along one dimension of a macro-cell. Typical values range between 3 and 10 [S1, S2, unofficial].
*   **Iso-value Threshold:** Determines visibility. Values vary by application (e.g., 0 HU for soft tissue, 1100 HU for stents) [S1, unofficial].
*   **Image Resolution:** Fixed at 512x512 in reported experiments [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Macro-cell Size Trade-offs:** Small macro-cells increase octree traversal and projection time, split rays into more segments, and increase expensive ray initialization steps for rays traveling along iso-surfaces without intersecting. Large macro-cells reduce the effectiveness of early scanline termination because small critical regions are less likely to be detected [S1, unofficial].
*   **Viewing Order:** Ray segments must be processed in correct viewing order to ensure the first intersection determines the final pixel color. Failure to sort child nodes by viewing vector causes visibility artifacts [S1, S2, unofficial].
*   **Coherency Buffer:** Distance coding is not used because it would require recomputation for each new iso-value [S1, unofficial].
*   **Performance Degradation:** Conventional ray casting can be faster than cell-based first-hit ray casting when soft tissue (low threshold) hides the stent, because ray-surface intersection distances are very short, reducing empty space traversal time. Cell-based suffers here because almost all macro-cells must be processed and fine-structured tissue creates "peep holes" delaying termination [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of macro-cells intersecting the iso-surface and the local ray path length. The technique reduces average ray path lengths to only a few steps per pixel [S1, unofficial].
*   **Memory Complexity:** Requires storage for the min-max octree and the cell hit buffer. The octree size is proportional to the volume size divided by $n^3$ [S1, S2, unofficial].
*   **Measured Performance:**
    *   In virtual angioscopy (inside stented aorta) at 512x512 resolution: Minimum frame time 1.103s vs 1.783s (conventional); Average 1.291s vs 2.119s [S1, unofficial].
    *   At 450 HU threshold (bones/stent visible): Cell-based time 1.953s, Conventional 8.563s (Speedup 4.38x) [S1, unofficial].
    *   At 1100 HU threshold (only stent visible): Cell-based time 0.609s, Conventional 12.781s (Speedup 20.98x) [S1, unofficial].
    *   Ray path reduction (typical fly-through): Conventional 262,144 rays, 35.494 steps/pixel. Cell-based 1,147,347 rays, 15.626 steps/pixel. With optimizations (mt, est): 396,808 rays, 3.932 steps/pixel [S1, unofficial].
*   **Strengths:** Fast perspective iso-surfacing, interactive threshold adjustment, no dedicated acceleration hardware required, good cache coherency (each macro-cell accessed once per frame) [S1, unofficial].
*   **Limitations:** Slower than conventional ray casting when low thresholds hide surfaces behind dense tissue due to "peep holes" and high macro-cell processing overhead [S1, unofficial].
*   **When to Use:** Fast perspective iso-surfacing is required, especially for high thresholds where surfaces are sparse or for virtual endoscopy [S1, S2, unofficial].
*   **When Not to Use:** When low thresholds are used and surfaces are dense/hidden, or when exact intersection accuracy is critical and the fast approximation is insufficient [S1, unofficial].

## Background
Cell-based first-hit ray casting was introduced by André Neubauer, Lukas Mroz, H. Hauser, and R. Wegenkittl in 2002 [S1, unofficial]. It was developed as part of the AlVis project for visualization of aluminium foam samples from high-resolution industrial CT and successfully tested for post-implantation assessment of stent placement in virtual angioscopy [S1, unofficial]. The technique builds on conventional ray casting and fast voxel traversal algorithms (Amanatides and Woo) [S1, unofficial]. It addresses the limitations of Surface Fitting techniques which require dedicated hardware and lack interactive threshold adjustment [S1, unofficial].

## Sources
[S1] (PDF) Cell-Based First-Hit Ray Casting, https://www.researchgate.net/publication/2521319_Cell-Based_First-Hit_Ray_Casting, fetched 2026-10-27 (unofficial)
[S2] Cell-Based First-Hit Ray Casting, https://old.cescg.org/CESCG-2002/ANeubauer/node5.html, fetched 2026-10-29 (unofficial)
