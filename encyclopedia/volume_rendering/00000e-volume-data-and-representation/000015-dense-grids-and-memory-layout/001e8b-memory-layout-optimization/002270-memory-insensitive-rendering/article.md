# Memory-Insensitive Rendering


## Overview
Memory-Insensitive Rendering (MIR) is an out-of-core technique for rendering arbitrarily large unstructured volumetric grids (specifically tetrahedral meshes) that requires only a constant, minimal amount of main memory [S1, unofficial]. It solves the problem of rendering datasets exceeding available RAM by avoiding random access to the source geometry, relying instead on sequential disk scans and external sorting [S1, unofficial]. Unlike sibling methods that optimize in-memory data layouts (e.g., [Voxel Buffer Memory Layout and Loop Optimization](1598-voxel-buffer-memory-layout-and-loop-optimization.md) or [Volume Interleaving](2174-volume-interleaving.md)), MIR prioritizes memory footprint over rendering speed, making it suitable for machines with severe memory constraints [S1, unofficial].

## How it works
**Inputs:**
1. A normalized tetrahedral grid file [S1, unofficial].
2. A transformation matrix [S1, unofficial].
3. Screen resolution (width $W$, height $H$) [S1, unofficial].
4. Transfer functions for lighting and compositing [S1, unofficial].

**Outputs:**
1. A rendered image buffer [S1, unofficial].

**Data Structures:**
* **Vertex File:** Binary file containing vertex positions $(x, y, z)$ and scalar field values [S1, unofficial].
* **Tetrahedra File:** Binary file containing tetrahedron definitions. In the normalized state, each record contains the actual vertex values (positions and scalars) for all 4 vertices, rather than indices [S1, unofficial].
* **Intersection File:** Binary file containing records of ray-cell intersections. Each record consists of:
  * Pixel ID $p_i$ (integer) [S1, unofficial].
  * Depth $z$ (float) [S1, unofficial].
  * Interpolated scalar field value for lighting (float) [S1, unofficial].

**Preprocessing (Normalization):**
If the input uses an indexed format (vertices with coordinates/scalars, tetrahedra with integer indices), the data must be dereferenced to create the normalized files [S1, unofficial]. This is performed in four passes, one for each vertex index of the tetrahedra:
1. Create two binary files: one for vertices, one for tetrahedra (initially with indices) [S1, unofficial].
2. For the current vertex index $k \in \{0, 1, 2, 3\}$:
   * Externally sort the tetrahedra file by the $k$-th vertex index using external mergesort. Time complexity: $O(t \log t)$, where $t$ is the number of tetrahedra [S1, unofficial].
   * Perform a synchronous scan of the vertex file and the sorted tetrahedra file. Read one record at a time from each. Since the tetrahedra are sorted by vertex index, the corresponding vertex data can be retrieved sequentially. Output the dereferenced position and scalar field values into a new temporary tetrahedra file. Time complexity: $O(v + t)$, where $v$ is the number of vertices [S1, unofficial].
3. Replace the original tetrahedra file with the new file. Repeat for all four indices.
4. The final tetrahedra file contains $t$ records, each with the actual values of its four vertices [S1, unofficial].

**Rendering Algorithm:**

1. **Ray Intersection Generation:**
   * Read each tetrahedron from the normalized file sequentially [S1, unofficial].
   * Transform the tetrahedron vertices using the specified transformation matrix [S1, unofficial].
   * Compute ray intersections for all pixels in the screen resolution [S1, unofficial].
   * For each pixel $\rho_i$ intersecting the cell in the interval $[z_0, z_1]$:
     * Output two records: $(\rho_i, z_0)$ and $(\rho_i, z_1)$ to the Intersection File [S1, unofficial].
     * Save an interpolated scalar field value for lighting calculations [S1, unofficial].
     * Memory usage is minimal: only the cell description and temporary storage for one intersection are held in RAM [S1, unofficial].
     * Disk space required is proportional to the number of ray stabbings [S1, unofficial].

2. **External Sort:**
   * Sort the Intersection File using an external sort algorithm [S1, unofficial].
   * The comparison function sorts primarily by Pixel ID $p_i$ (ascending), and secondarily by depth $z$ (ascending) [S1, unofficial].
   * Result: Records for a specific pixel are grouped sequentially and ordered by increasing depth (front-to-back) [S1, unofficial].

