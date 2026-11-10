# Structural and Geometric Compression

Also called: Data Structure and Compression

## Overview
Structural and Geometric Compression is a method for compressing particle-based volume data by storing only the surface depth values, connectivity links, and normals for each pixel column, rather than the full volumetric grid. It solves the problem of high memory usage in particle rendering by exploiting the sparsity of particle systems, where most voxels are empty. Unlike sibling methods that compress the entire 3D coefficient space (e.g., [Value compression](6-value-compression.md)), this method operates on a 2.5D representation (depth layers per pixel), achieving high compression ratios by storing only the relevant surface geometry and its immediate topological connections.

## How it works
The method compresses particle data by extracting surface information into a compact data structure per pixel column.

**Inputs and Outputs:**
*   **Input:** A set of particle spheres defined by center and radius.
*   **Output:** A compressed bitstream containing depth values, neighbor links, and normals for each pixel column.

**Data Structures:**
For each pixel column (voxel column along the view ray), the following data is stored:
1.  **Depth Values:** Stored in 32-bit floating-point format [S1]. Multiple entries may exist per column for translucency (entry and exit depths).
2.  **Neighbor Links:** Links to the four direct spatial neighbors (up, down, left, right) of each depth value. Each link is stored in 1 byte, totaling 32 bits (4 bytes) per depth entry [S1].
3.  **Normals:** Compressed to a 32-bit word using octahedron normal encoding. This consists of two 16-bit normalized integers [S1].

**Steps:**
1.  **Voxelization:** Mark voxels that overlap the surface of each particle sphere. Only these voxels are processed [S1].
2.  **Depth Extraction:** For each pixel column, extract the depth values of the surface intersections.
3.  **Link Generation:** Generate links to the four direct neighbors for each depth value. These links allow on-the-fly computation of surface connectivity during rendering [S1].
4.  **Normal Estimation:** Apply a Sobel operator to estimate derivatives in two directions to generate normals [S1].
5.  **Compression:**
    *   Store depth values as 32-bit floats [S1].
    *   Store neighbor links as 1-byte integers per direction, totaling 32 bits per depth entry [S1].
    *   Encode normals using octahedron encoding (two 16-bit integers) into a 32-bit word [S1].
