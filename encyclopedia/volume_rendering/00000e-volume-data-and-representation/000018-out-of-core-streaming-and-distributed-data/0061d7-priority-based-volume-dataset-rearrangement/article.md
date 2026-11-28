# Priority-Based Volume Dataset Rearrangement

Also called: Segmentation Metadata Generation, Volume Dataset Segmentation and Metadata Generation Algorithm

## Overview
Priority-Based Volume Dataset Rearrangement is an out-of-core data management method that reorganizes a volume dataset from storage to enable progressive loading based on semantic importance. It solves the latency problem in large-volume visualization by ensuring that high-priority data (e.g., surfaces or foreground objects) is loaded first, while lower-priority data (e.g., background or interior) loads in the background. It sets itself apart from sibling methods like [Sectioning](1636-sectioning.md) or [Chunk-based spatial partitioning](3862-chunk-based-spatial-partitioning.md) by decoupling the physical storage order from the logical priority order using explicit segmentation metadata, rather than relying solely on spatial locality or fixed grid structures.

## How it works
The method transforms a volume dataset $R$ into a rearranged linear stream and a corresponding metadata grid $R''$. The process ensures that the original spatial position of any voxel can be reconstructed from the rearranged data and metadata, even if only a subset of segments is loaded.

### Inputs and Outputs
**Inputs:**
1.  Volume dataset $R[X, Y, Z]$, a regular 3D grid of scalar values.
2.  Data block dimensions $B_x, B_y, B_z$ defining the size of each partition.
3.  Number of priority levels $\rho_{max}$. Typical value: 4 [S1].
4.  A priority assignment function $f$ (details depend on specific segmentation needs, e.g., thresholding) [S1].

**Outputs:**
1.  Segmentation metadata grid $R''[X'', Y'', Z'']$.
2.  Rearranged linearized volume dataset $R'$ (the physical storage order).

### Data Structures

**1. Segmentation Metadata Grid ($R''$)**
*   **Layout:** A 3D grid with dimensions $X'', Y'', Z''$.
*   **Dimensions:** $X'' = X'$, $Y'' = Y'$, $Z'' = Z'$, where $X', Y', Z'$ are the number of data blocks along each axis [S1].
*   **Content:** Each entry is a scalar value representing the priority level of the corresponding data block.
*   **Encoding:** Unsigned integer.
*   **Bit Width:** $\lceil \log_2 \rho_{max} \rceil$ bits per block [S1].
*   **Total Size:** $R''_{size} = N \times \lceil \log_2 \rho_{max} \rceil$ bits, where $N$ is the total number of data blocks [S1].

**2. Rearranged Dataset ($R'$)**
*   **Layout:** A 1D linear stream of data blocks.
*   **Ordering:** Blocks are ordered by descending priority. Segment $S_0$ (highest priority) comes first, followed by $S_1$, etc.
*   **Internal Order:** Within each segment $S_p$, blocks maintain the order established by the linearization step (Step 3).
*   **Voxel Linearization:** Voxels within each data block are also linearized according to the selected linearization scheme [S1].

### Algorithm Steps

**Step 1: Partitioning**
Divide the volume dataset $R[X, Y, Z]$ into data blocks. If the volume dimensions are $X, Y, Z$ and block dimensions are $B_x, B_y, B_z$, the number of blocks along each axis is $X' = \lceil X/B_x \rceil$, $Y' = \lceil Y/B_y \rceil$, $Z' = \lceil Z/B_z \rceil$ [G]. The total number of blocks is $N = X' \times Y' \times Z'$ [S1].

**Step 2: Priority Assignment and Metadata Generation**
Assign a priority level to each data block using function $f$.
$$ R''[X'', Y'', Z''] = f: (R'[X', Y', Z']) \rightarrow P $$
Where $P = \{0, 1, \dots, \rho_{max}-1\}$ is the set of priority levels, with 0 being the highest priority [S1].
Group all data blocks with the same priority $p$ into a segment $S_p$. The set of segments is:
$$ \Sigma = \{S_0, S_1, \dots, S_{\rho_{max}-1}\} $$
Every data block belongs to exactly one segment:
$$ R'[X', Y', Z'] = \bigcup_{r=0}^{\rho_{max}-1} S_r $$
Encode the priority of each block in $R''$ using $\lceil \log_2 \rho_{max} \rceil$ bits [S1].

**Step 3: Linearization of Original Dataset**
Linearize the original volume dataset $R'$ so that all data blocks are ordered in a one-dimensional stream according to a selected linearization (e.g., row-major order) [S1]. Let this linearized sequence be $L_{orig}$.

**Step 4: Rearrangement**
Rearrange the data blocks in $L_{orig}$ to form the output dataset $R'$.
1.  Sort the segments in descending priority order: $S_0, S_1, \dots, S_{\rho_{max}-1}$.
2.  Concatenate the blocks from each segment.
3.  Within each segment $S_p$, preserve the relative order of blocks as they appeared in $L_{orig}$ [S1].

