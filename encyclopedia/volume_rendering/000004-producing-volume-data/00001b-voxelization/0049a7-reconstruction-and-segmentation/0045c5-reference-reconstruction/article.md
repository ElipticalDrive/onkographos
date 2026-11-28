# Reference Reconstruction

Also called: Brick Merging, Component Reconstruction

## Overview
Reference Reconstruction assembles a complete volumetric dataset from exported component bricks (chunks), resolving spatial overlaps by taking the maximum value of the constituent components [S1, unofficial]. It differs from generative methods like [Voxel Coloring](4418-voxel-coloring.md) by strictly reconstructing an existing discrete grid structure rather than inferring geometry from projections, and from [Voronoi-like Segmentation Volume Generation](4218-voronoi-like-segmentation-volume-generation.md) by handling pre-existing discretized data rather than generating segmentation boundaries [S1, unofficial].

## How it works
**Inputs:** A list of component records, each containing a path to a raw data file (`.raw`), a spacing factor per axis, an extent (crop region), and an origin (placement offset).
**Outputs:** A 3D array `result` of shape $(N_z, N_y, N_x)$ and a boolean coverage array `coverage` of the same shape.

**Data Structures:**
*   `result`: `uint8` or `uint16` array, initialized to zeros.
*   `coverage`: `uint8` array, initialized to zeros, tracking how many bricks contribute to each voxel.
*   `block`: Temporary buffer for the current brick's data.

**Steps:**
1.  Initialize `result` and `coverage` arrays with shape `(N_z, N_y, N_x)` (ZYX order) filled with zeros [S1, unofficial].
2.  For each component record in the input list:
    a.  Read the raw data file into `block` [S1, unofficial].
    b.  Upsample `block` along each axis by repeating values `spacing` times, where `spacing` is derived from `comp['spacing_xyz'][::-1]` (ZYX order) [S1, unofficial].
    c.  Crop the upsampled `block` to the specified `extent_xyz[::-1]` (ZYX order) [S1, unofficial].
    d.  Map the cropped `block` to the `result` array at the position defined by `origin_xyz[::-1]` (ZYX order) using the operation:
        $$ \text{result}[target] = \max(\text{result}[target], \text{block}) $$
        This is implemented as `np.maximum(result[target], block, out=result[target])` [S1, unofficial].
    e.  Update the coverage map:
        $$ \text{coverage}[target] \leftarrow \text{coverage}[target] + (\text{block} > 0) $$
        where $(\text{block} > 0)$ is a boolean mask converted to integer [S1, unofficial].

