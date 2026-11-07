# Out-of-Core Point-Based Rendering

Also called: Point-based Volume Rendering

## Overview
Out-of-core point-based rendering is a method for the interactive visualization of very large volumetric datasets that bypasses the memory limitations of traditional in-core techniques by storing voxel data on disk [S1, unofficial]. It solves the problem of real-time interaction with high-resolution data by assuming voxel density is sufficient to represent isovalue structures as discrete points, eliminating the need for connectivity or topology information [S1, unofficial]. Unlike mesh-based methods which fail to provide interactive frame rates at full resolution for large datasets, this approach renders contours and volume approximations directly from sorted point clouds [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A gridded volumetric dataset (scalar field or RGB color volume) of arbitrary size, limited only by disk space and OS file size constraints.
*   **Output:** A rendered image of point primitives representing specific isovalue sets, updated interactively.

**Data Structures:**
1.  **Voxel File:** A single binary file containing all voxel position data.
2.  **Look-up Table (LUT):** A structure stored alongside or within the voxel file. For each unique color/value index $i$, it stores:
    *   `offset`: The byte offset in the voxel file where the voxelset $V_i$ begins.
    *   `length`: The number of voxels (or bytes) in the voxelset $V_i$.

**Numbered Steps:**

1.  **Pre-processing (Sorting and Mapping):**
    Iterate through the input gridded dataset. For each voxel at grid coordinates $(x, y, z)$ with function value $f$, map it to a color space representation. For an RGB color space, the mapping is defined as:
    $$ f_v : (x, y, z) \rightarrow (r, g, b) $$
    where $(r, g, b)$ are the byte-encoded function values [S1, unofficial]. Group all voxels by their resulting $(r, g, b)$ tuple.

2.  **Indexing (Morton Order):**
    Convert the 3D color tuple $(r, g, b)$ into a single linear index $I$ to determine storage order in the voxel file.
    *   **For RGB data:** Use a Morton order scheme (Z-order curve) to interleave the bits of the three color components. This ensures that colors close in RGB space are stored close to each other on disk [S1, unofficial].
    *   **For Greyscale data:** If the dataset is 8-bit greyscale, no Morton indexing is required. The greyscale value itself serves as the linear index [S1, unofficial].
    *   **Constraint:** The index for the background color (values to be excluded from rendering) must be $0$ [S1, unofficial].

3.  **Storage:**
    Write the voxel position data $(x, y, z)$ for each voxelset $V_I$ into the voxel file in the order of their linear indices. Simultaneously, populate the Look-up Table with the `offset` (current file position) and `length` (count of voxels) for each index $I$ [S1, unofficial].

4.  **Data Retrieval (Interactive Phase):**
    When a user selects a target color/value $(r, g, b)$:
    a. Calculate the linear index $I$ using the same mapping/indexing logic as pre-processing.
    b. Consult the Look-up Table to find the `offset` and `length` for index $I$.
    c. Perform a single seek operation to the `offset` in the voxel file.
    d. Read the `length` of position data into main memory [S1, unofficial].

5.  **Rendering:**
    Render the loaded points as small volumetric primitives (points) on the graphics hardware. No connectivity or topology information is used; rendering relies solely on the spatial distribution of the points [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct

def morton_encode(r, g, b):
    """
    Encodes three 8-bit integers into a single Morton index.
    Assumes 8-bit components. Interleaves bits: r0g0b0r1g1b1...
    """
    # Interleave bits for 8-bit components
    # This is a simplified representation of the Morton Z-order curve
    # In practice, this requires bit-shifting operations to interleave
    # bits from r, g, and b.
    # Example logic for 2-bit components (generalized to 8):
    # m = 0
    # for i in range(8):
    #     m |= ((r >> i) & 1) << (2*i)
    #     m |= ((g >> i) & 1) << (2*i + 1)
    #     m |= ((b >> i) & 1) << (2*i + 2)
    # Note: The exact bit-interleaving implementation depends on the 
    # specific Morton definition used, but the goal is a single linear index.
    # For the purpose of this article, we assume a standard Morton function exists.
    pass

def preprocess_dataset(input_grid, output_file_path, lut_path):
    """
    Pre-processing step: Sorts voxels by color/value and stores them.
    """
    # Initialize LUT
    # Assuming 256^3 possible RGB colors, but we only store entries for present colors
    # Or use a fixed-size array if memory allows for the LUT.
    # The paper implies a direct mapping where index 0 is background.
    
    # We need to know the max index to allocate LUT. 
    # For 8-bit RGB, max index is 2^24 - 1. 
    # However, Morton order maps 3D grid to 1D. 
    # The paper states: "index for the background color ... equals zero."
    
    luts = {} # Map index -> (offset, length)
    
    # Temporary storage for voxel data per index
    # Since we don't know the final file size, we might write to a temp buffer 
    # or calculate sizes first. The paper describes a "two-pass algorithm" 
    # implicitly by mentioning sorting.
    
    # Pass 1: Count voxels per color to determine file layout
    color_counts = {}
    
    for x, y, z in input_grid.coordinates():
        r, g, b = input_grid.get_value(x, y, z)
        
        # Determine background
        if is_background(r, g, b):
            continue
            
        idx = morton_encode(r, g, b)
        color_counts[idx] = color_counts.get(idx, 0) + 1

    # Pass 2: Write data
    current_offset = 0
    file = open(output_file_path, 'wb')
    
    # Sort indices to ensure deterministic order, though Morton order 
    # defines the logical order.
    sorted_indices = sorted(color_counts.keys())
    
    for idx in sorted_indices:
        count = color_counts[idx]
        luts[idx] = (current_offset, count)
        
        # Write voxel positions for this color
        for x, y, z in input_grid.get_voxels_with_color(idx):
            # Write position data. Format not strictly specified, 
            # assume 3 floats or 3 ints. [G] Assuming 32-bit floats for precision.
            file.write(struct.pack('fff', x, y, z))
            
        current_offset += count * 12 # 3 * 4 bytes
        
    file.close()
    
    # Save LUT
    save_lut(luts, lut_path)

def render_interactive(target_color, voxel_file_path, lut_path):
    """
    Interactive retrieval and rendering.
    """
    r, g, b = target_color
    
    # Check if background
    if is_background(r, g, b):
        return [] # No points to render
        
    idx = morton_encode(r, g, b)
    
    luts = load_lut(lut_path)
    
    if idx not in luts:
        return [] # Color not present in dataset
        
    offset, length = luts[idx]
    
    file = open(voxel_file_path, 'rb')
    file.seek(offset)
    
    # Read 'length' voxels
    # [G] Assuming 32-bit floats. If integers, adjust struct format.
    data = file.read(length * 12)
    file.close()
    
    # Parse into points
    points = []
    for i in range(length):
        x, y, z = struct.unpack('fff', data[i*12:(i+1)*12])
        points.append((x, y, z))
        
    return points
```

**Parameters and Typical Values:**
*   **Bit Depth:** 8-bit per channel for RGB (24-bit total) or 8-bit for greyscale [S1, unofficial].
*   **Position Storage:** Limited by the number of bits used to store position data. The paper notes this as a limiting factor for volumetric dimension [S1, unofficial].
*   **Background Index:** Must be $0$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Background Color:** The index for the background color must be zero. If the Morton encoding of the background color is not zero, it must be remapped or handled specifically to ensure it is not rendered [S1, unofficial].
*   **Greyscale Data:** For 8-bit greyscale, Morton indexing is skipped; values are used directly as indices. This simplifies the LUT but reduces the spatial locality optimization provided by Morton order [S1, unofficial].
*   **Memory Limits:** The method is limited by the number of points that can be rendered to achieve a sufficient frame rate, not by the total dataset size in memory [S1, unofficial].

## Subtopics

### [Hierarchical Sampled Point Volume Visualization](9322-hierarchical-sampled-point-volume-visualization.md)
A volume visualization method that replaces voxels with hierarchically sampled points to enable real-time previewing of 3D scalar fields. It solves the problem of time-consuming trial-and-error tuning of viewing parameters by allowing constant frame rate rendering through level-of-detail control.

## Operating characteristics
**Time and Memory Complexity:**
*   **Pre-processing:** $O(N \log N)$ or $O(N)$ depending on the sorting algorithm, where $N$ is the number of voxels. The Morton encoding is $O(1)$ per voxel.
*   **Retrieval:** $O(1)$ seek time plus $O(K)$ read time, where $K$ is the number of voxels in the selected isovalue set. This is significantly faster than scanning the entire dataset [S1, unofficial].
*   **Memory:** The method is out-of-core. Main memory usage is proportional to the size of the selected voxelset $K$ and the Look-up Table, not the total dataset size $N$ [S1, unofficial].

**Measured Performance:**
*   The method allows interactive frame rates for datasets that are "much too large to fit into graphic or main memory" [S1, unofficial].
*   Performance is limited by:
    1.  The number of points that can be rendered to achieve a sufficient frame rate (GPU bottleneck).
    2.  The number of bits used to store position data (disk I/O and memory bandwidth).
    3.  The maximum file-size supported by the operating system [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:**
    *   Enables real-time interaction with very large datasets (e.g., visible female dataset) [S1, unofficial].
    *   Does not require connectivity or topology information, simplifying the pipeline [S1, unofficial].
    *   Renders contours at the highest possible resolution (full voxel resolution) [S1, unofficial].
*   **Limitations:**
    *   Relies on the assumption that voxel density is high enough to approximate contours/volumes with points. Sparse data may result in poor visual quality [S1, unofficial].
    *   Limited by OS file size constraints and disk seek times for very large LUTs or fragmented files.
    *   No topological information is preserved, so features like holes or connectivity cannot be analyzed directly from the point cloud [S1, unofficial].

## Background
**Origin:**
The method was introduced by Nuber, Bruckschen, Hamann, and Joy from the Center for Image Processing and Integrated Computing (CIPIC) at the University of California, Davis [S1, unofficial].

**History and Development:**
The approach was developed to address the limitations of well-established 3D volumetric data visualization techniques, such as isosurface extraction and volume rendering, which usually cannot support real-time interaction with very large datasets [S1, unofficial]. By using out-of-core techniques, the limiting factor for interactive rendering shifts from available memory to hard-drive size [S1, unofficial].

**Usage:**
It is used for the interactive visualization of very large volumetric datasets, such as medical imaging data (e.g., bone-marrow color selection, visible female dataset) [S1, unofficial].

## Sources
[S1] Interactive Visualization of Very Large Datasets Using an Out-of-Core Point-based Approach, https://core.ac.uk/download/484268453.pdf, fetched 2026-10-27 (unofficial)
