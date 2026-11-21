# Chroma Subsampling

Also called: Chroma Subsampling for Volumetric Data, Chroma subsampling for volume, Color component subsampling

## Overview
Chroma subsampling for volumetric data reduces the bitrate of color information by lowering the spatial resolution of chrominance components (U, V) while retaining luminance (Y) at full resolution. It solves the problem of significant color information loss and "grey object" artifacts that occur when direct subsampling is applied to sparse volumetric data [S1]. Unlike direct subsampling, which discards data indiscriminately, this method defines one color value for every eight adjacent corners, preserving color quality at low bitrates by using a median-based selection strategy for existing voxels [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset where voxels have associated color values (typically Y, U, V components). The volume is sparse, meaning many positions are empty/transparent.
*   **Output:** A compressed representation where chrominance data is stored at 1/8th the spatial density of the luminance data, with a mechanism for reconstruction.

**Data Structures:**
The method operates on a grid of "corners" or voxel positions. For every group of eight adjacent corners (forming a $2 \times 2 \times 2$ block in the spatial domain), a single subsampled chroma voxel is defined.
*   **Subsampled Voxel:** Stores the median color value of the existing voxels in the corresponding $2 \times 2 \times 2$ block.
*   **Transparency Flag:** Implicitly handled; if no voxels exist in the block, the subsampled voxel is defined as transparent.

**Numbered Steps:**

1.  **Grouping:** Divide the volume grid into non-overlapping $2 \times 2 \times 2$ blocks of corner positions.
2.  **Existence Check:** For each block, identify which of the eight corner positions contain existing (non-transparent) voxels.
3.  **Value Selection:**
    *   If **no** existing voxels are present in any of the eight corners, define the subsampled voxel as **transparent**.
    *   If **one or more** existing voxels are present, calculate the **median** value of the chrominance components (U, V) of these existing voxels. Use this median value as the color value for the subsampled voxel.
    *   *Note:* The median is chosen over the average because it provides better sharpness and is less influenced by outlier values, ignoring high-difference values [S1].
4.  **Storage:** Store the resulting subsampled chroma values. Luminance (Y) is stored at full resolution (not subsampled).

**Decoder Side (Reconstruction):**
To reconstruct the full-resolution chroma map for rendering or further processing:
1.  **Interpolation or Repetition:** For positions where chroma data is missing (because the corresponding $2 \times 2 \times 2$ block had no existing voxels or for upsampling purposes), interpolate from neighboring voxels or repeat the preceding sample.
2.  **Simulation Choice:** In simulation contexts, the **repeating approach** (repeating the preceding sample) is employed for missing values because viewers' sense of color is barely affected by this approximation [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chroma_subsample_volume(volume):
    """
    volume: A sparse volume representation.
            Assumed to be iterable over grid positions.
            Each position has a 'has_voxel' boolean and a 'color' tuple (Y, U, V).
            Returns a subsampled chroma map and a transparency mask.
    """
    # Determine grid dimensions
    dims = volume.get_dimensions()
    subsampled_dims = tuple((d + 1) // 2 for d in dims) # Ceiling division for block alignment
    
    # Initialize output structures
    # chroma_map stores (U, V) for each 2x2x2 block
    chroma_map = {} 
    # transparency_map stores boolean: True if the block is entirely empty
    transparency_map = {}

    for x in range(0, dims[0], 2):
        for y in range(0, dims[1], 2):
            for z in range(0, dims[2], 2):
                existing_colors = []
                is_empty = True
                
                # Iterate over the 8 corners of the 2x2x2 block
                for dx in [0, 1]:
                    for dy in [0, 1]:
                        for dz in [0, 1]:
                            cx, cy, cz = x + dx, y + dy, z + dz
                            
                            # Check if voxel exists at this corner
                            if volume.has_voxel(cx, cy, cz):
                                is_empty = False
                                color = volume.get_color(cx, cy, cz)
                                # Extract chrominance components (U, V)
                                # Assuming color is (Y, U, V)
                                existing_colors.append((color[1], color[2]))
                
                if is_empty:
                    transparency_map[(x, y, z)] = True
                    chroma_map[(x, y, z)] = None # Transparent
                else:
                    transparency_map[(x, y, z)] = False
                    # Calculate median of existing chroma values
                    # Median is computed per component (U and V) independently
                    u_values = [c[0] for c in existing_colors]
                    v_values = [c[1] for c in existing_colors]
                    
                    median_u = median(u_values)
                    median_v = median(v_values)
                    
                    chroma_map[(x, y, z)] = (median_u, median_v)
                    
    return chroma_map, transparency_map

def median(values):
    """Helper to compute median of a list of numbers."""
    sorted_vals = sorted(values)
    n = len(sorted_vals)
    if n % 2 == 1:
        return sorted_vals[n // 2]
    else:
        # For even number of elements, standard median is average of two middle values
        # However, source emphasizes median for sharpness/outlier resistance.
        # If strict median of discrete samples is required, pick one of the middles.
        # Given the context of "median value of existing voxels", 
        # we assume standard statistical median.
        return (sorted_vals[n // 2 - 1] + sorted_vals[n // 2]) / 2.0

def reconstruct_chroma(chroma_map, transparency_map, dims):
    """
    Reconstruct full-resolution chroma using repetition for missing values.
    """
    full_chroma = {}
    
    for x in range(dims[0]):
        for y in range(dims[1]):
            for z in range(dims[2]):
                # Determine which block this voxel belongs to
                block_x = x // 2
                block_y = y // 2
                block_z = z // 2
                
                if transparency_map.get((block_x, block_y, block_z), False):
                    # If the block is empty, we might need to look for neighbors
                    # But for simulation, repeating preceding sample is used.
                    # Here we assume a simple fallback or interpolation.
                    # For simplicity in this self-contained example, we mark as transparent/zero
                    full_chroma[(x, y, z)] = (0, 0) 
                else:
                    uv = chroma_map.get((block_x, block_y, block_z))
                    if uv:
                        full_chroma[(x, y, z)] = uv
                    else:
                        # Fallback for missing data in non-empty blocks (edge case)
                        # In simulation, repeat preceding sample.
                        # This requires a defined order.
                        full_chroma[(x, y, z)] = (0, 0) # Placeholder for interpolation logic
    return full_chroma
```

**Parameters:**
*   **Subsampling Factor:** Fixed at 8 (one value per $2 \times 2 \times 2$ block).
*   **Median Calculation:** Applied independently to U and V components.

**Edge Cases and Pitfalls:**
*   **Empty Blocks:** If a $2 \times 2 \times 2$ block contains no voxels, the subsampled value is transparent. Failure to handle this leads to incorrect color assignment [S1].
*   **Low Bitrate Loss:** Without chroma subsampling, chroma information can be completely lost at low bitrates, resulting in grey objects. This method prevents that by preserving the median color of existing geometry [S1].
*   **Outliers:** Using the mean instead of the median can lead to poor sharpness and sensitivity to outlier color values. The median is robust against high-difference values [S1].

## Operating characteristics
*   **Compression Ratio:** The chrominance data size is reduced by a factor of 8 compared to full-resolution chroma storage.
*   **Performance:** In the Leo sequence experiment, chroma subsampling achieved 39.30 dB PSNR at 1.18 bpv (bits per voxel), whereas the non-subsampling approach required 2.90 bpv for the same quality [S1].
*   **Strengths:** Preserves color quality at very low bitrates; prevents "grey object" artifacts; median selection improves sharpness and reduces outlier impact.
*   **Limitations:** Introduces spatial aliasing in the chroma channels; reconstruction requires interpolation or repetition, which may introduce artifacts if not handled carefully (though repetition is acceptable for human perception of color) [S1].
*   **When to Use:** When bandwidth or storage is constrained, and color fidelity is more important than perfect chroma resolution. Particularly effective for sparse volumes where many voxels are empty.
*   **When Not to Use:** When high-fidelity chroma detail is required and bitrate is not a constraint.

## Background
Chroma subsampling is a well-known technique in image and video compression (e.g., 4:2:0, 4:2:2) that exploits the human visual system's lower sensitivity to color detail compared to luminance detail. Its application to volumetric data was proposed to address the specific challenges of sparse volumetric representations, where direct subsampling causes significant information loss. The method was introduced in the context of color volumetric compression for realistic view synthesis applications [S1].

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
