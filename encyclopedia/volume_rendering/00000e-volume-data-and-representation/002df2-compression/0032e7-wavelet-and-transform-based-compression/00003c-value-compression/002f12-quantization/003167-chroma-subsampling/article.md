# Chroma Subsampling

Also called: Chroma Subsampling for Volumetric Data, Chroma subsampling for volume, Color component subsampling

## Overview
Chroma subsampling for volumetric data reduces the bitrate of color information by lowering the spatial resolution of chrominance components (U, V) while retaining luminance (Y) at full resolution [S1]. It solves the problem of significant color information loss and "grey object" artifacts that occur when direct subsampling is applied to sparse volumetric data [S1]. Unlike direct subsampling, which discards data indiscriminately, this method defines one color value for every eight adjacent corners, preserving color quality at low bitrates by using a median-based selection strategy for existing voxels [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset where voxels have associated color values (typically Y, U, V components) and a grid of corner positions [S1].
*   **Output:** A compressed representation where chrominance data is stored at reduced spatial density, with a mechanism for reconstruction [S1].

**Data Structures:**
The method operates on a grid of corner or voxel positions [S1]. For every group of eight adjacent corners (forming a $2 \times 2 \times 2$ block in the spatial domain), a single subsampled chroma voxel is defined [S1].
*   **Subsampled Voxel:** Stores the median color value of the existing voxels in the corresponding $2 \times 2 \times 2$ block [S1].
*   **Transparency Flag:** Implicitly handled; if no voxels exist in the block, the subsampled voxel is defined as transparent [S1].

**Numbered Steps:**
1.  **Grouping:** Divide the volume grid into non-overlapping $2 \times 2 \times 2$ blocks of corner positions [S1].
2.  **Existence Check:** For each block, identify which of the eight corner positions contain existing (non-transparent) voxels [S1].
3.  **Value Selection:**
    *   If **no** existing voxels are present in any of the eight corners, define the subsampled voxel as **transparent** [S1].
    *   If **one or more** existing voxels are present, calculate the **median** value of the chrominance components (U, V) of these existing voxels [S1].
    *   Use this median value as the color value for the subsampled voxel [S1].
    *   *Note:* The median is chosen over the average because it provides better sharpness and is less influenced by outlier values, ignoring high-difference values [S1].
4.  **Storage:** Store the resulting subsampled chroma values [S1]. Luminance (Y) is stored at full resolution (not subsampled) [S1].

**Decoder Side (Reconstruction):**
To reconstruct the full-resolution chroma map for rendering or further processing:
1.  **Interpolation or Repetition:** For positions where chroma data is missing (because the corresponding $2 \times 2 \times 2$ block had no existing voxels or for upsampling purposes), interpolate from neighboring voxels or repeat the preceding sample [S1].
2.  **Simulation Choice:** In simulation contexts, the **repeating approach** (repeating the preceding sample) is employed for missing values because viewers' sense of color is barely affected by this approximation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chroma_subsample_volume(volume_data, dims):
    """
    volume_data: dict mapping (x, y, z) -> (Y, U, V) for existing voxels.
    dims: tuple (Dx, Dy, Dz) of grid dimensions.
    Returns: chroma_map (dict mapping block_origin -> (U, V)), transparency_map (dict mapping block_origin -> bool).
    """
    chroma_map = {}
    transparency_map = {}

    for x in range(0, dims[0], 2):
        for y in range(0, dims[1], 2):
            for z in range(0, dims[2], 2):
                existing_uv = []
                is_empty = True

                for dx in (0, 1):
                    for dy in (0, 1):
                        for dz in (0, 1):
                            cx, cy, cz = x + dx, y + dy, z + dz
                            # Boundary check for non-even grid dimensions
                            if cx < dims[0] and cy < dims[1] and cz < dims[2]:
                                if (cx, cy, cz) in volume_data:
                                    is_empty = False
                                    _, u, v = volume_data[(cx, cy, cz)]
                                    existing_uv.append((u, v))

                if is_empty:
                    transparency_map[(x, y, z)] = True
                    chroma_map[(x, y, z)] = None
                else:
                    transparency_map[(x, y, z)] = False
                    u_vals = [uv[0] for uv in existing_uv]
                    v_vals = [uv[1] for uv in existing_uv]
                    chroma_map[(x, y, z)] = (median(u_vals), median(v_vals))

    return chroma_map, transparency_map

def median(values):
    s = sorted(values)
    n = len(s)
    if n % 2 == 1:
        return s[n // 2]
    return (s[n // 2 - 1] + s[n // 2]) / 2.0

def reconstruct_chroma(chroma_map, transparency_map, dims):
    """
    Reconstructs full-resolution chroma using repetition of preceding samples for missing values.
    Exact interpolation formulas are not specified in the sources, but repetition is implemented as per the simulation choice [S1].
    """
    full_chroma = {}
    last_uv = (0.0, 0.0)
    for x in range(dims[0]):
        for y in range(dims[1]):
            for z in range(dims[2]):
                block_x, block_y, block_z = x // 2, y // 2, z // 2
                if transparency_map.get((block_x, block_y, block_z), False):
                    full_chroma[(x, y, z)] = last_uv
                else:
                    uv = chroma_map.get((block_x, block_y, block_z))
                    if uv is not None:
                        last_uv = uv
                        full_chroma[(x, y, z)] = uv
                    else:
                        full_chroma[(x, y, z)] = last_uv
    return full_chroma
```

**Parameters:**
*   **Subsampling Factor:** Fixed at 8 (one value per $2 \times 2 \times 2$ block) [S1].
*   **Median Calculation:** Applied independently to U and V components [S1].

**Edge Cases and Pitfalls:**
*   **Empty Blocks:** If a $2 \times 2 \times 2$ block contains no voxels, the subsampled value is transparent [S1].
*   **Low Bitrate Loss:** Without chroma subsampling, chroma information can be completely lost at low bitrates, resulting in grey objects [S1].
*   **Outliers:** Using the mean instead of the median can lead to poor sharpness and sensitivity to outlier color values [S1].

## Operating characteristics
*   **Compression Ratio:** The chrominance data size is reduced by a factor of 8 compared to full-resolution chroma storage [S1].
*   **Performance:** In the Leo sequence experiment, chroma subsampling achieved 39.30 dB PSNR at 1.18 bpv (bits per voxel), whereas the non-subsampling approach required 2.90 bpv for the same quality [S1].
*   **Strengths:** Preserves color quality at very low bitrates; prevents "grey object" artifacts; median selection improves sharpness and reduces outlier impact [S1].
*   **When to Use:** When bandwidth or storage is constrained, and color fidelity is more important than perfect chroma resolution [S1].
*   **When Not to Use:** When high-fidelity chroma detail is required and bitrate is not a constraint [S1].

## Background
Chroma subsampling is a well-known technique in image and video compression that exploits the human visual system's lower sensitivity to color detail compared to luminance detail [G]. Its application to volumetric data was proposed to address the specific challenges of sparse volumetric representations, where direct subsampling causes significant information loss [S1]. The method was introduced in the context of color volumetric compression for realistic view synthesis applications [S1].

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
