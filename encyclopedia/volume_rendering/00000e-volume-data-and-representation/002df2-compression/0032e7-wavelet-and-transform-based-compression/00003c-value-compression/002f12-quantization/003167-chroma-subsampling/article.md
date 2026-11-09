# Chroma Subsampling
Also called: Chroma Subsampling for Volumetric Data, Chroma subsampling for volume, Color component subsampling

## Overview
Chroma subsampling reduces the bitrate of volumetric data by lowering the resolution of chrominance components (U, V) while retaining luminance (Y) at full resolution, addressing the problem of color information loss at low bitrates. It differs from direct subsampling by defining one color value for every eight adjacent corners using a median-based aggregation, which preserves sharpness and reduces outlier influence compared to averaging [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of existing voxels, each with a color value (typically in YUV or similar space) and a validity flag.
*   **Output:** A subsampled volume where chrominance data is stored at a coarser resolution (1 value per 2x2x2 corner group), while luminance remains per-voxel.

**Data Structures:**
The method operates on groups of eight adjacent corners (a 2x2x2 block). For each block, a single chrominance value is computed. The structure does not explicitly store the 8 individual chroma values but rather the aggregated result.

**Steps:**
1.  **Grouping:** Partition the volume corners into non-overlapping 2x2x2 blocks.
2.  **Validity Check:** For each block, determine if any of the eight corners contain an existing (valid) voxel.
3.  **Aggregation:**
    *   If **no** existing voxel is present in any of the eight corners, define the subsampled voxel as **transparent** (or null).
    *   If **one or more** existing voxels are present, compute the **median** value of the chrominance components (U, V) of the existing voxels. The luminance (Y) is not subsampled in this specific step; it is handled separately or kept at full resolution.
    *   *Note:* The median is chosen over the average because it provides better sharpness and is less influenced by outlier values [S1].
4.  **Decoding/Reconstruction:**
    *   When rendering or reconstructing, missing chroma values (where the subsampled voxel is transparent or undefined) must be handled.
    *   Two methods are cited: interpolation from neighboring voxels or repeating the preceding sample.
    *   The repeating approach is employed in simulations because viewers' sense of color is barely affected by this approximation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chroma_subsample_volume(voxels):
    """
    voxels: A list or grid of voxel objects.
    Each voxel has:
      - is_valid: boolean
      - color: tuple (Y, U, V)
    Returns:
      subsampled_chroma: A grid or list of chroma values (U, V) for 2x2x2 blocks.
    """
    subsampled_chroma = []
    
    # Iterate through all 2x2x2 blocks
    # Note: Implementation details for grid iteration depend on the specific volume layout.
    # Assuming a simple linear iteration over blocks for clarity.
    for block in get_2x2x2_blocks(voxels):
        existing_voxels = [v for v in block if v.is_valid]
        
        if not existing_voxels:
            # No existing voxel: define as transparent
            subsampled_chroma.append(None)
        else:
            # One or more voxels present: use median of chrominance
            # Extract U and V components
            u_values = [v.color[1] for v in existing_voxels]
            v_values = [v.color[2] for v in existing_voxels]
            
            # Compute median
            median_u = median(u_values)
            median_v = median(v_values)
            
            subsampled_chroma.append((median_u, median_v))
            
    return subsampled_chroma

def reconstruct_chroma(subsampled_chroma, neighbors=None):
    """
    Reconstruct full-resolution chroma for rendering.
    """
    reconstructed = []
    for chroma_val in subsampled_chroma:
        if chroma_val is None:
            # Missing value: repeat preceding sample or interpolate
            # Using repeating approach as per source [S1]
            if reconstructed:
                prev = reconstructed[-1]
                reconstructed.append(prev)
            else:
                reconstructed.append((0, 0)) # Default if no previous
        else:
            reconstructed.append(chroma_val)
    return reconstructed

def median(values):
    """Helper to compute median of a list."""
    sorted_vals = sorted(values)
    n = len(sorted_vals)
    if n % 2 == 1:
        return sorted_vals[n // 2]
    else:
        return (sorted_vals[n // 2 - 1] + sorted_vals[n // 2]) / 2
```

**Parameters:**
*   **Subsampling Factor:** 2x2x2 (8 corners per value) [S1].
*   **Aggregation Function:** Median [S1].

**Edge Cases and Pitfalls:**
*   **Empty Blocks:** If a 2x2x2 block contains no valid voxels, the result is transparent. This must be handled during decoding to avoid artifacts [S1].
*   **Outliers:** Using the median prevents single outlier voxels from skewing the color, which is a key advantage over averaging [S1].
*   **Color Loss:** Without subsampling, chroma information can be completely lost at low bitrates, resulting in grey objects. Subsampling mitigates this by preserving color data more efficiently [S1].

## Operating characteristics
*   **Bitrate Reduction:** Chroma subsampling significantly reduces the bitrate required for chrominance components. In the Leo sequence experiment, it achieved 39.30 dB PSNR at 1.18 bits per voxel (bpv), whereas non-subsampling required 2.90 bpv for the same quality [S1].
*   **Quality:** At low bitrates (e.g., 0.2 bpv), non-subsampled chroma results in grey objects, while subsampled chroma retains color information [S1].
*   **Complexity:** The method adds computational overhead for median calculation and block grouping, but this is negligible compared to the savings in storage and transmission.
*   **Limitations:** The method assumes that chrominance is less critical than luminance for perceived quality, which is generally true for human vision. However, it may introduce color artifacts if the median is not representative of the local color distribution (e.g., in high-frequency color regions).

## Background
Chroma subsampling is a well-known technique in image and video compression (e.g., 4:2:0, 4:2:2) that exploits the human visual system's lower sensitivity to color detail compared to luminance detail. Its application to volumetric data, as described in [S1], adapts this principle to 3D space by subsampling chrominance across 2x2x2 corner groups. This approach was proposed to improve the quality of reconstructed color components in multiview volume synthesis applications.

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
