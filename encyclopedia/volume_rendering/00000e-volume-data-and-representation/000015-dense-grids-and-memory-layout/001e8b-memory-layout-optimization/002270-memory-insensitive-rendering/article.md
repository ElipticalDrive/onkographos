# Memory-Insensitive Rendering

## Overview
Memory-Insensitive Rendering (MIR) is a completely disk-based technique for rendering arbitrarily large unstructured volumetric grids on machines with limited main memory [S1, unofficial]. It solves the problem of rendering datasets that exceed available RAM by avoiding random access to the dataset, requiring only a small, constant amount of main memory [S1, unofficial].

## How it works
**Inputs:**
*   Normalized tetrahedron file (containing vertex positions and scalar values for each tetrahedron) [S1, unofficial].
*   Transformation matrix [S1, unofficial].
*   Screen resolution (defining pixel IDs $\rho_i$) [S1, unofficial].
*   Transfer functions for lighting and compositing [S1, unofficial].

**Outputs:**
*   Rendered image [S1, unofficial].

**Data Structures:**
1.  **Vertex File:** Binary file containing vertex records [S1, unofficial]. Each record holds the 3D position $(x, y, z)$ and scalar field value [S1, unofficial].
2.  **Tetrahedra File (Normalized):** Binary file containing $t$ records [S1, unofficial]. Each record holds the actual vertex values (positions and scalars) for the 4 vertices of a tetrahedron, rather than indices [S1, unofficial].
3.  **Intersection File:** Binary file containing records generated during Step 1 [S1, unofficial]. Each record contains:
    *   Pixel ID $\rho_i$ (integer) [S1, unofficial].
    *   Entry depth $z_0$ (float) [S1, unofficial].
    *   Exit depth $z_1$ (float) [S1, unofficial].
    *   Interpolated scalar field value (float, for lighting) [S1, unofficial].

**Preprocessing: Normalization**
If the input data uses an indexed mesh format (vertices with coordinates/scalars; tetrahedra with 4 integer indices), it must be normalized to dereference pointers [S1, unofficial].
1.  Create two binary files: one for vertices, one for tetrahedra [S1, unofficial].
2.  For each of the 4 vertex indices in the tetrahedra file, externally sort the tetrahedra file by that index using external mergesort [S1, unofficial]. This takes time $O(t \log t)$ [S1, unofficial].
3.  Perform a synchronous scan of the vertex file and the sorted tetrahedra file, reading one record at a time to output the dereferenced position and scalar field values [S1, unofficial]. This takes time $O(v + t)$ because references are sorted [S1, unofficial].
4.  Repeat steps 2-3 for all 4 vertex indices [S1, unofficial]. The final tetrahedra file contains $t$ records with actual vertex values [S1, unofficial].