6.  **Rendering:**
    *   For frontmost surface rendering, read the first depth and normal entry of every pixel column [S1].
    *   For translucency, iterate over all depth and normal entries, applying a scattering and absorption model using ray segment lengths computed from the differences between adjacent entry and exit depths [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data structure for a single depth entry in a pixel column
class DepthEntry:
    depth: float32      # 32-bit float
    links: [uint8, uint8, uint8, uint8] # 4 bytes: up, down, left, right
    normal: uint32      # 32-bit octahedron encoded normal

def compress_particle_data(particles, image_width, image_height):
    # Initialize storage for each pixel column
    # Each column stores a list of DepthEntry objects
    column_data = [[[] for _ in range(image_height)] for _ in range(image_width)]
    
    # Step 1: Voxelization - mark voxels overlapping particle surfaces
    for particle in particles:
        voxels = get_voxels_overlapping_sphere(particle.center, particle.radius)
        for voxel in voxels:
            # Mark voxel as part of surface
            mark_voxel_surface(voxel)

    # Step 2: Depth Extraction and Link Generation
    for x in range(image_width):
        for y in range(image_height):
            # Get depth values for this column
            depths = get_depth_values_for_column(x, y)
            
            # Sort depths to ensure correct entry/exit order
            depths.sort()
            
            for i, depth in enumerate(depths):
                entry = DepthEntry()
                entry.depth = depth
                
                # Step 3: Generate links to neighbors
                # Links point to the index of the neighbor's depth entry in the same column
                # or a special value if no neighbor exists
                entry.links = [
                    get_neighbor_link(x, y, i, direction='up'),
                    get_neighbor_link(x, y, i, direction='down'),
                    get_neighbor_link(x, y, i, direction='left'),
                    get_neighbor_link(x, y, i, direction='right')
                ]
                
                # Step 4: Normal Estimation
                # Use Sobel operator on depth derivatives
                normal = estimate_normal_sobel(depths, i)
                entry.normal = octahedron_encode(normal)
                
                column_data[x][y].append(entry)
                
    return column_data

def render_frontmost(column_data):
    for x in range(image_width):
        for y in range(image_height):
            entries = column_data[x][y]
            if entries:
                # Read first depth and normal
                first_entry = entries[0]
                depth = first_entry.depth
                normal = octahedron_decode(first_entry.normal)
                # Render pixel at depth with normal
                render_pixel(x, y, depth, normal)

def render_translucent(column_data):
    for x in range(image_width):
        for y in range(image_height):
            entries = column_data[x][y]
            if len(entries) < 2:
                continue
            
            # Iterate over all depth entries
            for i in range(len(entries) - 1):
                entry_in = entries[i]
                entry_out = entries[i+1]
                
                # Compute ray segment length
                segment_length = entry_out.depth - entry_in.depth
                
                # Apply scattering and absorption model
                color = apply_scattering_absorption(
                    entry_in.normal, 
                    segment_length, 
                    particle_properties
                )
                accumulate_color(x, y, color)
```

**Parameters:**
*   **Smoothing Iterations:** The number of iterations for the 3x3 smoothing kernel depends on the distance of the surface point to the camera to maintain a constant world-space filter radius [S1].
*   **Link Size:** 1 byte per neighbor direction [S1].
*   **Normal Encoding:** Octahedron encoding with two 16-bit integers [S1].

**Edge Cases:**
*   **Empty Columns:** Pixel columns with no particle intersections should be handled gracefully (e.g., skip rendering or use background color).
*   **Boundary Links:** Links at the image boundaries must be handled to avoid out-of-bounds access.
*   **Precision:** Depth values are stored as 32-bit floats, which may introduce precision errors for very large or very small depth values [S1].

## Subtopics

### [Fractal Compression](5719-fractal-compression.md)
Fractal compression is a lossy volume compression method that exploits self-similarity within the dataset by mapping each "range block" to a larger, dissimilar "domain block" from the same volume, storing only the affine transformation required to convert the domain block into the range block. Unlike vector quantization, which stores a reference to a codeword, fractal compression stores the geometric relationship between two different-sized blocks, allowing for high compression ratios at the cost of extreme encoding complexity and iterative reconstruction during decoding.

### [Parallel Edgebreaker](7789-parallel-edgebreaker.md)
Parallel Edgebreaker is a parallel compression algorithm for isosurfaces where each part of the mesh is encoded independently from the others, based on the Edgebreaker algorithm. It solves the bottleneck of sequential mesh compression in distributed environments by trading a slight increase in file size for significant reductions in transmission and processing time. Unlike sequential methods that require global traversal, this method allows independent encoding of mesh partitions, making it suitable for high-latency network transfers where processing speed outweighs storage efficiency.

## Operating characteristics
*   **Time Complexity:** Encoding complexity is dominated by voxelization and link generation, which is linear with respect to the number of particles and surface voxels. Decoding is efficient as it involves simple lookups and on-the-fly connectivity computation [S1].
*   **Memory Complexity:** Memory usage is proportional to the number of surface voxels per pixel column, significantly less than storing the full 3D volume [S1].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** High compression ratios for sparse particle data; efficient rendering for frontmost and translucent surfaces; on-the-fly connectivity computation reduces storage overhead [S1].
*   **Limitations:** Limited to particle-based data; smoothing iterations depend on camera distance, which may require dynamic adjustment [S1].
*   **When to use:** Suitable for rendering large-scale particle systems where memory bandwidth is a bottleneck and the data is sparse.
*   **When not to use:** Not suitable for dense volumetric data or non-particle-based volumes.

## Background
The method was introduced in "Memory-Efficient On-The-Fly Voxelization of Particle Data" [S1]. It builds on the idea of storing surface geometry rather than volumetric data to reduce memory footprint. The use of octahedron normal encoding and neighbor links allows for efficient rendering and connectivity computation without storing explicit mesh structures [S1].

## Sources
[S1] Memory-Efficient On-The-Fly Voxelization of Particle Data, https://doi.org/10.2312/pgv.20151150, fetched 2026-10-26
