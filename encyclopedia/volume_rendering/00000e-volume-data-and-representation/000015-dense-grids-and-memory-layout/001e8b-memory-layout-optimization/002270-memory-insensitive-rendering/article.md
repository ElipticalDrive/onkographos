# Memory-Insensitive Rendering


## Overview
Memory-Insensitive Rendering (MIR) is an out-of-core volume rendering technique designed to render arbitrarily large unstructured volumetric grids on machines with limited main memory [S1, unofficial]. It solves the problem of memory exhaustion by avoiding random access to the dataset, relying instead on sequential disk I/O and external sorting [S1, unofficial]. Unlike sibling methods that optimize in-memory cache locality [1798-memory-layout-optimization.md], MIR prioritizes constant memory usage over rendering speed, making it suitable for datasets that exceed available RAM by orders of magnitude [S1, unofficial].

## How it works
MIR operates on unstructured tetrahedral grids. The method consists of a preprocessing normalization phase followed by a three-step rendering pipeline: intersection generation, external sorting, and accumulation.

### Inputs and Outputs
*   **Inputs:**
    *   A normalized tetrahedron file containing $t$ tetrahedra, where each tetrahedron is defined by 4 vertices with actual position $(x, y, z)$ and scalar field values (not indices) [S1, unofficial].
    *   A vertex file containing $v$ vertices with position and scalar values [S1, unofficial].
    *   A transformation matrix for spatial alignment [S1, unofficial].
    *   Screen resolution (defining the pixel grid) [S1, unofficial].
    *   Transfer functions for lighting and compositing [S1, unofficial].
*   **Outputs:**
    *   A rendered image.
    *   Intermediate files: normalized tetrahedron file, ray intersection file [S1, unofficial].

### Data Structures
1.  **Vertex Record:** Stores 3D coordinates $(x, y, z)$ and scalar field value.
2.  **Tetrahedron Record (Normalized):** Stores 4 Vertex Records. No indices are present.
3.  **Ray Intersection Record:** Stores:
    *   Pixel ID $p_i$ (integer).
    *   Entry depth $z_0$ (float).
    *   Exit depth $z_1$ (float).
    *   Interpolated scalar field value for lighting (float) [S1, unofficial].

### Preprocessing: Normalization
If the input data uses indexed mesh format (vertices with coordinates, tetrahedra with 4 integer indices), it must be normalized to dereference pointers [S1, unofficial]. This process ensures that each tetrahedron record contains actual vertex data, allowing sequential access without random lookups [S1, unofficial].

1.  Create two binary files: `vertices.bin` (list of $v$ vertices) and `tetrahedra.bin` (list of $t$ tetrahedra with indices) [S1, unofficial].
2.  For each of the 4 vertex indices in `tetrahedra.bin`:
    a.  Perform an external mergesort on `tetrahedra.bin` based on the current vertex index. Time complexity: $O(t \log t)$ [S1, unofficial].
    b.  Perform a synchronous scan of `vertices.bin` and the sorted `tetrahedra.bin`. Read one record at a time from both files. Match the sorted index from the tetrahedron file with the corresponding vertex in the vertex file. Output the dereferenced position and scalar field values into a new temporary tetrahedron file [S1, unofficial].
    c.  Replace the original `tetrahedra.bin` with the new file.
3.  Repeat step 2 for all 4 indices. The final `tetrahedra.bin` contains $t$ records, each with 4 vertices containing actual values. Time complexity for the scan phase: $O(v + t)$ [S1, unofficial].

### Rendering Steps

**Step 1: Ray Intersection Generation**
Read each tetrahedron from the normalized file sequentially.
1.  Transform the tetrahedron vertices using the input transformation matrix [S1, unofficial].
2.  Compute ray intersections for all pixels in the screen resolution. For each pixel $\rho_i$ that intersects the tetrahedron in the depth interval $[z_0, z_1]$:
    a.  Create a Ray Intersection Record containing $(p_i, z_0, z_1, \text{scalar})$. The scalar value is interpolated for lighting calculations [S1, unofficial].
    b.  Write the record to the `intersections.bin` file.
3.  Memory usage is minimal, requiring only storage for one cell description and temporary intersection variables [S1, unofficial]. Disk space usage is proportional to the number of ray stabbings [S1, unofficial].

**Step 2: External Sort**
Sort the `intersections.bin` file using an external sort algorithm.
1.  Define the comparison function:
    *   Primary key: Pixel ID $p_i$ (ascending).
    *   Secondary key: Depth $z$ (ascending).
2.  The sort ensures that all records for a specific pixel are grouped together and ordered by increasing depth (front-to-back) [S1, unofficial].