**Rendering Steps:**
1.  **Ray Intersection Generation:** Read each tetrahedron from the normalized file [S1, unofficial]. Transform it with the specified transformation matrix [S1, unofficial]. Compute all ray intersections for the screen resolution [S1, unofficial]. For each pixel $\rho_i$ intersecting the cell in the interval $[z_0, z_1]$, output two records: $(\rho_i, z_0)$ and $(\rho_i, z_1)$ [S1, unofficial]. Also save an interpolated scalar field value for lighting [S1, unofficial]. Memory usage is minimal, holding only the cell description and temporary storage for one intersection [S1, unofficial].
2.  **External Sort:** Sort the intersection file using an external sort [S1, unofficial]. The compare function sorts primarily on pixel ID $\rho_i$, and secondarily on the depth of intersection $z$ [S1, unofficial]. This ensures records for a particular pixel are sequential and ordered by increasing depth [S1, unofficial].
3.  **Compositing:** Traverse the sorted file [S1, unofficial]. Use the transfer functions to light and composite the samples, which are already in the correct front-to-back order [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct

def memory_insensitive_rendering(normalized_tetra_file, transform_matrix, screen_resolution, transfer_functions):
    width, height = screen_resolution
    num_pixels = width * height
    intersection_file = "intersections.bin"
    sorted_file = "intersections_sorted.bin"

    # Step 1: Ray Intersection Generation
    with open(normalized_tetra_file, "rb") as tetra_f, open(intersection_file, "wb") as inter_f:
        while True:
            raw = tetra_f.read(16 * 4)  # 4 vertices * 4 floats (x,y,z,scalar)
            if not raw: break
            vertices = [list(struct.unpack("4f", raw[i*16:(i+1)*16])) for i in range(4)]
            transformed = [transform_vertex(v, transform_matrix) for v in vertices]

            for y in range(height):
                for x in range(width):
                    ray_origin = [float(x), float(y), 0.0]
                    ray_dir = [0.0, 0.0, 1.0]
                    hit = ray_tetra_intersection(transformed, ray_origin, ray_dir)
                    if hit:
                        z0, z1, scalar = hit
                        pixel_id = y * width + x
                        # Output two records per hit as specified: (rho_i, z0) and (rho_i, z1)
                        inter_f.write(struct.pack("4f", pixel_id, z0, z0, 0.0))
                        inter_f.write(struct.pack("4f", pixel_id, z1, z1, scalar))

    # Step 2: External Sort
    # Sort intersection_file primarily by pixel_id, secondarily by depth z.
    # Requires an external merge sort implementation [G].
    external_sort_on_disk(intersection_file, sorted_file, key_fields=[0, 1])

    # Step 3: Compositing
    image = [0.0, 0.0, 0.0] * num_pixels
    current_pixel = -1
    samples = []

    with open(sorted_file, "rb") as sorted_f:
        while True:
            raw = sorted_f.read(16)
            if not raw: break
            pixel_id, z, z2, scalar = struct.unpack("4f", raw)

            if pixel_id != current_pixel:
                if current_pixel != -1:
                    composite_pixel(image, current_pixel, samples, transfer_functions)
                current_pixel = pixel_id
                samples = []
            
            # First record of interval has z0, second has z1 and scalar
            if z == z2:  # Entry record
                samples.append(('entry', z))
            else:        # Exit record
                samples.append(('exit', z, scalar))

        if current_pixel != -1:
            composite_pixel(image, current_pixel, samples, transfer_functions)

    return image

def transform_vertex(v, M):
    """Applies 4x4 transformation matrix to homogeneous vertex [G]."""
    x, y, z, s = v
    w = M[3][0]*x + M[3][1]*y + M[3][2]*z + M[3][3]
    return [
        (M[0][0]*x + M[0][1]*y + M[0][2]*z + M[0][3]) / w,
        (M[1][0]*x + M[1][1]*y + M[1][2]*z + M[1][3]) / w,
        (M[2][0]*x + M[2][1]*y + M[2][2]*z + M[2][3]) / w,
        s
    ]

def ray_tetra_intersection(vertices, origin, direction):
    """Computes ray-tetrahedron intersection using standard slab method [G]."""
    # Simplified: returns (z0, z1, scalar) or None
    # Assumes orthographic projection along Z for disk-based simplicity
    min_z, max_z = float('inf'), float('-inf')
    scalars = []
    for v in vertices:
        if v[2] < min_z: min_z = v[2]
        if v[2] > max_z: max_z = v[2]
        scalars.append(v[3])
    if origin[2] < min_z or origin[2] > max_z:
        return None
    # Linear interpolation of scalar at midpoint for lighting
    scalar_val = sum(scalars) / 4.0
    return (min_z, max_z, scalar_val)

def composite_pixel(image, pixel_id, samples, transfer_functions):
    """Front-to-back alpha compositing using transfer functions [G]."""
    color = [0.0, 0.0, 0.0]
    alpha = 0.0
    entry_z = None
    exit_z = None
    
    for s in samples:
        if s[0] == 'entry':
            entry_z = s[1]
        elif s[0] == 'exit':
            exit_z = s[1]
            c, a = transfer_functions(s[2])
            color = [color[i] + c[i] * a * (1 - alpha) for i in range(3)]
            alpha = alpha + a * (1 - alpha)
            
    if alpha > 0:
        base = pixel_id * 3
        image[base] = color[0]
        image[base+1] = color[1]
        image[base+2] = color[2]

def external_sort_on_disk(input_path, output_path, key_fields):
    """Placeholder for external merge sort implementation [G]."""
    pass
```

## Operating characteristics
*   **Time Complexity:** Normalization sorting takes $O(t \log t)$ using external mergesort [S1, unofficial]. Synchronous scan takes $O(v + t)$ [S1, unofficial]. The rendering phase involves one scan over the grid, one external-memory sort, and a final accumulation scan [S1, unofficial].
*   **Memory Complexity:** Requires a small, constant amount of main memory [S1, unofficial].
*   **Disk Space:** Proportional to the number of actual ray stabbings between rays and cells [S1, unofficial].
*   **Strengths:** Can render arbitrarily large datasets on machines with very little memory [S1, unofficial]. Good load balancing characteristics [S1, unofficial].
*   **Limitations:** Substantially slower than in-core techniques like ZSWEEP, which can be up to an order of magnitude faster [S1, unofficial]. The external sort is generally the most time-consuming step [S1, unofficial].
*   **When to use:** When rendering unstructured grids that exceed available RAM and speed is not the primary constraint [S1, unofficial].
*   **When not to use:** When high performance is required and sufficient memory is available [S1, unofficial].

## Background
Memory-Insensitive Rendering was introduced as part of out-of-core rendering techniques for large, unstructured grids [S1, unofficial]. It is essentially an external memory version of techniques previously considered by [9, 6] [S1, unofficial]. While [9] discarded the technique as too inefficient due to lack of coherency between rays, [6] used it for its good load balancing characteristics, optimizing it to save space [S1, unofficial]. The method was implemented and tested on large datasets under different conditions [S1, unofficial].

## Sources
[S1] (PDF) Out-of-core rendering of large, unstructured grids, https://www.researchgate.net/publication/3208958_Out-of-core_rendering_of_large_unstructured_grids, fetched 2026-10-26 (unofficial)