3. **Compositing:**
   * Traverse the sorted Intersection File sequentially [S1, unofficial].
   * For each pixel, accumulate the samples in the order they appear (which is front-to-back) [S1, unofficial].
   * Apply transfer functions to light and composite the samples [S1, unofficial].
   * Write the final color value to the image buffer [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def memory_insensitive_rendering(normalized_tet_file, transform_matrix, width, height, transfer_func):
    """
    Renders a volume using the Memory-Insensitive Rendering technique.
    Assumes normalized_tet_file contains dereferenced tetrahedra.
    """
    intersection_file = open("intersections.bin", "wb")
    
    # Step 1: Generate Intersections
    for tet in read_tetrahedra(normalized_tet_file):
        transformed_verts = [transform(v, transform_matrix) for v in tet.vertices]
        for pixel_id in range(width * height):
            ray = get_ray_for_pixel(pixel_id, width, height)
            z0, z1, scalar = compute_ray_tet_intersection(transformed_verts, ray)
            if z0 is not None:
                # Output two records: (pixel_id, z0) and (pixel_id, z1) with scalar
                write_record(intersection_file, pixel_id, z0, scalar)
                write_record(intersection_file, pixel_id, z1, scalar)
                
    intersection_file.close()
    
    # Step 2: External Sort
    # Sort by (pixel_id ASC, z ASC)
    sorted_file_path = external_sort("intersections.bin", key=lambda rec: (rec.pixel_id, rec.z))
    
    # Step 3: Compositing
    image = create_image(width, height)
    current_pixel = -1
    pixel_samples = []
    
    for rec in read_sorted_records(sorted_file_path):
        if rec.pixel_id != current_pixel:
            if current_pixel != -1:
                render_pixel(image, current_pixel, pixel_samples, transfer_func)
            current_pixel = rec.pixel_id
            pixel_samples = []
        pixel_samples.append(rec)
        
    if current_pixel != -1:
        render_pixel(image, current_pixel, pixel_samples, transfer_func)
        
    return image

def render_pixel(image, pixel_id, samples, transfer_func):
    """
    Composites samples for a single pixel using front-to-back alpha compositing.
    """
    color = (0.0, 0.0, 0.0, 0.0)
    alpha = 0.0
    for sample in samples:
        c, a = transfer_func(sample.scalar)
        color = color + (c * a * (1 - alpha))
        alpha = alpha + a * (1 - alpha)
    image[pixel_id] = color

# Helper functions
def read_tetrahedra(path):
    with open(path, "rb") as f:
        while chunk := f.read(128): yield Tetrahedron.from_bytes(chunk)

def transform(v, m): return apply_matrix(v, m)
def get_ray_for_pixel(pid, w, h): return Ray(origin=(0,0,0), direction=(0,0,1)) # [G]
def compute_ray_tet_intersection(verts, ray): return (z0, z1, scalar) # [G]
def write_record(f, pid, z, s): f.write(struct.pack("ifff", pid, z, s, 0.0))
def external_sort(path, key): return path + ".sorted" # [S1, unofficial]
def create_image(w, h): return [[0.0]*4 for _ in range(w*h)]
def read_sorted_records(path):
    with open(path, "rb") as f:
        while chunk := f.read(20): yield Record.from_bytes(chunk)
```

**Edge Cases and Pitfalls:**
* **Precision:** Floating-point precision in depth sorting must be sufficient to maintain correct front-to-back ordering, especially for thin cells or grazing rays.
* **Disk I/O:** The performance is heavily dependent on disk I/O speed. External sorting is the most time-consuming step [S1, unofficial].
* **Memory Usage:** The algorithm assumes a constant, small amount of RAM. If the system runs out of memory for the external sort buffers, the sort will fail or degrade significantly.

## Operating characteristics
* **Time Complexity:** Normalization sorting takes $O(t \log t)$ [S1, unofficial]. Synchronous scan takes $O(v + t)$ [S1, unofficial]. The rendering phase involves one pass over the geometry ($O(t \cdot \text{rays})$), one external sort ($O(N \log N)$ where $N$ is the number of intersections), and one pass over the sorted intersections ($O(N)$).
* **Memory Complexity:** $O(1)$ with respect to the dataset size. Memory usage is constant, depending only on the size of a single tetrahedron and temporary buffers for sorting/compositing [S1, unofficial].
* **Disk Space:** Proportional to the number of ray stabbings (intersections) [S1, unofficial].
* **Strengths:** Can render arbitrarily large datasets on machines with very little memory [S1, unofficial]. Simple implementation. Good load balancing characteristics [S1, unofficial].
* **Limitations:** Substantially slower than in-core techniques (up to an order of magnitude) due to disk I/O and external sorting [S1, unofficial]. Does not exploit ray coherence [S1, unofficial].
* **When to use:** When rendering very large unstructured volumetric datasets on systems with severe memory constraints.
* **When not to use:** When rendering speed is critical, or when sufficient memory is available for in-core techniques.

## Background
Memory-Insensitive Rendering was introduced as an out-of-core rendering technique for large, unstructured grids [S1, unofficial]. It is based on an external memory version of techniques previously considered by [9, 6] (cited in the source) [S1, unofficial]. The technique was noted as being discarded by [9] due to inefficiency from lack of ray coherence, but used by [6] for its load balancing characteristics after optimization [S1, unofficial]. The method was implemented and tested on large datasets, demonstrating its ability to render arbitrarily large images with minimal memory [S1, unofficial].

## Sources
[S1] Out-of-core rendering of large, unstructured grids, https://www.researchgate.net/publication/3208958_Out-of-core_rendering_of_large_unstructured_grids, fetched 2026-10-26 (unofficial)
