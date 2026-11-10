# Chroma Subsampling

Also called: Chroma Subsampling for Volumetric Data, Chroma subsampling for volume, Color component subsampling

## Overview
Chroma subsampling reduces the bitrate of volumetric data by lowering the resolution of chrominance components (U, V) while maintaining the luminance component (Y) at full resolution [S1]. Unlike direct subsampling which causes significant information loss in sparse volumes, this method defines one color value for every eight adjacent corners, preserving color quality at low bitrates [S1]. It is distinct from sibling methods like [Attribute Quantization](1912-attribute-quantization.md) and [Scalar Quantization](5726-scalar-quantization.md) because it specifically targets the perceptual redundancy in color channels rather than general value precision, preventing the "grey object" artifact common in non-subsampled low-bitrate encoding [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of existing voxels, each containing a luminance value $Y$ and chrominance values $U, V$.
*   **Output:** A compressed representation where chrominance is stored at 1/8th the spatial density of the original voxels, with luminance remaining at full density.

**Data Structures:**
The method operates on groups of eight adjacent corners in the voxel grid. For each group, a single subsampled chrominance value is computed. The structure does not store explicit coordinates for the subsampled values; they are implicitly defined by the grid alignment of the eight corners.

**Steps:**
1.  **Grouping:** Divide the voxel grid into non-overlapping groups of eight adjacent corners.
2.  **Existence Check:** Determine if any voxel exists in the current group of eight corners.
    *   If no voxels exist in any of the eight corners, define the subsampled voxel as transparent (or null).
    *   If one or more voxels exist, proceed to step 3.
3.  **Median Calculation:** Compute the median value of the existing voxels' chrominance components ($U$ and $V$) within the group. The median is chosen over the average because it provides better sharpness and is less influenced by outlier values [S1].
    *   Let $C_i$ be the chrominance value of the $i$-th existing voxel in the group.
    *   The subsampled chrominance $C_{sub}$ is:
        $$ C_{sub} = \text{median}(\{C_i\}) $$
4.  **Storage:** Store $C_{sub}$ for the group. The luminance $Y$ is stored for each individual voxel as per standard quantization [Parent](5935-quantization.md).

**Decoder Side Reconstruction:**
1.  **Interpolation/Repetition:** For missing chrominance values in the reconstructed volume, interpolate from neighboring voxels or repeat the preceding sample [S1].
2.  **Simulation Choice:** In simulation environments, the repeating approach is employed because viewers' sense of color is barely affected by this approximation [S1].

**Pitfalls and Edge Cases:**
*   **Low Bitrate Loss:** Without chroma subsampling, chroma information can be completely lost at low bitrates, resulting in grey objects [S1].
*   **Outliers:** The median method ignores high different values (outliers), which is beneficial for sharpness but may smooth out fine color details if outliers are significant [S1].
*   **Transparency:** Correctly handling groups with no existing voxels is critical to avoid artifacts; these must be explicitly marked as transparent [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chroma_subsampling_encode(voxels):
    """
    Encodes chrominance for a volume using 8-corner grouping.
    voxels: list of voxel objects, each with attributes:
        - pos: (x, y, z) integer coordinates
        - exists: boolean
        - Y: float (luminance)
        - U: float (chrominance)
        - V: float (chrominance)
    Returns:
        subsampled_chroma: dict mapping group_key -> (U_sub, V_sub)
        full_luminance: dict mapping voxel_pos -> Y
    """
    subsampled_chroma = {}
    full_luminance = {}

    # Group voxels by their 8-corner group key
    # Assuming a regular grid, group key can be derived from floor(pos / 8)
    # or by iterating through all possible 8-corner blocks.
    # For simplicity, we assume voxels are provided in a way that allows
    # grouping by 8 adjacent corners.
    
    # Note: The source describes "eight adjacent corners". 
    # In a 3D grid, this typically implies a 2x2x2 block.
    
    groups = {} # key: (gx, gy, gz), value: list of voxels in this block
    
    for v in voxels:
        if not v.exists:
            continue
            
        # Determine group index (assuming 2x2x2 blocks)
        gx = v.pos[0] // 2
        gy = v.pos[1] // 2
        gz = v.pos[2] // 2
        key = (gx, gy, gz)
        
        if key not in groups:
            groups[key] = []
        groups[key].append(v)
        
        # Store full luminance for all existing voxels
        full_luminance[v.pos] = v.Y

    for key, group_voxels in groups.items():
        if len(group_voxels) == 0:
            # No existing voxels in this 2x2x2 block
            # Define as transparent
            subsampled_chroma[key] = None
        else:
            # Extract chrominance values
            u_values = [v.U for v in group_voxels]
            v_values = [v.V for v in group_voxels]
            
            # Compute median
            u_sub = median(u_values)
            v_sub = median(v_values)
            
            subsampled_chroma[key] = (u_sub, v_sub)

    return subsampled_chroma, full_luminance

def median(values):
    """Helper to compute median of a list of numbers."""
    sorted_vals = sorted(values)
    n = len(sorted_vals)
    if n % 2 == 1:
        return sorted_vals[n // 2]
    else:
        return (sorted_vals[n // 2 - 1] + sorted_vals[n // 2]) / 2.0

def chroma_subsampling_decode(subsampled_chroma, full_luminance, grid_shape):
    """
    Decodes chrominance by repeating or interpolating.
    Uses repeating approach as per source [S1].
    """
    reconstructed_voxels = []
    
    # Iterate through all positions in the grid
    for x in range(grid_shape[0]):
        for y in range(grid_shape[1]):
            for z in range(grid_shape[2]):
                pos = (x, y, z)
                
                # Get luminance
                Y = full_luminance.get(pos, None)
                if Y is None:
                    continue # Skip non-existing voxels
                
                # Determine group key
                gx = x // 2
                gy = y // 2
                gz = z // 2
                key = (gx, gy, gz)
                
                # Get subsampled chroma
                chroma = subsampled_chroma.get(key, None)
                
                if chroma is None:
                    # Transparent or missing
                    U, V = 0, 0 # Or handle as transparent
                else:
                    U, V = chroma
                
                reconstructed_voxels.append({
                    'pos': pos,
                    'Y': Y,
                    'U': U,
                    'V': V
                })
                
    return reconstructed_voxels
```

## Operating characteristics
*   **Bitrate Reduction:** Chroma subsampling significantly reduces bitrate for chrominance components. In the Leo sequence experiment, it achieved 39.30 dB PSNR at 1.18 bpv (bits per voxel), whereas non-subsampling required 2.90 bpv for the same quality [S1].
*   **Quality:** Prevents the complete loss of color information at low bitrates, avoiding the "grey object" artifact [S1].
*   **Complexity:** The encoding involves grouping and median calculation, which is computationally light. The median operation is $O(N \log N)$ for $N$ voxels in a group (where $N \le 8$), making it effectively constant time per group.
*   **Limitations:** The method relies on the assumption that color perception is less sensitive to high-frequency chrominance changes. It may not be suitable for applications requiring precise color fidelity at the voxel level.

## Background
Chroma subsampling is a technique adapted from image and video compression (e.g., YUV 4:2:0) to volumetric data [S1]. It was proposed in the context of realistic view synthesis applications to handle the high data rates of volumetric textures [S1]. The method was tested on the multiview Leo sequence captured at the University of Bristol [S1].

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
