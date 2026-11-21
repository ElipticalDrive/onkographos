# Density Bitfield Packing

Also called: packbits

## Overview
Density Bitfield Packing compresses a floating-point density grid into a binary bitfield to accelerate ray marching by explicitly skipping empty space [S1, S2, unofficial]. It differs from general memory layout optimizations by converting continuous density values into discrete occupancy flags, enabling traversal algorithms to jump over empty regions rather than sampling them [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A density grid `grid` of shape $[C, H^3]$ (or $[C, H, H, H]$) containing floating-point values, where $H$ is the spatial dimension and $H \% 2 == 0$ [S1, S2, unofficial]. $C$ represents the number of channels or batches.
*   **Input:** A threshold scalar `thresh` used for binarization [S1, S2, unofficial].
*   **Output:** A bitfield `bitfield` of shape $[C, H^3 / 8]$ with data type `uint8` [S1, S2, unofficial].

**Data Structures:**
The output `bitfield` stores 8 density samples per byte [S1, S2, unofficial]. For a given channel $c$ and linear grid index $idx$, the corresponding bit in the bitfield is set if the density value exceeds the threshold [S1, S2, unofficial]. The linear index in the bitfield is derived from the linear index in the grid divided by 8 [S1, S2, unofficial].

**Steps:**
1.  Ensure the input grid is a contiguous tensor [S1, unofficial].
2.  Calculate the output buffer size $N = C \cdot H^3 / 8$ [S1, unofficial].
3.  Invoke the backend packing kernel `_backend.packbits(grid, N, thresh, bitfield)` [S1, unofficial].
4.  The kernel iterates through each element of the grid [S1, S2, unofficial]. For each density value $v$ at linear index $idx$:
    *   Determine the bit position: $b = idx \pmod 8$ [S1, S2, unofficial].
    *   Determine the byte position: $byte\_idx = \lfloor idx / 8 \rfloor$ [S1, S2, unofficial].
    *   If $v \ge \text{thresh}$, set the $b$-th bit of `bitfield[byte_idx]` to 1 [S1, S2, unofficial].
5.  Return the `bitfield` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def packbits(grid: list, thresh: float) -> list:
    """
    Packs a floating-point density grid into a bitfield.
    
    Args:
        grid: Flat list of floats of shape [C, H^3]. H must be even.
        thresh: Float threshold for binarization.
        
    Returns:
        List of uint8 values of shape [C, H^3 / 8].
    """
    total_voxels = len(grid)
    N = total_voxels // 8
    bitfield = [0] * N
    
    for idx, v in enumerate(grid):
        if v >= thresh:
            byte_idx = idx // 8
            bit_pos = idx % 8
            bitfield[byte_idx] |= (1 << bit_pos)
            
    return bitfield
```

**Parameters:**
*   `thresh`: The density threshold. Values $\ge$ `thresh` are considered occupied (bit 1), values $<$ `thresh` are empty (bit 0) [S1, S2, unofficial]. Typical values: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Grid Dimensions:** The height $H$ must be even ($H \% 2 == 0$) as required by the implementation [S1, S2, unofficial].
*   **Precision:** Floating-point comparisons against `thresh` determine occupancy [S1, S2, unofficial].
*   **Memory Layout:** The grid must be contiguous for the backend kernel execution [S1, unofficial].

## Subtopics

### Bitwise Density Computation
Calculates the number of filled voxels between a scene point and a light position to simulate translucency, avoiding expensive bit counting with textures by using an arithmetic dyadic approach [S3, unofficial]. This method is used in solid voxelization for translucency effects [S3, unofficial]. It provides better solutions than taking a difference between front and back depth maps, which can contain artifacts [S3, unofficial]. On a Geforce 6, this approach exceeds 200 Hz framerate for models with 60,000 triangles [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(C \cdot H^3)$ for the packing step, as every voxel must be visited once to determine its bit state [S1, S2, unofficial].
*   **Memory Complexity:** The output size is reduced by a factor of 8 compared to the input grid (1 bit per voxel vs. 32 bits per float) [S1, S2, unofficial].
*   **Performance:** The packing itself is a fast preprocessing step. The benefit is realized during ray marching, where empty space skipping reduces the number of density evaluations [S1, S2, unofficial].
*   **Strengths:** Significant reduction in memory bandwidth during traversal; simple implementation [S1, S2, unofficial].
*   **Limitations:** Loss of density information (binary only); requires a threshold choice which may affect visual quality [S1, S2, unofficial].

## Background
Density Bitfield Packing is a technique used in volume rendering to optimize ray traversal [S1, S2, unofficial]. It is also applied in solid voxelization for translucency and CSG operations [S3, unofficial]. The specific implementation details described here are derived from unofficial repositories related to ER-NeRF and IDDR-NGP [S1, S2, unofficial].

## Sources
[S1] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
[S2] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
[S3] Single-pass GPU Solid Voxelization and Applications, https://inria.hal.science/inria-00345291, fetched 2026-10-26 (unofficial)
