# Out-of-Core Point-Based Rendering

Also called: Point-based Volume Rendering

## Overview
Out-of-core point-based rendering is a method for the interactive visualization of very large volumetric datasets that exceed available main memory [S1, unofficial]. It solves the problem of real-time interaction with high-resolution data by treating voxels as discrete points rather than connected mesh elements, thereby avoiding the computational cost of topology extraction [S1, unofficial]. Unlike mesh-based methods which fail to provide interactive frame rates at full resolution for massive datasets, this approach allows rendering isovalue structures (contours) at the highest possible resolution by sorting voxels by function value and storing them in a single file [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A gridded volumetric dataset (scalar field) defined on a Cartesian grid.
*   **Output:** A single file containing sorted voxel data and a look-up table, enabling real-time rendering of specific isovalue sets.

**Data Structures:**
1.  **Voxel File:** A single contiguous file storing all voxel sets $\{V\}$. Each voxel set $V_{(r,g,b)}$ contains the 3D coordinates $(x,y,z)$ of all voxels sharing a specific function value (mapped to color).
2.  **Look-up Table (LUT):** A structure stored alongside or within the file header. For each unique function value (indexed by color), it stores:
    *   `offset`: The byte position in the file where the voxel set begins.
    *   `length`: The number of voxels (or byte count) in that set.
3.  **Indexing Scheme:** A mapping from RGB color values to linear indices in the LUT. For RGB data, a Morton order scheme is used. For 8-bit greyscale data, the value itself serves as the index.

**Steps:**

1.  **Pre-processing (Data Generation):**
    *   Iterate through the input gridded dataset.
    *   Map each voxel's grid coordinates $(x, y, z)$ to a function value $fv$. For RGB-colored datasets, this is a mapping $fv : (x, y, z) \rightarrow (r, g, b)$ [S1, unofficial].
    *   Group voxels by their function value. A voxel set is defined as $V_{(r,g,b)} = \{v_i \mid fv(v_i) = (r, g, b)\}$ [S1, unofficial].
    *   Sort the voxel sets. The sorting order determines the linear index $k$ for each color.
    *   Write the voxel coordinates for each set $V_k$ to the output file.
    *   Record the `offset` (start position) and `length` (count of voxels) for each set $k$ in the Look-up Table.

2.  **Indexing (Morton Order for RGB):**
    *   To allow fast retrieval of a specific color $(r, g, b)$, convert the 3-byte RGB value into a single linear index $k$ using a Morton order scheme [S1, unofficial].
    *   Morton order interleaves the bits of the three color components to create a linear index that preserves spatial locality in color space.
    *   **Constraint:** The index for the background color (values not encoded or rendered) must be zero [S1, unofficial].
    *   *Note:* For 8-bit greyscale data, skip Morton indexing; use the greyscale value directly as the index [S1, unofficial].

3.  **Data Retrieval (Interactive Phase):**
    *   User selects a target function value (color) $C_{target}$.
    *   Compute the linear index $k$ for $C_{target}$ using the indexing scheme (Morton for RGB, direct for greyscale).
    *   Access the Look-up Table to retrieve `offset` and `length` for index $k$.
    *   Perform a single seek/read operation to load the voxel coordinates for $V_k$ from the file into memory [S1, unofficial].

4.  **Rendering:**
    *   Render the loaded points as small volumetric primitives (points) [S1, unofficial].
    *   No connectivity or topology information is used; rendering relies solely on the spatial distribution of the points [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_volumetric_data(grid, output_file_path):
    """
    grid: 3D array of function values (e.g., RGB tuples or integers).
    output_file_path: Path to the output file.
    """
    # Step 1: Group voxels by function value
    # Dictionary mapping index k to list of (x, y, z) coordinates
    voxel_sets = {}
    
    # Determine indexing scheme based on data type
    is_greyscale = isinstance(grid[0,0,0], int) # Simplified check
    
    for x in range(grid.shape[0]):
        for y in range(grid.shape[1]):
            for z in range(grid.shape[2]):
                value = grid[x, y, z]
                
                # Determine index k
                if is_greyscale:
                    k = value
                else:
                    # Morton order for RGB (r, g, b)
                    r, g, b = value
                    k = morton_order_3d(r, g, b)
                
                if k not in voxel_sets:
                    voxel_sets[k] = []
                voxel_sets[k].append((x, y, z))
    
    # Step 2: Sort indices to ensure consistent ordering
    sorted_indices = sorted(voxel_sets.keys())
    
    # Step 3: Write to file and build Look-up Table
    lut = {}
    current_offset = 0
    
    with open(output_file_path, 'wb') as f:
        for k in sorted_indices:
            points = voxel_sets[k]
            num_points = len(points)
            
            # Write voxel coordinates (e.g., as 3 floats or ints)
            # Implementation detail: packing format depends on precision requirements
            for (x, y, z) in points:
                write_voxel(f, x, y, z)
            
            # Record LUT entry
            lut[k] = {
                'offset': current_offset,
                'length': num_points
            }
            
            # Update offset for next set
            current_offset += num_points * bytes_per_voxel
            
    # Save LUT (e.g., to header or separate file)
    save_lut(lut, output_file_path)

def retrieve_and_render(target_color, output_file_path):
    """
    target_color: RGB tuple or integer value.
    Returns list of (x, y, z) coordinates for the target isovalue.
    """
    # Determine index k
    if is_greyscale_data:
        k = target_color
    else:
        r, g, b = target_color
        k = morton_order_3d(r, g, b)
    
    # Load LUT
    lut = load_lut(output_file_path)
    
    if k not in lut:
        return []
    
    entry = lut[k]
    offset = entry['offset']
    length = entry['length']
    
    # Single seek/read operation
    with open(output_file_path, 'rb') as f:
        f.seek(offset)
        raw_data = f.read(length * bytes_per_voxel)
        
    # Parse raw data into coordinates
    points = parse_voxels(raw_data, length)
    return points

def morton_order_3d(r, g, b):
    """
    Converts 3D RGB coordinates to a 1D Morton index.
    Interleaves bits of r, g, b.
    """
    # Standard bit-interleaving algorithm
    index = 0
    for i in range(8): # Assuming 8-bit components
        index |= ((r >> i) & 1) << (3 * i)
        index |= ((g >> i) & 1) << (3 * i + 1)
        index |= ((b >> i) & 1) << (3 * i + 2)
    return index
```

**Parameters and Constraints:**
*   **Background Color Index:** Must be 0 [S1, unofficial].
*   **Limiting Factors:**
    1.  Number of points renderable at a sufficient frame rate [S1, unofficial].
    2.  Number of bits used to store position data (precision vs. file size) [S1, unofficial].
    3.  Maximum file size supported by the operating system [S1, unofficial].

## Subtopics

### [Hierarchical Sampled Point Volume Visualization](9322-hierarchical-sampled-point-volume-visualization.md)
Replaces voxels with hierarchically sampled points for real-time previewing and LOD control, allowing constant frame rates by estimating point counts in advance.

## Operating characteristics
*   **Time Complexity:** Pre-processing is $O(N)$ where $N$ is the number of voxels, dominated by sorting/grouping. Interactive retrieval is $O(1)$ for LUT access plus $O(M)$ for reading $M$ points, where $M$ is the size of the selected isovalue set.
*   **Memory Complexity:** Out-of-core. Main memory usage is limited to the size of the currently loaded voxel set, not the entire dataset. Disk usage is proportional to $N \times \text{bytes\_per\_voxel}$.
*   **Performance:** Enables real-time interaction and full-resolution rendering of isovalue structures for datasets too large for main memory [S1, unofficial].
*   **Strengths:** No connectivity/topology computation required; handles very large datasets; real-time isovalue switching.
*   **Limitations:** Result is a set of points ("isoclouds") rather than a continuous surface; quality depends on voxel density; limited by OS file size limits and rendering throughput [S1, unofficial].

## Background
The method was introduced by Nuber, Bruckschen, Hamann, and Joy from the Center for Image Processing and Integrated Computing (CIPIC) at the University of California, Davis [S1, unofficial]. It addresses the limitations of established techniques like isosurface extraction and volume rendering, which typically cannot support real-time interaction with very large discretized volumetric scalar fields [S1, unofficial]. The approach leverages the high density of voxels with the same function value to approximate contours using points.

## Sources
[S1] Interactive Visualization of Very Large Datasets Using an Out-of-Core Point-based Approach, https://core.ac.uk/download/484268453.pdf, fetched 2026-10-27 (unofficial)
