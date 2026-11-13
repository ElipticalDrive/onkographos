# Chroma Subsampling

Also called: Chroma Subsampling for Volumetric Data, Chroma subsampling for volume, Color component subsampling

## Overview
Chroma subsampling for volumetric data reduces the bitrate of color information by lowering the resolution of chrominance components (U, V) while maintaining luminance (Y) at full resolution [S1]. Unlike direct subsampling, which causes significant information loss in sparse volumetric representations, this method defines one color value for every eight adjacent corners of a voxel grid, preserving color quality at low bitrates [S1]. It is distinct from sibling methods like Scalar Quantization because it operates on the spatial arrangement of existing voxels to determine color validity, rather than merely reducing precision of existing values [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset where voxels have associated luminance (Y) and chrominance (U, V) values. The data is sparse, meaning not all grid positions contain valid surface voxels.
*   **Output:** A compressed representation where chrominance is stored at 1/8th the spatial density of the luminance, with a mechanism to reconstruct full-resolution colors during decoding.

**Data Structures:**
The method operates on a grid of corners. For every group of eight adjacent corners in the 3D grid, a single subsampled chroma voxel is defined.
*   **Subsampled Voxel:** Stores a single color value (U, V) or a "transparent" flag.
*   **Luminance:** Remains at full voxel resolution (one Y per original voxel).

**Algorithm Steps:**
1.  **Grouping:** Divide the 3D grid into non-overlapping groups of eight adjacent corners. Each group corresponds to a single subsampled chroma voxel location.
2.  **Existence Check:** For each group, determine if any of the eight constituent corners contain an existing (non-transparent) voxel from the original volume.
3.  **Color Assignment (Encoder):**
    *   If **no** existing voxel is present in any of the eight corners, define the subsampled voxel as **transparent** [S1].
    *   If **one or more** existing voxels are present, calculate the color value using the **median** of the chrominance values (U, V) of the existing voxels in that group [S1].
    *   *Note:* The median is chosen over the average because it provides better sharpness and is less influenced by outlier values (high differences) [S1].
4.  **Reconstruction (Decoder):**
    *   For each original voxel location, retrieve the chrominance value from the corresponding subsampled group.
    *   If the subsampled voxel is **transparent**, the color is missing.
    *   To fill missing values, employ one of two strategies:
        *   **Interpolation:** Interpolate from neighboring valid voxels [S1].
        *   **Repetition:** Repeat the preceding sample (nearest-neighbor in traversal order) [S1]. The source notes that viewers' sense of color is barely affected by this repetition, making it a valid simulation choice [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chroma_subsample_encode(volume):
    """
    Encodes chrominance (U, V) using 8-corner subsampling.
    volume: List of existing voxels, each with position (x,y,z) and color (u, v).
    Returns: A dictionary mapping subsampled grid indices to (u, v) or None.
    """
    subsampled_data = {}
    
    # Determine grid bounds to iterate over all 8-corner groups
    # Assuming integer coordinates for corners
    min_x = min(v.pos[0] for v in volume)
    max_x = max(v.pos[0] for v in volume)
    min_y = min(v.pos[1] for v in volume)
    max_y = max(v.pos[1] for v in volume)
    min_z = min(v.pos[2] for v in volume)
    max_z = max(v.pos[2] for v in volume)

    for x in range(min_x, max_x + 1, 8):
        for y in range(min_y, min_y + 1, 8): # Simplified iteration for logic clarity
            for z in range(min_z, min_z + 1, 8):
                # Identify the 8 corners of the current group
                corners = [
                    (x, y, z), (x+1, y, z), (x+2, y, z), (x+3, y, z),
                    (x+4, y, z), (x+5, y, z), (x+6, y, z), (x+7, y, z)
                ]
                # Note: The source implies a 3D grouping of 8 corners. 
                # In 3D, 8 corners usually form a cube (2x2x2). 
                # The source text says "eight adjacent corners" and shows Fig 8.
                # We assume a 2x2x2 block for standard 3D subsampling logic, 
                # or a specific 1D/2D arrangement if implied by "corners".
                # Given "volume", 2x2x2 is the standard interpretation of 8 neighbors.
                
                existing_colors = []
                for corner_pos in corners:
                    # Find if a voxel exists at this corner
                    voxel = volume.get(corner_pos)
                    if voxel is not None:
                        existing_colors.append(voxel.color) # (u, v)

                if not existing_colors:
                    subsampled_data[(x//8, y//8, z//8)] = None # Transparent
                else:
                    # Calculate median of existing colors
                    # Median is calculated component-wise for U and V
                    u_values = [c[0] for c in existing_colors]
                    v_values = [c[1] for c in existing_colors]
                    
                    median_u = median(u_values)
                    median_v = median(v_values)
                    
                    subsampled_data[(x//8, y//8, z//8)] = (median_u, median_v)
                    
    return subsampled_data

def chroma_subsample_decode(subsampled_data, volume_geometry):
    """
    Decodes chrominance by filling missing values.
    subsampled_data: Dict mapping subsampled indices to (u, v) or None.
    volume_geometry: List of all voxel positions that need color.
    Returns: Full volume with Y, U, V.
    """
    full_volume = {}
    
    # Map each voxel position to its subsampled group index
    for pos in volume_geometry:
        group_idx = (pos[0]//8, pos[1]//8, pos[2]//8)
        color = subsampled_data.get(group_idx)
        
        if color is not None:
            full_volume[pos] = color
        else:
            # Missing value: Use repetition of preceding sample
            # In a simulation, 'preceding' implies a traversal order.
            # Here we assume a simple nearest-neighbor search in a pre-defined order
            # or interpolation. The source prefers repetition for simulation.
            full_volume[pos] = get_preceding_sample(pos, full_volume) 
            
    return full_volume

def median(values):
    sorted_vals = sorted(values)
    n = len(sorted_vals)
    if n % 2 == 1:
        return sorted_vals[n // 2]
    else:
        return (sorted_vals[n // 2 - 1] + sorted_vals[n // 2]) / 2

def get_preceding_sample(pos, filled_volume):
    """
    Placeholder for the 'repeating approach'.
    In practice, this would look back in the traversal order or 
    use nearest valid neighbor.
    """
    # [G] Implementation detail: In a real raster scan, this would be the last 
    # assigned non-None value. For this generic pseudocode, we return a default 
    # or nearest neighbor if available.
    return (0, 0) # Placeholder for missing logic
```

**Parameters:**
*   **Subsampling Factor:** 8 (one value per 8 corners).
*   **Aggregation Function:** Median.
*   **Reconstruction Method:** Repetition (preferred for simulation) or Interpolation.

**Edge Cases and Pitfalls:**
*   **Complete Loss of Color:** Without subsampling, chroma information can be completely lost at low bitrates, resulting in grey objects [S1].
*   **Outliers:** Using the mean instead of the median can lead to poor sharpness and sensitivity to outlier color values [S1].
*   **Transparency:** Correctly identifying groups with *no* existing voxels is critical to avoid assigning arbitrary colors to empty space [S1].

## Operating characteristics
*   **Bitrate Reduction:** Chroma subsampling significantly reduces bitrate for chrominance components. In the Leo sequence experiment, it achieved 39.30 dB PSNR at 1.18 bpv (bits per voxel), whereas non-subsampling required 2.90 bpv for the same quality [S1].
*   **Quality:** Preserves color quality at low bitrates where direct subsampling would fail [S1].
*   **Complexity:** The encoding involves grouping and median calculation, which is computationally light. Decoding involves simple lookup and repetition/interpolation.
*   **Limitations:** The method relies on the spatial distribution of voxels. If voxels are extremely sparse or irregularly distributed, the median might not represent the local color well. The "repetition" decoding strategy may introduce color artifacts in regions with rapid color changes, though subjective tests suggest this is minimal [S1].

## Background
Chroma subsampling is a technique adapted from image compression (e.g., JPEG, MPEG) to volumetric data. In images, chroma is subsampled because the human visual system is less sensitive to color detail than luminance detail. For volumes, where intensity and color are defined only for existing voxels, direct subsampling causes significant information loss. This method was proposed to address this specific challenge in volumetric view synthesis applications [S1].

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
