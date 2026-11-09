# Evolutionary Volumetric Data Structure (EVDS) / L-Block Compression (LBC)

Also called: L-block Covering, L-blocks, Polymerization Strategy

## Overview
The Evolutionary Volumetric Data Structure (EVDS), stored as an L-block Covering (LBC), is a hierarchical sparse data structure designed for the real-time incremental compression and segmentation of filamentary volumetric data, such as neuronal tissue [S1, unofficial]. It solves the problem of managing massive, sparse datasets (e.g., 30 TB mouse brain scans) by packaging significant data into connected L-blocks (LBs) while ignoring "white space," allowing for on-the-fly generation without storing the entire volume in memory [S1, unofficial]. Unlike standard AABB-trees, the LBC allows for easy incremental box building and maintains connectivity between nodes without requiring traversal through a parent node [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Input:** A stream of consecutive 2D image slices (serial sections) scanned in the XY plane at depths $Z$ and $Z+1$. Each voxel has a grayscale value.
*   **Output:** An L-block Covering (LBC) representing the Enhanced Volume Data Set (EVDS), consisting of a set of connected L-blocks and their hierarchical relationships.

### Data Structures

**L-Block (LB)**
The basic unit of storage. An LB is an axis-aligned box containing voxel data.
*   **Dimensions:** Typically $(2, 2, 2)$, but can be $(m, n, 1)$ or other dimensions.
*   **Content:** Stores valid voxels and associated "white space" (empty voxels within the bounding box).
*   **Storage Cost:** In the reference implementation, 6 bytes per LB are used for data and whitespace [S1, unofficial].

**L-Block Covering (LBC)**
A hierarchical structure storing connected LBs.
*   **Structure:** Similar to an AABB-tree but optimized for incremental updates and local connectivity.
*   **Header Cost:** One bit for marking leaf nodes, and a 2-byte pointer for each node [S1, unofficial].
*   **Connectivity:** LBs sharing an active edge are assumed to be connected. The structure maintains this connectivity explicitly [S1, unofficial].

### Algorithm Steps

1.  **Thresholding and Partitioning:**
    *   Apply a conservative labeling function to segment critical data. Vertices are considered "significant" if their grayscale values pass a threshold test (e.g., above a minimum and below a maximum level) [S1, unofficial].
    *   Edges are labeled "active" if and only if both adjacent vertices are significant [S1, unofficial].
    *   Voxels that do not pass the threshold are treated as "white space" [S1, unofficial].
    *   Partition the EVDS into $2 \times 2 \times 2$ cells. If any voxel in a cell is valid (significant), store that cell as a $(2, 2, 2)$ LB [S1, unofficial].

2.  **Incremental Generation (Polymerization):**
    *   Process consecutive serial sections scanned in the XY plane at depths $Z$ and $Z+1$ [S1, unofficial].
    *   Retain only the immediate predecessor ($Z$) in memory. Use the current image data ($Z+1$) to enhance the $Z$-plane data [S1, unofficial].
    *   Package regions of interest in the $Z$-plane into $(m, n, 1)$ L-blocks and add them to the evolving EVDS [S1, unofficial].
    *   Focus on connected components in the extended volume data set. Isolated vertices outside connected components may be retained as individual LBs. Remaining vertices are treated as whitespace and ignored [S1, unofficial].

3.  **Noise Removal:**
    *   **Noise Removal I (Smear):** Remove large smear noise that exists on single layers [S1, unofficial].
    *   **Noise Removal II (Isolated):** Eliminate small disconnected LBs. Since LBC is a local packaging structure, noise is wrapped in small isolated packages and removed [S1, unofficial].

4.  **Merging:**
    *   Combine LBs to minimize header overhead.
    *   Extend LBs in positive X, Y, and Z axis directions if the space saved by eliminating L-block overhead is greater than the space lost by storing empty data (whitespace) [S1, unofficial].
    *   Note: Combining larger LBs with smaller ones may be problematic due to the storage of whitespace [S1, unofficial].

5.  **Neuron Reconstruction (Optional Post-Processing):**
    *   Dilate LBs within each cluster so they overlap "nearby" LBs [S1, unofficial].
    *   Join overlapping LBCs to fill gaps caused by noise [S1, unofficial].
    *   Form an expanded connectivity graph linking overlapping dilated LBs [S1, unofficial].
    *   Reduce complexity by identifying short cycles that do not change global topology [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_serial_sections(current_slice, previous_slice, evds):
    """
    Incrementally updates the EVDS with the current slice.
    current_slice: 2D array of grayscale values at depth Z+1
    previous_slice: 2D array of grayscale values at depth Z (retained in memory)
    evds: The evolving L-block Covering structure
    """
    # Step 1: Thresholding
    # Define significance thresholds
    MIN_VAL = 100
    MAX_VAL = 255
    
    # Identify significant vertices in current slice
    significant_vertices = []
    for y in range(height):
        for x in range(width):
            val = current_slice[y, x]
            if MIN_VAL <= val <= MAX_VAL:
                significant_vertices.append((x, y))
                
    # Step 2: Partitioning into L-Blocks
    # Partition into 2x2x2 cells. Since we are processing slice Z+1,
    # we package regions of interest into (m, n, 1) LBs.
    # For simplicity, assume 2x2x2 blocks are formed from 2x2x2 voxel grids.
    # In the incremental step, we primarily form (m, n, 1) blocks for the current slice.
    
    new_lbs = []
    block_size_x = 2
    block_size_y = 2
    block_size_z = 1 # Incremental step processes one slice at a time
    
    for y in range(0, height, block_size_y):
        for x in range(0, width, block_size_x):
            # Check if any voxel in this 2x2x1 block is significant
            is_valid = False
            for dy in range(block_size_y):
                for dx in range(block_size_x):
                    if (x + dx, y + dy) in significant_vertices:
                        is_valid = True
                        break
                if is_valid: break
            
            if is_valid:
                # Create a new LB
                lb = create_lb(x, y, z_depth, block_size_x, block_size_y, block_size_z)
                new_lbs.append(lb)
                
    # Step 3: Connectivity and Merging
    # Add new LBs to EVDS
    for lb in new_lbs:
        evds.add_lb(lb)
        
    # Merge LBs to reduce overhead
    # Extend LBs in positive X, Y, Z directions if space saving > space lost
    evds.merge_lbs()
    
    # Step 4: Noise Removal
    # Remove isolated small LBs
    evds.remove_isolated_noise()
    
    return evds

def create_lb(x, y, z, w, h, d):
    """
    Creates an L-Block.
    """
    # Store valid voxels and whitespace
    # Cost is 6 bytes per LB in reference implementation
    return {
        'x': x, 'y': y, 'z': z,
        'w': w, 'h': h, 'd': d,
        'data': None # Placeholder for voxel data and whitespace
    }
```

### Parameters
*   **LB Size:** Basic unit is $(2, 2, 2)$, but $(m, n, 1)$ is used for incremental processing [S1, unofficial].
*   **Voxel Dimensions:** $0.37 \, \mu m \times 0.37 \, \mu m \times 0.5 \, \mu m$ [S1, unofficial].
*   **Thresholds:** Conservative labeling function initially; specific values depend on stain and data density [S1, unofficial].

### Edge Cases and Pitfalls
*   **Merging Complexity:** Combining larger LBs with smaller ones is problematic because expanding an LB may require storing "white" space along with relevant data [S1, unofficial].
*   **Connectivity Ambiguity:** Two neighboring LBs might not be physically connected; the EVDS must be checked to determine if a connection actually exists [S1, unofficial].
*   **False Positives in Reconstruction:** Dilating LBs and joining overlapping LBCs may join LBCs that should be kept separate [S1, unofficial].

## Subtopics

### Incremental Generation
*   **What it is:** A processing mode where the EVDS is built on-the-fly as serial sections are scanned, retaining only the immediate predecessor in memory.
*   **Difference:** Unlike batch processing methods, it allows real-time compression and segmentation with bounded memory usage.
*   **When to choose:** When processing continuous data streams (e.g., KESM microscopy) where total data size exceeds available RAM.
*   **Link:** [Incremental Generation](5378-incremental-generation.md)

### Noise Removal
*   **What it is:** Strategies to eliminate smear noise (single-layer artifacts) and isolated disconnected LBs.
*   **Difference:** LBC's local packaging structure naturally isolates noise, making it easier to remove compared to global volumetric filters.
*   **When to choose:** When the input data contains significant artifacts from the scanning process.
*   **Link:** [Noise Removal](5379-noise-removal.md)

### Neuron Reconstruction
*   **What it is:** Post-processing step involving dilation, joining overlapping LBCs, and graph simplification to model neuronal threads.
*   **Difference:** Uses the LBC's connectivity graph to identify major threads and reduce complexity by removing short cycles.
*   **When to choose:** When the goal is biological modeling rather than just visualization.
*   **Link:** [Neuron Reconstruction](5380-neuron-reconstruction.md)

## Operating characteristics

*   **Time Complexity:** Real-time incremental generation is possible. Merging and noise removal are local operations, bounded by the size of the section [S1, unofficial].
*   **Memory Complexity:** Bounded by the size of the current and previous sections. Does not require storing the entire volume [S1, unofficial].
*   **Measured Performance:**
    *   **Neuron Data (250x230x90):** Uncompressed 5 MB becomes 1.1 MB (65,611 LBs), yielding a compression factor of ~4 (20% of original size). Merging reduces LB count to 28,938 (<0.4 MB) and further to 10,684 (<0.18 MB) [S1, unofficial].
    *   **Cell Body Data (500x500x100):** Uncompressed 25 MB becomes 0.8 MB (50,218 LBs), yielding a compression factor of ~31 (3.2% of original size). Merging reduces LB count to 10,684 (<0.18 MB) [S1, unofficial].
*   **Scaling:** Suitable for large datasets (e.g., 30 TB mouse brain scans) due to incremental processing [S1, unofficial].
*   **Strengths:** Real-time processing, low memory footprint, efficient compression for filamentary data, easy incremental updates.
*   **Limitations:** Poor performance for short, blobby structures (cell bodies) compared to octrees; merging complexity; potential for false positives in reconstruction.
*   **When to use:** Filamentary volumetric data (neurons) from serial sectioning microscopy.
*   **When not to use:** Dense volumetric data or data with short, blobby structures where octrees may be more efficient.

## Background

*   **Origin:** Developed by B. McCormick et al. for processing data from the Knife Edge Scanning Microscope (KESM) [S1, unofficial].
*   **History:** Presented in 2004 at the Eurographics Symposium on Rendering [S1, unofficial].
*   **Usage:** Used in the KESM pipeline for processing stained mouse brain tissue data. Also used for isosurface generation via isosplats and raytracing visualization [S1, unofficial].

## Sources

[S1] Compression, Segmentation, and Modeling of Filamentary Volumetric Data, https://doi.org/10.2312/sm.20041411, fetched 2026-10-26 (unofficial)
