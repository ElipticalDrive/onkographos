# Parallel Sphere Rendering with Global Z-Buffer Reduction

Also called: CM-5/T3D Sphere Renderer, Parallel Sphere Rendering

## Overview
This method performs parallel volume rendering of atomic data by treating atoms as opaque spheres and resolving visibility through a global z-buffer reduction across distributed processors [S1, unofficial]. It solves the problem of correct front-to-back compositing in a distributed memory environment without requiring a global sort of all primitives, which is computationally prohibitive at scale [S1, unofficial]. Unlike hierarchical approaches that rely on spatial partitioning for load balancing, this method uses a trivial load-balancing strategy where each processor renders a disjoint subset of the atom list, relying on the global reduction to resolve occlusion [S1, unofficial]. It is distinct from standard ray-casting volume rendering by explicitly rasterizing geometric primitives (spheres) rather than sampling a scalar field [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   A dataset of $M$ atoms, where each atom is defined by position $(x, y, z)$ and radius $r$.
*   Image dimensions $W \times H$ (pixels).
*   Number of processors $P$.
*   Camera parameters (viewpoint, projection matrix).

**Outputs:**
*   A final $W \times H$ RGB image buffer.
*   A final $W \times H$ z-buffer (depth buffer).

### Data Structures
Each processor maintains local buffers for the current chunk of atoms:
1.  **Local Z-Buffer:** An array of size $W \times H$. Stores the depth ($z$) of the closest primitive rendered to each pixel by this processor. Initialized to $\infty$ (or maximum representable depth) for new chunks, or retained from previous chunks if processing partial datasets [S1, unofficial].
2.  **Local Image Buffer:** An array of size $W \times H$. Stores the RGB color of the closest primitive. Initialized to black $(0,0,0)$ for new chunks [S1, unofficial].
3.  **Composite Z-Buffer:** An array of size $W \times H$. Used to store the result of the global minimum reduction [S1, unofficial].
4.  **Composite Image Buffer:** An array of size $W \times H$. Used to store the result of the global maximum reduction [S1, unofficial].

**T3D Specific Packing:**
On the T3D architecture, the z-buffer and RGB color are combined into a single 64-bit integer to optimize reduction operations [S1, unofficial].
*   **Bits 63–24 (40 bits):** Depth value ($z$). Most significant bits.
*   **Bits 23–0 (24 bits):** RGB color value. Least significant bits.
This packing allows a single integer maximum reduction to simultaneously resolve depth occlusion and select the correct color [S1, unofficial].

### Parameters
*   **Tessellation Factor:** For comparison benchmarks, an SGI sphere drawing routine uses a tessellation factor of 3 with an octahedral tessellation method [S1, unofficial].
*   **Projection:** Orthogonal projection is used for bit-mapped sphere primitives on SGI hardware [S1, unofficial].

### Algorithm Steps

1.  **Data Distribution:**
    *   If parallel disk I/O is unavailable (e.g., T3D Alpha processors), Processor 0 reads the atom data from disk and distributes it to other processors [S1, unofficial].
    *   Distribute the atom list among $P$ processors. Each processor $p$ receives a subset of atoms $A_p$ [S1, unofficial].

2.  **Local Rendering:**
    *   For each processor $p$, initialize local Z-buffer to $\infty$ and Image buffer to black if starting a new chunk, or retain previous values if continuing [S1, unofficial].
    *   For each atom in $A_p$:
        *   Project the sphere onto the image plane.
        *   Rasterize the sphere (or its tessellation approximation).
        *   For each pixel covered by the sphere, calculate the depth $z_{local}$.
        *   If $z_{local} < Z_{buffer}[pixel]$, update $Z_{buffer}[pixel] = z_{local}$ and $Image_{buffer}[pixel] = color$ [S1, unofficial].

3.  **Global Z-Buffer Reduction (Minimum):**
    *   Perform a global reduction on the Z-buffer across all processors using the **minimum** operator.
    *   Result is stored in `Composite_Z_Buffer`.
    *   For each pixel $i$:
        $$ Z_{global}[i] = \min_{p=0}^{P-1} (Z_{local}^{(p)}[i]) $$
    *   This identifies the globally closest depth for each pixel [S1, unofficial].

4.  **Visibility Masking:**
    *   Each processor iterates over its local pixels.
    *   For each pixel $i$:
        *   If $Z_{local}^{(p)}[i] == Z_{global}[i]$, the pixel is visible (closest). Keep the color $C_{local}^{(p)}[i]$.
        *   Otherwise, the pixel is obscured by a closer pixel on another processor. Set $C_{local}^{(p)}[i] = (0, 0, 0)$ (black/minimal value) [S1, unofficial].

5.  **Global Image Reduction (Maximum):**
    *   Perform a global reduction on the Image buffer across all processors using the **maximum** operator.
    *   Result is stored in `Composite_Image_Buffer`.
    *   For each pixel $i$:
        $$ C_{global}[i] = \max_{p=0}^{P-1} (C_{local}^{(p)}[i]) $$
    *   Since obscured pixels are set to black (minimal value), the maximum operator selects the color of the visible pixel [S1, unofficial].
    *   After this step, every processor holds an identical image for the current chunk [S1, unofficial].

6.  **Iteration:**
    *   If the dataset does not fit in memory, read the next portion of atoms.
    *   Repeat steps 2–5 using the existing (non-empty) Z-buffer and Image buffers from the previous iteration [S1, unofficial].
    *   If rendering in tiles, repeat the entire process for the next tile until the full image is rendered [S1, unofficial].

7.  **Output:**
    *   Processor 0 writes the final image to disk [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_sphere_rendering(atoms, width, height, num_processors, my_rank, num_atoms):
    # 1. Data Distribution
    # Processor 0 reads file and distributes chunks.
    # Each processor gets a list of atoms assigned to it.
    local_atoms = distribute_atoms(atoms, my_rank, num_processors)

    # Initialize buffers
    # If this is the first chunk, init to infinity/black.
    # If subsequent chunk, retain previous values.
    if is_first_chunk:
        z_buffer = array_of_size(width * height, value=INFINITY)
        image_buffer = array_of_size(width * height, value=BLACK)
    else:
        z_buffer = previous_z_buffer
        image_buffer = previous_image_buffer

    # 2. Local Rendering
    for atom in local_atoms:
        # Project sphere to screen space
        screen_coords = project_sphere(atom, camera_params)
        
        # Rasterize sphere
        for pixel in screen_coords:
            z_val = calculate_depth(atom, pixel)
            color_val = atom.color
            
            # Local z-buffer test
            if z_val < z_buffer[pixel]:
                z_buffer[pixel] = z_val
                image_buffer[pixel] = color_val

    # 3. Global Z-Buffer Reduction (Minimum)
    # Uses MPI_Allreduce or equivalent with MIN operator
    composite_z_buffer = global_reduce(z_buffer, operator=MIN)

    # 4. Visibility Masking
    for pixel in range(width * height):
        if z_buffer[pixel] != composite_z_buffer[pixel]:
            # Obscured by another processor's closer pixel
            image_buffer[pixel] = BLACK

    # 5. Global Image Reduction (Maximum)
    # Uses MPI_Allreduce or equivalent with MAX operator
    # On T3D, this is merged with step 3 using 64-bit packing
    composite_image_buffer = global_reduce(image_buffer, operator=MAX)

    # Update buffers for next chunk if necessary
    previous_z_buffer = composite_z_buffer
    previous_image_buffer = composite_image_buffer

    # 6. Check for more data
    if has_more_atoms():
        return parallel_sphere_rendering(next_chunk, width, height, num_processors, my_rank, num_atoms)
    else:
        return composite_image_buffer
```

### Edge Cases and Pitfalls
*   **I/O Bottleneck:** On the T3D, parallel disk I/O is not available to Alpha processors. If all processors attempt to read the data file, the YMP host becomes overloaded. The solution is to have Processor 0 read the data and distribute it [S1, unofficial].
*   **Precision:** The T3D packing relies on 40 bits for depth and 24 bits for color. Ensure depth values fit within 40 bits to avoid overflow into the color channel [S1, unofficial].
*   **Degenerate Input:** If multiple processors render the exact same depth for a pixel, the visibility mask (step 4) will keep the pixel for all such processors. The subsequent maximum reduction (step 5) will select one color. If colors are identical, the result is correct. If colors differ (e.g., due to lighting variations), the result is non-deterministic but visually consistent with "closest" logic [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Compositing Time (CM-5):** The compositing operation takes $O(N \log P)$ time, where $N$ is the number of pixels and $P$ is the number of processors [S1, unofficial].
*   **Compositing Time (T3D):** The Binary Swap compositing scheme runs in $O(N/2 + N/4 + N/8 + \dots + N/P)$ time. As $P \to \infty$, this simplifies to $O(N)$ [S1, unofficial]. This is optimal for a dense image distribution on $P$ processors [S1, unofficial].
*   **Memory:** Each processor stores $O(N)$ for the z-buffer and image buffer, where $N = W \times H$.

### Measured Performance
Rendering rates for 5 million spheres on a $1024 \times 1024$ image [S1, unofficial]:

| Platform | With I/O (spheres/sec) | Computation Only (spheres/sec) |
| :--- | :--- | :--- |
| T3D | 660,000 | 1,689,000 |
| CM-5 | 318,000 | 334,000 |
| SGI Onyx (Tessellated) | 19,000 | 19,000 |
| SGI Onyx (Bit-mapped) | 79,000 | 87,000 |

*Note: SGI times do not include data transfer time from SDA to local disk, which takes several minutes over Ethernet [S1, unofficial].*

### Strengths and Limitations
*   **Strengths:**
    *   High performance on massively parallel supercomputers (MPPs) [S1, unofficial].
    *   Portable to other supercomputers or workstation clusters using basic message passing primitives [S1, unofficial].
    *   Does not rely on hardware-specific rendering engines, allowing integration with polygonal and volumetric renderers [S1, unofficial].
    *   T3D optimization merges two reductions and a comparison into one operation, reducing overhead [S1, unofficial].
*   **Limitations:**
    *   Trivial load balancing may lead to imbalance if atom distribution is non-uniform (e.g., varying radii) [S1, unofficial].
    *   Global reductions can become a bottleneck if the network topology is not optimized (though Binary Swap mitigates this) [S1, unofficial].
    *   Requires significant memory for z-buffers if image resolution is very high [S1, unofficial].

## Background

### Origin and History
The method was developed for the CM-5 and later ported to the T3D supercomputers [S1, unofficial]. The CM-5 version used two separate global reductions (min for z-buffer, max for image) [S1, unofficial]. The T3D version improved upon this by merging the z-buffer comparison and color compositing into a single 64-bit integer maximum reduction operation, utilizing the Binary Swap compositing scheme [S1, unofficial].

### Usage
The algorithm is designed for rendering atomic data (spheres) in scientific visualization. It is particularly suited for environments where global sorting is too expensive and hardware-accelerated rasterization is not available or portable [S1, unofficial]. It serves as a building block for hybrid renderers that combine polygon, sphere, and volume rendering [S1, unofficial].

## Sources

[S1] Parallel sphere rendering, https://digital.library.unt.edu/ark:/67531/metadc678482/?q=parallel volume rendering, fetched 2026-10-29 (unofficial)
