# Chroma Subsampling
Also called: Chroma Subsampling for Volumetric Data, Chroma subsampling for volume, Color component subsampling

## Overview
Chroma subsampling for volumetric data is a compression technique that reduces the resolution of chrominance components (U, V) while maintaining luminance (Y) at full resolution, thereby improving color quality at low bitrates [S1]. Unlike direct subsampling which causes significant information loss in sparse volumes, this method defines one color value for every eight adjacent corners, preserving color fidelity better than naive averaging or dropping [S1]. It is specifically designed to prevent the complete loss of chroma information (resulting in grey objects) that occurs when coding sparse volumetric data without subsampling [S1].

## How it works
**Inputs and Outputs**
- **Input:** A set of existing voxels in a volumetric grid, each with a color value (typically in YUV or similar space where chroma is separated).
- **Output:** A compressed representation where chroma values are stored at a lower resolution (1 value per 2x2x2 block of corners), while luminance remains at full resolution.

**Data Structures**
The method operates on groups of eight adjacent corners in the volumetric grid. The output structure stores a single chroma value for each such group. The luminance component is stored separately at the original voxel resolution.

**Steps**
1. **Grouping:** Divide the volumetric grid into non-overlapping groups of eight adjacent corners (a 2x2x2 block).
2. **Existence Check:** For each group, determine if any of the eight corners contain an existing voxel (i.e., is not empty/transparent).
3. **Transparent Case:** If no existing voxel is present in any of the eight corners, define the subsampled voxel as transparent [S1].
4. **Non-Transparent Case:** If one or more voxels are present in the eight adjacent corners, calculate the median value of the chroma components (U, V) of the existing voxels [S1].
5. **Storage:** Store the calculated median chroma value for the group. The luminance (Y) is stored for each individual voxel as usual.
6. **Decoding (Interpolation/Repetition):** On the decoder side, missing chroma values for individual voxels within a group are recovered by either:
   - Interpolating from neighboring voxels' chroma values, or
   - Repeating the preceding sample (the chroma value of the group) [S1].
   The repeating approach is employed in simulations because viewers' sense of color is barely affected by this approximation [S1].

**Formulas**
Let $G$ be a group of eight corners $C_1, ..., C_8$. Let $V_i$ be the chroma value (U, V) of corner $i$ if it exists, and $\emptyset$ if it does not.
Let $E = \{ V_i \mid C_i \text{ exists} \}$ be the set of existing chroma values in the group.

If $E = \emptyset$:
$$ \text{Chroma}_{subsampled} = \text{Transparent} $$

If $E \neq \emptyset$:
$$ \text{Chroma}_{subsampled} = \text{Median}(E) $$

The median is chosen over the average because it provides better sharpness and is less influenced by outlier values [S1].

**Parameters**
- **Subsampling Factor:** 8:1 for chroma (1 value per 8 corners).
- **Luminance Resolution:** Full (1 value per voxel).
- **Decoder Strategy:** Repeating (preferred for simulation) or Interpolation [S1].

**Edge Cases and Pitfalls**
- **Empty Groups:** If a 2x2x2 block has no voxels, the result is transparent. Failure to handle this may lead to incorrect color assignment [S1].
- **Outliers:** Using the mean instead of the median can lead to blurred colors due to outlier influence. The median is robust against this [S1].
- **Low Bitrate Loss:** Without chroma subsampling, chroma information can be completely lost at low bitrates, resulting in grey objects. This method prevents that [S1].
- **Precision:** The median operation is discrete; ensure integer or fixed-point arithmetic is handled correctly to avoid rounding errors before median selection [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chroma_subsample_volume(voxels):
    """
    voxels: list of (x, y, z, y_val, u_val, v_val) for existing voxels.
    Returns: subsampled_chroma_grid, where each entry is (u, v) or None (transparent).
    """
    # Determine grid bounds
    max_x = max(v[0] for v in voxels)
    max_y = max(v[1] for v in voxels)
    max_z = max(v[2] for v in voxels)
    
    # Initialize subsampled grid (2x2x2 blocks)
    # Dimensions are ceil(max_dim / 2)
    dim_x = (max_x + 1) // 2
    dim_y = (max_y + 1) // 2
    dim_z = (max_z + 1) // 2
    
    subsampled_grid = [[[[None for _ in range(dim_z)] for _ in range(dim_y)] for _ in range(dim_x)]]
    
    # Group voxels by their 2x2x2 block index
    block_voxels = {}
    
    for x, y, z, y_val, u_val, v_val in voxels:
        # Calculate block index
        bx = x // 2
        by = y // 2
        bz = z // 2
        
        key = (bx, by, bz)
        if key not in block_voxels:
            block_voxels[key] = []
        block_voxels[key].append((u_val, v_val))
        
    # Compute median for each block
    for bx in range(dim_x):
        for by in range(dim_y):
            for bz in range(dim_z):
                key = (bx, by, bz)
                if key in block_voxels:
                    chroma_values = block_voxels[key]
                    # Extract U and V components
                    u_vals = [c[0] for c in chroma_values]
                    v_vals = [c[1] for c in chroma_values]
                    
                    # Calculate median
                    median_u = sorted(u_vals)[len(u_vals) // 2]
                    median_v = sorted(v_vals)[len(v_vals) // 2]
                    
                    subsampled_grid[bx][by][bz] = (median_u, median_v)
                else:
                    subsampled_grid[bx][by][bz] = None # Transparent
                    
    return subsampled_grid

def decode_chroma(subsampled_grid, x, y, z):
    """
    Recovers chroma for a specific voxel (x,y,z) from the subsampled grid.
    Uses repeating approach: returns the chroma value of the block containing (x,y,z).
    """
    bx = x // 2
    by = y // 2
    bz = z // 2
    
    # Check bounds
    if bx >= len(subsampled_grid) or by >= len(subsampled_grid[0]) or bz >= len(subsampled_grid[0][0]):
        return None # Out of bounds, treat as transparent
        
    return subsampled_grid[bx][by][bz]
```

## Operating characteristics
- **Compression Ratio:** Achieves 39.30 dB PSNR at 1.18 bpv (bits per voxel) for chroma components, whereas non-subsampling requires 2.90 bpv for the same quality [S1].
- **Quality:** Subjective results show that without subsampling, objects appear grey at low bitrates (e.g., 0.2 bpv), while subsampling preserves color [S1].
- **Complexity:** Encoding requires grouping and median calculation, which is $O(N)$ where $N$ is the number of voxels. Decoding is $O(1)$ per voxel for the repeating approach.
- **Strengths:** Preserves color quality at very low bitrates; robust to outliers; simple to implement.
- **Limitations:** Reduces chroma resolution, which may cause color bleeding at boundaries if interpolation is not used; only applies to color data, not luminance or geometry.

## Background
Chroma subsampling is a well-known technique in image and video compression (e.g., 4:2:0, 4:2:2) that exploits the human visual system's lower sensitivity to color detail compared to luminance detail [S1]. Its application to volumetric data was proposed to address the specific challenge of sparse volumetric representations where direct subsampling leads to significant information loss [S1]. The method was tested on the multiview Leo sequence captured at the University of Bristol [S1].

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