**Step 3: Accumulation**
Traverse the sorted `intersections.bin` file sequentially.
1.  Group records by Pixel ID $p_i$.
2.  For each pixel, iterate through its records in order (which is already front-to-back due to the sort).
3.  Apply the transfer functions to light and composite the samples using standard volume rendering accumulation (e.g., front-to-back alpha blending) [S1, unofficial].
4.  Output the final color for the pixel [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def memory_insensitive_rendering(vertices_file, tetrahedra_file, transform_matrix, resolution, transfer_functions):
    # Preprocessing: Normalize tetrahedra
    # 1. Create initial files if not already normalized
    # 2. Dereference indices for all 4 vertices
    normalized_tetrahedra_file = normalize_tetrahedra(tetrahedra_file, vertices_file)
    
    # Step 1: Generate Ray Intersections
    intersections_file = open("intersections.bin", "wb")
    with open(normalized_tetrahedra_file, "rb") as tet_file:
        for tet in tet_file:
            # Transform vertices
            transformed_verts = [transform(v, transform_matrix) for v in tet.vertices]
            
            # Compute intersections with all pixels
            # This step assumes a ray-casting approach against the tetrahedron
            for pixel_id in range(resolution.width * resolution.height):
                ray = get_ray_for_pixel(pixel_id, resolution)
                intersection = compute_tet_ray_intersection(transformed_verts, ray)
                
                if intersection.is_intersecting:
                    record = {
                        'pixel_id': pixel_id,
                        'z_entry': intersection.z0,
                        'z_exit': intersection.z1,
                        'scalar': interpolate_scalar(intersection.point, transformed_verts, tet.scalar_values)
                    }
                    write_record(intersections_file, record)
    intersections_file.close()
    
    # Step 2: External Sort
    # Sort primarily by pixel_id, secondarily by z_entry
    sorted_intersections_file = external_sort(intersections_file, key=lambda r: (r['pixel_id'], r['z_entry']))
    
    # Step 3: Accumulation
    image = create_image(resolution)
    with open(sorted_intersections_file, "rb") as sorted_file:
        current_pixel = None
        samples = []
        
        for record in sorted_file:
            if record['pixel_id'] != current_pixel:
                # Process previous pixel
                if current_pixel is not None:
                    image[current_pixel] = composite_samples(samples, transfer_functions)
                
                # Start new pixel
                current_pixel = record['pixel_id']
                samples = []
            
            samples.append(record)
        
        # Process last pixel
        if current_pixel is not None:
            image[current_pixel] = composite_samples(samples, transfer_functions)
            
    return image

def normalize_tetrahedra(tetrahedra_file, vertices_file):
    # Create binary files for vertices and tetrahedra
    # For each of the 4 indices in tetrahedra:
    #   1. External sort tetrahedra by index
    #   2. Synchronous scan vertices and sorted tetrahedra to dereference
    # Return path to normalized file
    pass

def compute_tet_ray_intersection(vertices, ray):
    # Compute intersection of ray with tetrahedron defined by vertices
    # Returns z0, z1, and intersection point if intersecting
    pass

def interpolate_scalar(point, vertices, scalars):
    # Barycentric interpolation of scalar values
    pass

def composite_samples(samples, transfer_functions):
    # Front-to-back compositing using transfer functions
    pass
```

### Parameters and Edge Cases
*   **Parameters:**
    *   Transformation matrix: Defines the spatial orientation of the volume [S1, unofficial].
    *   Screen resolution: Determines the number of rays and output pixels [S1, unofficial].
    *   Transfer functions: Define the mapping from scalar values to color and opacity [S1, unofficial].
*   **Edge Cases and Pitfalls:**
    *   **Disk Space:** The intermediate intersection file size is proportional to the number of ray stabbings. For dense volumes or high resolutions, this file can be very large, requiring significant disk space [S1, unofficial].
    *   **Sorting Time:** The external sort is the most time-consuming step. Performance depends on the efficiency of the external sort implementation and disk I/O bandwidth [S1, unofficial].
    *   **Precision:** Floating-point precision in depth calculations ($z_0, z_1$) affects the ordering in the sort. Incorrect ordering can lead to incorrect compositing [S1, unofficial].
    *   **Empty Pixels:** Pixels with no intersections should be handled correctly (e.g., background color) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:**
    *   Normalization: $O(t \log t)$ for sorting, $O(v + t)$ for scanning [S1, unofficial].
    *   Rendering: Dominated by the external sort step, which is generally the most time-consuming part of the algorithm [S1, unofficial].
*   **Memory Complexity:** $O(1)$ main memory usage (constant amount), as the algorithm processes data sequentially from disk [S1, unofficial].
*   **Strengths:**
    *   Can render arbitrarily large datasets on machines with very little memory [S1, unofficial].
    *   Simple implementation compared to in-core techniques [S1, unofficial].
    *   Good load balancing characteristics [S1, unofficial].
*   **Limitations:**
    *   Substantially slower than in-core techniques (up to an order of magnitude) due to disk I/O and sorting overhead [S1, unofficial].
    *   Does not exploit coherence between rays, which can lead to redundant computations [S1, unofficial].
    *   High disk space requirements for intermediate files [S1, unofficial].
*   **When to Use:**
    *   When the dataset size exceeds available RAM.
    *   When memory constraints are stricter than time constraints.
    *   For one-off rendering of very large datasets where preprocessing cost is acceptable [S1, unofficial].
*   **When Not to Use:**
    *   When real-time or interactive rendering is required.
    *   When disk I/O bandwidth is a bottleneck.
    *   When the dataset fits comfortably in RAM, in which case in-core methods are faster [S1, unofficial].

## Background
Memory-Insensitive Rendering was introduced in the paper "Out-of-core rendering of large, unstructured grids" [S1, unofficial]. The technique is based on an external memory version of techniques previously considered by [9, 6] (cited in the source, but not provided in the source list) [S1, unofficial]. The authors note that [9] discarded the technique as too inefficient because it did not use coherency between rays, while [6] used it for its good load balancing characteristics but had to optimize it to save space [S1, unofficial]. MIR is useful for rendering arbitrarily large images of unstructured grids where memory is the primary constraint [S1, unofficial].

## Sources
[S1] Out-of-core rendering of large, unstructured grids, https://www.researchgate.net/publication/3208958_Out-of-core_rendering_of_large_unstructured_grids, fetched 2026-10-26 (unofficial)