**Pitfalls:**
*   **Lossy Merging:** Overlapping bricks are resolved by `np.maximum`. If overlapping regions contain different non-zero values, the lower value is discarded, potentially losing information [S1, unofficial].
*   **Precision:** Ensure the output array type (`uint8`/`uint16`) matches the source data to prevent overflow or truncation [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def reference_reconstruction(components, shape_zyx):
    """
    Reconstructs a volume from component bricks.
    
    Args:
        components: List of dicts with keys:
            'path': str, path to .raw file
            'spacing_xyz': tuple of int, (x, y, z) spacing factors
            'extent_xyz': tuple of int, (x, y, z) crop extent
            'origin_xyz': tuple of int, (x, y, z) placement origin
        shape_zyx: tuple of int, (N_z, N_y, N_x)
        
    Returns:
        result: np.ndarray, shape (N_z, N_y, N_x)
        coverage: np.ndarray, shape (N_z, N_y, N_x)
    """
    # Initialize result and coverage arrays
    result = np.zeros(shape_zyx, dtype=np.uint8)
    coverage = np.zeros(shape_zyx, dtype=np.uint8)
    
    for comp in components:
        # Load raw data
        # Note: In a real implementation, this would read the binary file.
        # Here we assume 'block_raw' is the loaded data of shape (H, W, D) or similar.
        # The source code implies reading into a block variable.
        block_raw = load_raw_file(comp['path']) # [G] Placeholder for actual file I/O
        
        # Upsample by repeating values
        # spacing_xyz is (x, y, z), but numpy repeat works on axes 0,1,2 (z,y,x)
        # So we reverse spacing for axis alignment
        spacing_zyx = comp['spacing_xyz'][::-1]
        
        # Repeat along axis 0 (z), axis 1 (y), axis 2 (x)
        block = block_raw
        for axis, factor in enumerate(spacing_zyx):
            if factor > 1:
                block = np.repeat(block, factor, axis=axis)
                
        # Crop to extent
        # extent_xyz is (x, y, z), reverse to (z, y, x) for slicing
        extent_zyx = comp['extent_xyz'][::-1]
        
        # Define slice objects for cropping
        # Assuming extent defines the size of the crop, not start/end.
        # The source says "Crop block to extent_xyz". 
        # If extent is size, we take the first 'extent' elements.
        # If extent is [start, end], we use that. 
        # Given "crop to extent", it usually implies size. 
        # Let's assume extent_zyx is the size of the region to take from the start.
        # If the source implies a specific region, more context is needed.
        # Standard interpretation: take block[:extent_zyx[0], :extent_zyx[1], :extent_zyx[2]]
        block_cropped = block[:extent_zyx[0], :extent_zyx[1], :extent_zyx[2]]
        
        # Map to result
        # origin_xyz is (x, y, z), reverse to (z, y, x)
        origin_zyx = comp['origin_xyz'][::-1]
        
        # Calculate target slice in result
        # The block_cropped is placed at origin_zyx
        z_start, y_start, x_start = origin_zyx
        z_end = z_start + block_cropped.shape[0]
        y_end = y_start + block_cropped.shape[1]
        x_end = x_start + block_cropped.shape[2]
        
        # Ensure we don't go out of bounds
        z_end = min(z_end, shape_zyx[0])
        y_end = min(y_end, shape_zyx[1])
        x_end = min(x_end, shape_zyx[2])
        
        # Adjust block_cropped if it exceeds bounds (though source implies valid inputs)
        # For robustness, we crop the block to fit
        actual_z = z_end - z_start
        actual_y = y_end - y_start
        actual_x = x_end - x_start
        
        target_block = block_cropped[:actual_z, :actual_y, :actual_x]
        
        # Update result with maximum
        np.maximum(result[z_start:z_end, y_start:y_end, x_start:x_end], 
                   target_block, 
                   out=result[z_start:z_end, y_start:y_end, x_start:x_end])
        
        # Update coverage
        # block > 0 creates a boolean array, convert to uint8
        mask = (target_block > 0).astype(np.uint8)
        coverage[z_start:z_end, y_start:y_end, x_start:x_end] += mask
        
    return result, coverage
```

## Subtopics

### Variable Resolution Reconstruction
Derives reduced-resolution basis matrices $U^{(k)} \in \mathbb{R}^{J_k \times R_k}$ by subsampling the original bases $U^{(k)}$ at a reduced rate $J_k < I_k$ in the corresponding index dimensions, then reconstructs a lower resolution volume $A$ from these bases [S2, book]. Simple regular subsampling achieves very good results, though other re-sampling and interpolation methods can be considered [S2, book]. This allows for multiresolution hierarchies similar to mipmapping [S2, book].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the total number of voxels in the output volume and the number of components, dominated by the upsampling and merging operations [S1, unofficial].
*   **Memory Complexity:** Proportional to the size of the output volume plus the temporary storage for each component brick [S1, unofficial].
*   **Strengths:** Simple, deterministic reconstruction of existing data structures; handles overlapping regions via a clear priority rule (maximum) [S1, unofficial].
*   **Limitations:** Lossy merging if overlapping bricks have conflicting values; requires precise metadata (spacing, origin, extent) for correct alignment [S1, unofficial].
*   **When to use:** When reconstructing a full-resolution volume from pre-computed, exported chunks or bricks [S1, unofficial].
*   **When not to use:** When the source data is not already discretized into bricks, or when lossy merging of overlapping values is unacceptable [S1, unofficial].

## Background
Reference Reconstruction is a method for assembling volumetric data from component parts, as described in the VolumeSegmenter project [S1, unofficial]. It is designed to handle the reconstruction of voxel grids from exported bricks, particularly in the context of large-scale volumetric datasets [S1, unofficial].

## Sources
[S1] Winson-L/VolumeSegmenter, https://github.com/Winson-L/VolumeSegmenter, fetched 2026-10-27 (unofficial)
[S2] Multiscale Tensor Approximation for Volume Data, https://www.ifi.uzh.ch/dam/jcr:00000000-5bd6-887a-ffff-ffffb737772b/TA_Volumes.pdf, fetched 2026-10-29 (book)