**Step 5: Voxel Linearization**
Linearize the voxels within each individual data block of the rearranged dataset $R'$ according to the selected linearization [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def priority_based_rearrangement(volume, block_dims, rho_max, priority_func):
    """
    volume: 3D array (X, Y, Z)
    block_dims: tuple (Bx, By, Bz)
    rho_max: int, number of priority levels
    priority_func: function(block_data) -> int, returns priority 0..rho_max-1
    """
    X, Y, Z = volume.shape
    Bx, By, Bz = block_dims
    
    # Step 1: Partitioning
    # Calculate number of blocks per axis
    X_blocks = math.ceil(X / Bx)
    Y_blocks = math.ceil(Y / By)
    Z_blocks = math.ceil(Z / Bz)
    
    # Create list of block indices and their original linear index
    # Linear index follows row-major order over blocks
    blocks = []
    for z in range(Z_blocks):
        for y in range(Y_blocks):
            for x in range(X_blocks):
                # Define block boundaries
                x_start = x * Bx
                y_start = y * By
                z_start = z * Bz
                x_end = min(x_start + Bx, X)
                y_end = min(y_start + By, Y)
                z_end = min(z_start + Bz, Z)
                
                # Extract block data
                block_data = volume[z_start:z_end, y_start:y_end, x_start:x_end]
                
                # Calculate original linear index
                linear_idx = z * (Y_blocks * X_blocks) + y * X_blocks + x
                
                blocks.append({
                    'linear_idx': linear_idx,
                    'data': block_data,
                    'coords': (x, y, z)
                })
    
    # Step 2: Priority Assignment
    # Assign priority to each block
    for block in blocks:
        block['priority'] = priority_func(block['data'])
        
    # Generate Metadata Grid R''
    # R'' is a 3D grid of priorities matching the block grid dimensions
    metadata_grid = [[None] * X_blocks for _ in range(Y_blocks)]
    for z in range(Z_blocks):
        for y in range(Y_blocks):
            for x in range(X_blocks):
                # Find block at this coordinate
                # In row-major block order:
                idx = z * (Y_blocks * X_blocks) + y * X_blocks + x
                block = blocks[idx]
                metadata_grid[y][x] = block['priority'] # Note: indexing order depends on linearization choice
                
    # Group blocks into segments
    segments = {p: [] for p in range(rho_max)}
    for block in blocks:
        p = block['priority']
        segments[p].append(block)
        
    # Step 3: Linearization of Original Dataset
    # The 'blocks' list is already linearized in row-major order (Step 3 logic)
    # We use the 'linear_idx' to maintain relative order within segments
    
    # Step 4: Rearrangement
    # Sort segments by priority (descending, i.e., 0 first)
    sorted_priorities = sorted(segments.keys())
    
    rearranged_blocks = []
    for p in sorted_priorities:
        # Get blocks in segment S_p
        seg_blocks = segments[p]
        # Sort by original linear index to maintain internal order
        seg_blocks.sort(key=lambda b: b['linear_idx'])
        rearranged_blocks.extend(seg_blocks)
        
    # Step 5: Voxel Linearization
    # Flatten each block's data into a 1D array
    final_dataset = []
    for block in rearranged_blocks:
        # Flatten block data (e.g., row-major)
        flat_data = block['data'].flatten()
        final_dataset.append(flat_data)
        
    return metadata_grid, final_dataset
```

### Parameters and Edge Cases
*   **$\rho_{max}$**: The number of priority levels. Typical value is 4 [S1].
*   **$N$**: Total number of data blocks.
*   **Metadata Omission**: The number of data blocks in each segment can be omitted from the metadata representation because it can be derived during reconstruction by counting occurrences of each priority level in $R''$ [S1].
*   **Reconstruction**: The original position of each voxel can be reconstructed from the rearranged dataset and metadata $R''$, even if only partial segments are loaded [S1].
*   **Pitfall**: The function $f$ must be consistent. If $f$ depends on global statistics (e.g., global histogram), pre-processing is required. If $f$ is local, it can be computed on-the-fly, but metadata generation still requires a pass [S1].

## Subtopics

### [Improved Reading Algorithm with Pre-processing](18684-improved-reading-algorithm-with-pre-processing.md)
Extends the base method by pre-processing voxels into foreground/background and surface/interior categories before assigning priorities. This enables immediate visualization of the surface (Segment $S_0$) while less important segments load in the background, significantly reducing time to first surface view.

## Operating characteristics
*   **Time Complexity**: The rearrangement requires a single pass to assign priorities ($O(N)$) and a sorting/grouping step ($O(N \log N)$ or $O(N)$ with bucket sort depending on $\rho_{max}$). Linearization is $O(N)$.
*   **Memory Complexity**: Requires storage for the metadata grid $R''$ ($N \times \lceil \log_2 \rho_{max} \rceil$ bits) and the rearranged dataset. The original dataset can be discarded after rearrangement if storage is limited [S1].
*   **Strengths**: Enables true progressive loading based on semantic importance rather than just spatial proximity. Reduces time to first view for critical data.
*   **Limitations**: Requires an initial preprocessing pass to generate metadata and rearrange data. The rearrangement is a one-time cost (or infrequent). If the priority assignment is inaccurate, the user experience may degrade.
*   **When to use**: Large volumes where semantic importance varies significantly and the user benefits from seeing high-priority features (e.g., surfaces) immediately.
*   **When not to use**: Small volumes that fit in memory, or when spatial locality is the primary optimization goal (use [Chunk-based spatial partitioning](3862-chunk-based-spatial-partitioning.md) instead).

## Background
The method was introduced by B. Madoš et al. in 2021 in the paper "Reading Volume Datasets from Storage – Using Segmentation Metadata, for an Enhanced User Experience" [S1]. It addresses the limitations of traditional out-of-core rendering where data is loaded in spatial order, potentially delaying the visualization of critical features. The algorithm is part of a broader effort to enhance user experience in volume visualization by leveraging segmentation metadata.

## Sources
[S1] Reading Volume Datasets from Storage – Using Segmentation Metadata, for an Enhanced User Experience, https://doi.org/10.12700/aph.18.5.2021.5.12, fetched 2026-10-30
